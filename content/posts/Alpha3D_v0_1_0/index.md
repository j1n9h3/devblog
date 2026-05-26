---
title: "Alpha3D v0.1.1"
date: 2026-05-26
draft: false
description: ""
tags: ["更新日志"]
categories: ["Alpha3D 更新日志"]
---


## 日志系统更新

采用 X-Macro 设计，实现数据和行为分离。

通常写代码，数据和行为是绑在一起的：


```C++
// Log.h
class Log {
public:
    static void Init();
    static void Shutdown();

    // 每个模块手写 getter
    static std::shared_ptr<spdlog::logger>& GetWindowLogger()  { return s_WindowLogger;  }
    static std::shared_ptr<spdlog::logger>& GetShaderLogger()  { return s_ShaderLogger;  }
    static std::shared_ptr<spdlog::logger>& GetTextureLogger() { return s_TextureLogger; }
    static std::shared_ptr<spdlog::logger>& GetSceneLogger()   { return s_SceneLogger;   }
    static std::shared_ptr<spdlog::logger>& GetMeshLogger()    { return s_MeshLogger;    }
    static std::shared_ptr<spdlog::logger>& GetConfigLogger()  { return s_ConfigLogger;  }

private:
    // 每个模块手写声明
    static std::shared_ptr<spdlog::logger> s_WindowLogger;
    static std::shared_ptr<spdlog::logger> s_ShaderLogger;
    static std::shared_ptr<spdlog::logger> s_TextureLogger;
    static std::shared_ptr<spdlog::logger> s_SceneLogger;
    static std::shared_ptr<spdlog::logger> s_MeshLogger;
    static std::shared_ptr<spdlog::logger> s_ConfigLogger;
};
```

X-Macro 把数据单独抽出来：

```C++
// Log.h
#define LOG_MODULES \
    X(Window) \
    X(Shader) \
    X(Texture) \
    X(Scene) \
    X(Mesh) \
    X(Config)

class Log {
public:
    static void Init();
    static void Shutdown();

    #define X(name) \
        static std::shared_ptr<spdlog::logger>& Get##name##Logger() { return s_##name##Logger; }
        LOG_MODULES
    #undef X

private:
    #define X(name) static std::shared_ptr<spdlog::logger> s_##name##Logger;
        LOG_MODULES
    #undef X
};
```

增加数据的时候不影响行为，尽可能地复用代码。


## 相机移动实现

右键按住时激活相机控制，WASD 移动相机，松开恢复正常鼠标


<video src="https://zhjh-oss.oss-cn-beijing.aliyuncs.com/Alpha3D_01.mp4" controls width="460px"></video>


```C++
void Camera::ProcessInput(GLFWwindow* window)
{
    if (glfwGetMouseButton(window, GLFW_MOUSE_BUTTON_2) == GLFW_PRESS) {
        float cameraSpeed = 0.05f; // adjust accordingly
        if (glfwGetKey(window, GLFW_KEY_W) == GLFW_PRESS)
            position += cameraSpeed * direction;
        if (glfwGetKey(window, GLFW_KEY_S) == GLFW_PRESS)
            position -= cameraSpeed * direction;
        if (glfwGetKey(window, GLFW_KEY_A) == GLFW_PRESS)
            position -= glm::normalize(glm::cross(direction, up)) * cameraSpeed;
        if (glfwGetKey(window, GLFW_KEY_D) == GLFW_PRESS)
            position += glm::normalize(glm::cross(direction, up)) * cameraSpeed;

        if (glfwGetKey(window, GLFW_KEY_D) == GLFW_PRESS)
            glfwSetInputMode(window, GLFW_CURSOR, GLFW_CURSOR_DISABLED);

        glfwSetWindowUserPointer(window, this);

        glfwSetInputMode(window, GLFW_CURSOR, GLFW_CURSOR_DISABLED);
        glfwSetCursorPosCallback(window, mouse_callback);

        view = glm::lookAt(position, position + direction, up);
    }
    else {
        glfwSetInputMode(window, GLFW_CURSOR, GLFW_CURSOR_NORMAL);
        this->first_mouse = true;
    }
}
```

GLFW 要求 注册函数 签名满足 `mouse_callback(GLFWwindow* window, double xpos, double ypos)`，在计算 `lastX` 和 `lastY` 时候需要 `Window` 的长和宽用于计算中间值，用回调函数把 window 变量保存在 GLFW 的 user_pointer 指针中。


```C++
void mouse_callback(GLFWwindow* window, double xpos, double ypos)
{
    Camera* camera = static_cast<Camera*>(glfwGetWindowUserPointer(window));
    camera->ProcessMouseMovement(window, xpos, ypos);
}
```

计算相机朝向参考 [learnopengl-cn](https://learnopengl-cn.github.io/01%20Getting%20started/09%20Camera/#_6)。



```C++
void Camera::ProcessMouseMovement(GLFWwindow* window, double xpos, double ypos) {

    if (first_mouse) {
        lastX = xpos;
        lastY = ypos;
        first_mouse = false;
    }

    float xoffset = xpos - lastX;
    float yoffset = lastY - ypos;
    lastX = xpos;
    lastY = ypos;

    float sensitivity = 0.05f;
    xoffset *= sensitivity;
    yoffset *= sensitivity;

    this->yaw += xoffset;
    this->pitch += yoffset;

    if (this->pitch > 89.0f)
        this->pitch = 89.0f;
    if (this->pitch < -89.0f)
        this->pitch = -89.0f;

    glm::vec3 front;
    front.x = cos(glm::radians(pitch)) * cos(glm::radians(yaw));
    front.y = sin(glm::radians(pitch));
    front.z = cos(glm::radians(pitch)) * sin(glm::radians(yaw));
    this->direction = glm::normalize(front);
}
```
