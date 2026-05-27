---
title: "Alpha3D v0.1.2"
date: 2026-05-27
draft: false
description: "修复 Window 与 Camera 争用 glfwSetWindowUserPointer 导致的回调冲突；屏蔽 Windows 输入法干扰"
tags: ["图形引擎", "渲染", "摄像机", "GLFW"]
categories: ["Alpha3D 更新日志"]
cover: https://zhjh-oss.oss-cn-beijing.aliyuncs.com/Alpha3D_v0_1_2.png
---

## Phong 光照模型实现

{{< BILIBILI BV1AKGy6cEwe 2 >}}

每个顶点从 5 个 float（位置 + UV）改为 6 个 float（位置 + 法线），UV 暂时弃用：

```cpp
float vertices[] = {
    // 位置                 // 法线
    -0.5f, -0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
     0.5f, -0.5f, -0.5f,  0.0f,  0.0f, -1.0f,
    // ...
};
```

场景里有两类物体：**光源**（白色小立方体）和**被照物体**（10 个旋转立方体），各用一套 shader。`light.vert / light.frag` 只做变换，frag 直接输出白色；`light_obj.vert / light_obj.frag` 负责完整的 Phong 光照计算。

**light_obj.vert** — 把顶点的世界坐标和法线传给片元：

```glsl
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aNormal;

uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;
uniform mat3 normalMatrix;

out vec3 FragPos;
out vec3 Normal;

void main()
{
    gl_Position = projection * view * model * vec4(aPos, 1.0);
    FragPos = vec3(model * vec4(aPos, 1.0));  // 世界空间坐标，供片元计算光向量
    Normal = normalMatrix * aNormal;           // 用法线矩阵变换，避免缩放错误
}
```

**light_obj.frag** — Phong 光照 = 环境光 + 漫反射 + 镜面反射，再乘距离衰减：

```glsl
#version 330 core
out vec4 FragColor;
in vec3 Normal;
in vec3 FragPos;

uniform vec3 lightPos;
uniform vec3 objectColor;
uniform vec3 lightColor;
uniform vec3 viewPos;

void main()
{
    vec3 norm     = normalize(Normal);
    vec3 lightDir = normalize(lightPos - FragPos);
    vec3 viewDir  = normalize(viewPos  - FragPos);

    // 环境光：始终存在的基底亮度
    float ambientStrength = 0.1;
    vec3 ambient = ambientStrength * lightColor;

    // 漫反射：法线与光方向夹角越小，越亮
    float diff    = max(dot(norm, lightDir), 0.0);
    vec3  diffuse = diff * lightColor;

    // 镜面反射：反射方向与视线越对齐，高光越强；128 是光泽度
    float specularStrength = 0.5;
    vec3  reflectDir = reflect(-lightDir, norm);
    float spec     = pow(max(dot(viewDir, reflectDir), 0.0), 128);
    vec3  specular = specularStrength * spec * lightColor;

    // 距离衰减：二次曲线，远处光强迅速下降
    float distance    = length(lightPos - FragPos);
    float attenuation = 1.0 / (1.0 + 0.09 * distance + 0.032 * distance * distance);

    diffuse  *= attenuation;
    specular *= attenuation;

    vec3 result = (ambient + diffuse + specular) * objectColor;
    FragColor = vec4(result, 1.0);
}
```

衰减参数 `constant=1`、`linear=0.09`、`quadratic=0.032` 对应约 50 单位的有效照射距离，参考 [learnopengl-cn](https://learnopengl-cn.github.io/02%20Lighting/05%20Light%20casters/)。

模型矩阵里如果有非均匀缩放，直接用 `model * normal` 变换法线会出错——法线不再垂直于表面。正确做法是用**法线矩阵**（模型矩阵左上 3×3 的逆转置）：

```cpp
glm::mat3 normalMatrix = glm::mat3(glm::transpose(glm::inverse(trans)));
shader.setMat3("normalMatrix", normalMatrix);
```


## User Pointer 冲突修复

`Window` 和 `Camera` 都想用 `glfwSetWindowUserPointer` 存自己的指针，但 GLFW 每个窗口只有一个 user pointer，后设置的会覆盖前者，导致回调崩溃。

创建一个上下文结构体，把所有需要通过 user pointer 传递的对象打包进去：

```cpp
// WindowContext.h
struct WindowContext {
    Window* window;
    Camera* camera;
};
```

然后修改各处的设置和获取逻辑。

**Window.cpp — 初始化时存入结构体**

```cpp
Window::Window(int width, int height, const std::string& title) {
    // ... 原有初始化 ...

    m_context.window = this;
    m_context.camera = nullptr;

    glfwSetWindowUserPointer(glfw_window, &m_context);
    glfwSetFramebufferSizeCallback(glfw_window, framebuffer_size_callback);
}
```

**Window.cpp — 回调里从结构体取 window**

```cpp
void Window::framebuffer_size_callback(GLFWwindow* glfw_window, int width, int height) {
    auto* ctx = static_cast<WindowContext*>(glfwGetWindowUserPointer(glfw_window));
    if (!ctx || !ctx->window) return;

    Window* window = ctx->window;
    window->width = width;
    window->height = height;
    if (window->m_ResizeCallback)
        window->m_ResizeCallback(width, height);
    glViewport(0, 0, width, height);
}
```

**Camera.cpp — 回调里从结构体取 camera**

```cpp
void mouse_callback(GLFWwindow* glfw_window, double xpos, double ypos) {
    auto* ctx = static_cast<WindowContext*>(glfwGetWindowUserPointer(glfw_window));
    if (!ctx || !ctx->camera) return;

    ctx->camera->ProcessMouseMovement(glfw_window, xpos, ypos);
}
```

**Camera::ProcessInput — 不再重复 SetWindowUserPointer**

```cpp
void Camera::ProcessInput(GLFWwindow* window) {
    if (glfwGetMouseButton(window, GLFW_MOUSE_BUTTON_2) == GLFW_PRESS) {
        // glfwSetWindowUserPointer(window, this);  ❌ 删掉这行

        glfwSetInputMode(window, GLFW_CURSOR, GLFW_CURSOR_DISABLED);
        glfwSetCursorPosCallback(window, mouse_callback);
        view = glm::lookAt(position, position + direction, up);
    } else {
        glfwSetInputMode(window, GLFW_CURSOR, GLFW_CURSOR_NORMAL);
        first_mouse = true;
    }
}
```

之后无论有多少系统需要在回调中访问，都只需要扩展结构体，而不会互相覆盖。


## 屏蔽输入法

GLFW 窗口默认关联了 Windows 的输入法上下文（`HIMC`），按键会先经过 IME 处理，导致游戏内输入出现延迟或弹出候选框。用 `ImmAssociateContextEx` 传入 `NULL` 解除关联，键盘消息就直接进入消息队列：

```cpp
// 在文件顶部，必须在 glfw 之前 include
#define GLFW_EXPOSE_NATIVE_WIN32
#include <GLFW/glfw3native.h>

#include <windows.h>
#include <imm.h>
#pragma comment(lib, "imm32.lib")
```

```cpp
Window::Window(int width, int height, const std::string& title) {
    // ...
    glfw_window = glfwCreateWindow(...);

    #ifdef _WIN32
    HWND hwnd = glfwGetWin32Window(glfw_window);
    ImmAssociateContextEx(hwnd, NULL, IACE_IGNORENOCONTEXT);
    #endif
    // ...
}
```

`#ifdef _WIN32` 用于跨平台兼容——`_WIN32` 是编译器在 Windows 下自动注入的宏，Mac/Linux 没有，包裹后非 Windows 平台的编译器直接跳过这段，避免 `HWND` 未定义的编译错误。

`#define GLFW_EXPOSE_NATIVE_WIN32` 必须在 `glfw3native.h` 之前定义，因为该头文件用宏控制各平台的函数声明，不定义则 `glfwGetWin32Window` 不会被声明，编译器直接报错。这是**特性开关（Feature Flag）**的设计思路，按需开启，避免 `windows.h` 污染其他平台的命名空间。
