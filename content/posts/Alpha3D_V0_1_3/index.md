---
title: "Alpha3D v0.1.3"
date: 2026-05-27T20:00:00+08:00
draft: false
description: "用漫反射贴图和镜面贴图替换材质的纯色向量，实现基于纹理的 Phong 光照"
tags: ["图形引擎", "渲染", "光照", "纹理"]
categories: ["Alpha3D 更新日志"]
cover: https://zhjh-oss.oss-cn-beijing.aliyuncs.com/Alpha3D_v0_1_3.png
---


## 光照贴图

上个版本的 `Material` 结构体用 `vec3` 描述漫反射和镜面颜色，整个物体是一个统一的颜色。这个版本把它们换成 `sampler2D`，让每个像素从贴图采样，物体表面可以有细节。

**Material 结构体的变化**

```glsl
// 之前：固定颜色
struct Material {
    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
    float shininess;
};

// 现在：贴图采样
struct Material {
    sampler2D diffuse;
    sampler2D specular;
    float shininess;
};
```

`ambient` 也一并去掉，环境光直接用漫反射贴图的采样值代替，这样暗处的颜色和亮处保持一致。

**Fragment Shader 中的采样**

```glsl
in vec2 TexCoords;

// 环境光用漫反射贴图
vec3 ambient = light.ambient * vec3(texture(material.diffuse, TexCoords));

// 漫反射
float diff = max(dot(norm, lightDir), 0.0);
vec3 diffuse = light.diffuse * diff * vec3(texture(material.diffuse, TexCoords));

// 镜面反射用独立的镜面贴图
float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess);
vec3 specular = light.specular * spec * vec3(texture(material.specular, TexCoords));
```

镜面贴图（`specular.png`）控制哪些区域有高光、哪些没有，比起全局统一的 `specularStrength` 更精确。

## 顶点数据加入 UV

顶点从 6 个 float（位置 + 法线）扩展到 8 个 float，末尾追加 UV 坐标：

```cpp
// positions          // normals           // texture coords
-0.5f, -0.5f, -0.5f,  0.0f,  0.0f, -1.0f,  0.0f, 0.0f,
 0.5f, -0.5f, -0.5f,  0.0f,  0.0f, -1.0f,  1.0f, 0.0f,
 // ...
```

`Mesh.cpp` 的 stride 和属性指针随之更新：

```cpp
vertex_count = size / (8 * sizeof(float));

// position (location = 0)
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 8 * sizeof(float), (void*)0);

// normal (location = 1)
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 8 * sizeof(float), (void*)(3 * sizeof(float)));

// texcoord (location = 2)
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, 8 * sizeof(float), (void*)(6 * sizeof(float)));
glEnableVertexAttribArray(2);
```

Vertex Shader 对应新增输入和输出：

```glsl
layout (location = 2) in vec2 aTexCoords;
out vec2 TexCoords;

// main 里：
TexCoords = aTexCoords;
```

## 纹理绑定到 Uniform

`sampler2D` uniform 对应的不是颜色值，而是纹理单元编号。CPU 侧先把贴图绑定到 slot，再把 slot 编号传给 shader：

```cpp
Texture diffuseTexture("textures/diffuse.png");
Texture specularTexture("textures/specular.png");

shader.setInt("material.diffuse",  0);
diffuseTexture.Bind(0);

shader.setInt("material.specular", 1);
specularTexture.Bind(1);
```

## Texture 支持 RGBA

加载 PNG 时需要处理 alpha 通道，原来硬编码 `GL_RGB` 会导致格式错误：

```cpp
GLenum format = (nrChannels == 4) ? GL_RGBA : GL_RGB;
glTexImage2D(GL_TEXTURE_2D, 0, format, m_Width, m_Height, 0, format, GL_UNSIGNED_BYTE, data);
```

根据 `stbi_load` 返回的通道数自动选择格式，JPG 走 RGB，PNG 走 RGBA。
