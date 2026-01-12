---
title: 模型加载与渲染
link: model_loading
catalog: true
date: 2026-01-12 10:26:00
description: 施工中的第一篇文章，加载并且渲染模型什么的简单的东西。
tags:
  - 计算机图形学
  - OpenGL
  - 模型加载
categories:
  - 笔记
  - 图形学
---

本文将简单介绍如何在 OpenGL 图形学程序 中使用 [assimp](https://github.com/assimp/assimp) 读取加载 OBJ 模型，并且渲染出来。

## 什么是 OBJ 模型

OBJ 文件是 Wavefront 公司为它的一套基于工作站的 3D 建模和动画软件"Advanced Visualizer"开发的一种文件格式，其是一种存储了定点坐标、三角面等信息的文本文件，可以直接用写字板打开进行查看和编辑修改。此外 OBJ 文件需要`*.mtl`文件来存储漫反射、高光、金属度等纹理信息。

OBJ 格式支持多边形(Polygon)，直线(Lines)，表面(Surfaces)，和自由形态曲线(Free-form Curves)。直线和多角形通过它们的点来描述，曲线和表面则根据于它们的控制点和依附于曲线类型的额外信息来定义。这些信息支持规则和不规则的曲线，包括那些基于贝塞尔(Bezier)曲线，B 样条(B-spline)，基数(Cardinal/Catmull-Rom 样条)，和泰勒方程(Taylor equations)的曲线。

## Assimp

The Open Asset Importer Library，OpenAssimp/Assimp，是一个用于导入各种 3D 模型文件进入内存的 C++库（也可以导出部分格式），基于 BSD 协议发布，并且额外提供了一套 C 语言 API。它会将所有的模型数据加载至 Assimp 的通用数据结构中。当 Assimp 加载完模型之后，我们就能够从 Assimp 的数据结构中提取我们所需的所有数据了。由于 Assimp 的数据结构保持不变，不论导入的是什么种类的文件格式，它都能够将我们从这些不同的文件格式中抽象出来，用同一种方式访问我们需要的数据。

跟建模时的流程类似，它通常会将整个模型加载进一个场景(Scene)对象，它会包含导入的模型/场景中的所有数据。Assimp 会将场景载入为一系列的节点(Node)，每个节点包含了场景对象中所储存数据的索引，每个节点都可以有任意数量的子节点。Assimp 数据结构的（简化）模型如下：

![图片](/img/pic/cg/assimp_structure.webp)


- 和材质和网格(Mesh)一样，所有的场景/模型数据都包含在 Scene 对象中。Scene 对象也包含了场景根节点的引用。

- 场景的 Root node（根节点）可能包含子节点（和其它的节点一样），它会有一系列指向场景对象中 mMeshes 数组中储存的网格数据的索引。Scene 下的 mMeshes 数组储存了真正的 Mesh 对象，节点中的 mMeshes 数组保存的只是场景中网格数组的索引。

- 一个 Mesh 对象本身包含了渲染所需要的所有相关数据，像是顶点位置、法向量、纹理坐标、面(Face)和物体的材质。

- 一个网格包含了多个面。Face 代表的是物体的渲染图元(Primitive)（三角形、方形、点）。一个面包含了组成图元的顶点的索引。由于顶点和索引是分开的，使用一个索引缓冲来渲染是非常简单的（见你好，三角形）。

- 最后，一个网格也包含了一个 Material 对象，它包含了一些函数能让我们获取物体的材质属性，比如说颜色和纹理贴图（比如漫反射和镜面光贴图）。

## 测试

这是一段测试代码喵。

`model.hpp`
```cpp
#include <assimp/Importer.hpp>
#include <assimp/scene.h>
#include <assimp/postprocess.h>
#include <glm/glm.hpp>
#include <shader.hpp>
#include <vector>

struct vertex
{
    glm::vec3 position;
    glm::vec3 normal;
    glm::vec2 tex_coords;
};

enum class texture_type
{
    diffuse,
    specular,
    normal
};

struct texture
{
    GLuint id;
    texture_type type;
    aiString path;
};

class mesh
{
public:
    std::vector<vertex> vertices;
    std::vector<GLuint> indices;
    std::vector<texture> textures;

    mesh(std::vector<vertex> vertices, const std::vector<GLuint>& indices, const std::vector<texture>& textures);
    void draw(shader& shader_program);

    void bind_instance_vao(GLuint location);

    void draw_instanced(size_t instance_amount, shader& shader_program);

private:
    GLuint VAO, VBO, EBO;
    void setup_mesh();
};

class model
{
public:
    model(const std::string& path, bool gamma = false)
        : gamma_correction(gamma)
    {
        load_model(path);
    }

    void draw(shader& shader_program);

    void bind_instance_buffer(GLuint location, const std::vector<glm::mat4>& instance_matrices);

    void draw_instanced(size_t instance_amount, shader& shader_program);

    inline void debug_print()
    {
        printf("model debug info:\n");
        for(const auto& m : meshes)
        {
            printf(" mesh: %zu vertices, %zu indices, %zu textures\n", m.vertices.size(), m.indices.size(), m.textures.size());
            for(const auto& t : m.textures)
            {
                printf("  texture id: %u, type: %d, path: %s\n", t.id, static_cast<int>(t.type), t.path.C_Str());
            }
        }
        printf(" loaded %zu textures\n", loaded_textures.size());
        for(const auto& t : loaded_textures)
        {
            printf("  texture id: %u, type: %d, path: %s\n", t.id, static_cast<int>(t.type), t.path.C_Str());
        }
        printf(" directory: %s\n", directory.c_str());
    }

private:
    std::vector<mesh> meshes;
    std::string directory;
    std::vector<texture> loaded_textures;
    bool gamma_correction;

    void load_model(const std::string& path);
    void process_node(aiNode* node, const aiScene* scene);
    mesh process_mesh(aiMesh* mesh, const aiScene* scene);
    std::vector<texture> load_material_textures(aiMaterial* mat, aiTextureType type, texture_type tex_type);
};
```

`model.cpp`
```cpp
#include <assimp/material.h>
#include <model.hpp>
#include <gfx.hpp>

mesh::mesh(std::vector<vertex> vertices, const std::vector<GLuint>& indices, const std::vector<texture>& textures)
{
    this->vertices = std::move(vertices);
    this->indices = std::move(indices);
    this->textures = std::move(textures);

    setup_mesh();
}

void model::draw(shader& shader_program)
{
    shader_program.use();
    for(auto& m : meshes)
        m.draw(shader_program);
}

void model::load_model(const std::string& path)
{
    Assimp::Importer importer;
    const auto scene = importer.ReadFile(path, aiProcess_Triangulate | aiProcess_GenSmoothNormals | aiProcess_FlipUVs | aiProcess_CalcTangentSpace);
    if(!scene || scene->mFlags & AI_SCENE_FLAGS_INCOMPLETE || !scene->mRootNode)
    {
        printf("ERROR::ASSIMP::%s\n", importer.GetErrorString());
        return;
    }
    directory = path.substr(0, path.find_last_of('/'));
    process_node(scene->mRootNode, scene);
}

void model::process_node(aiNode* node, const aiScene* scene)
{
    for(GLuint i = 0; i < node->mNumMeshes; ++i)
    {
        const auto mesh = scene->mMeshes[node->mMeshes[i]];
        meshes.emplace_back(process_mesh(mesh, scene));
    }
    for(GLuint i = 0; i < node->mNumChildren; ++i)
    {
        process_node(node->mChildren[i], scene);
    }
}

mesh model::process_mesh(aiMesh* mesh, const aiScene* scene)
{
    std::vector<vertex> vertices;
    std::vector<GLuint> indices;
    std::vector<texture> textures;

    for(GLuint i = 0; i < mesh->mNumVertices; ++i)
    {
        vertex v;
        v.position = glm::vec3(mesh->mVertices[i].x, mesh->mVertices[i].y, mesh->mVertices[i].z);
        v.normal = glm::vec3(mesh->mNormals[i].x, mesh->mNormals[i].y, mesh->mNormals[i].z);
        if(mesh->mTextureCoords[0])
            v.tex_coords = glm::vec2(mesh->mTextureCoords[0][i].x, mesh->mTextureCoords[0][i].y);
        else
            v.tex_coords = glm::vec2(0.0f, 0.0f);
        vertices.emplace_back(v);
    }
    for(GLuint i = 0; i < mesh->mNumFaces; ++i)
    {
        const auto face = mesh->mFaces[i];
        for(GLuint j = 0; j < face.mNumIndices; ++j)
            indices.push_back(face.mIndices[j]);
    }
    if(mesh->mMaterialIndex >= 0)
    {
        const auto material = scene->mMaterials[mesh->mMaterialIndex];
        std::vector<texture> diffuse_maps = load_material_textures(material, aiTextureType_DIFFUSE, texture_type::diffuse);
        textures.insert(textures.end(), diffuse_maps.begin(), diffuse_maps.end());
        std::vector<texture> specular_maps = load_material_textures(material, aiTextureType_SHININESS, texture_type::specular);
        textures.insert(textures.end(), specular_maps.begin(), specular_maps.end());
        std::vector<texture> normal_maps = load_material_textures(material, aiTextureType_HEIGHT, texture_type::normal);
        textures.insert(textures.end(), normal_maps.begin(), normal_maps.end());
    }
    return {vertices, indices, textures};
}

std::vector<texture> model::load_material_textures(aiMaterial* mat, aiTextureType type, texture_type tex_type)
{
    std::vector<texture> textures;
    for(GLuint i = 0; i < mat->GetTextureCount(type); ++i)
    {
        aiString str;
        mat->GetTexture(type, i, &str);
        bool skip = false;
        for(const auto& loaded_tex : loaded_textures)
        {
            if(std::strcmp(loaded_tex.path.C_Str(), str.C_Str()) == 0)
            {
                textures.push_back(loaded_tex);
                skip = true;
                break;
            }
        }
        if(!skip)
        {
            texture tex;
            tex.id = load_dds((directory + '/' + str.C_Str()).c_str());
            tex.type = tex_type;
            tex.path = str;
            textures.push_back(tex);
            loaded_textures.push_back(tex);

            GLenum error = glGetError();
            if(error != GL_NO_ERROR)
            {
                printf("OpenGL error after loading material %s: %d\n", str.C_Str(), error);
            }
        }
    }
    return textures;
}

void model::bind_instance_buffer(GLuint location, const std::vector<glm::mat4>& instance_matrices)
{
    GLuint buffer;
    glGenBuffers(1, &buffer);
    glBindBuffer(GL_ARRAY_BUFFER, buffer);
    glBufferData(GL_ARRAY_BUFFER, instance_matrices.size() * sizeof(glm::mat4), &instance_matrices[0], GL_DYNAMIC_DRAW);

    for(auto& m : meshes)
    {
        m.bind_instance_vao(location);
    }
}

void model::draw_instanced(size_t instance_amount, shader& shader_program)
{
    shader_program.use();
    for(auto &m : meshes)
    {
        m.draw_instanced(instance_amount, shader_program);
    }
}

void mesh::draw(shader& shader_program)
{
    GLuint diffuse_index = 1;
    GLuint specular_index = 1;
    GLuint normal_index = 1;

    for(GLint i = 0; i < textures.size(); ++i)
    {
        glActiveTexture(GL_TEXTURE0 + i);
        std::string number;
        std::string name;
        switch(textures[i].type)
        {
        case texture_type::diffuse:
            name = "texture_diffuse";
            number = std::to_string(diffuse_index++);
            break;
        case texture_type::specular:
            name = "texture_specular";
            number = std::to_string(specular_index++);
            break;
        case texture_type::normal:
            name = "texture_normal";
            number = std::to_string(normal_index++);
            break;
        }
        shader_program.set_uniform(("material_" + name + number).c_str(), i);
        glBindTexture(GL_TEXTURE_2D, textures[i].id);
    }
    glActiveTexture(GL_TEXTURE0);

    glBindVertexArray(VAO);
    glDrawElements(GL_TRIANGLES, static_cast<GLsizei>(indices.size()), GL_UNSIGNED_INT, 0);
    glBindVertexArray(0);
}

void mesh::setup_mesh()
{
    glGenVertexArrays(1, &VAO);
    glGenBuffers(1, &VBO);
    glGenBuffers(1, &EBO);

    glBindVertexArray(VAO);
    glBindBuffer(GL_ARRAY_BUFFER, VBO);
    glBufferData(GL_ARRAY_BUFFER, vertices.size() * sizeof(vertex), &vertices[0], GL_STATIC_DRAW);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, indices.size() * sizeof(GLuint), &indices[0], GL_STATIC_DRAW);

    // vertex pos
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, sizeof(vertex), (void*)0);
    // vertex normal
    glEnableVertexAttribArray(1);
    glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, sizeof(vertex), (void*)offsetof(vertex, normal));
    // vertex texture coords
    glEnableVertexAttribArray(2);
    glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, sizeof(vertex), (void*)offsetof(vertex, tex_coords));

    glBindVertexArray(0);
}

void mesh::bind_instance_vao(GLuint location)
{
    GLuint _VAO = this->VAO;
    glBindVertexArray(_VAO);
    GLsizei vec4size = sizeof(glm::vec4);
    glEnableVertexAttribArray(location);
    glVertexAttribPointer(location, 4, GL_FLOAT, GL_FALSE, 4 * vec4size, (void*)0);
    glEnableVertexAttribArray(location + 1);
    glVertexAttribPointer(location + 1, 4, GL_FLOAT, GL_FALSE, 4 * vec4size, (void*)(1 * vec4size));
    glEnableVertexAttribArray(location + 2);
    glVertexAttribPointer(location + 2, 4, GL_FLOAT, GL_FALSE, 4 * vec4size, (void*)(2 * vec4size));
    glEnableVertexAttribArray(location + 3);
    glVertexAttribPointer(location + 3, 4, GL_FLOAT, GL_FALSE, 4 * vec4size, (void*)(3 * vec4size));

    glVertexAttribDivisor(location, 1);
    glVertexAttribDivisor(location + 1, 1);
    glVertexAttribDivisor(location + 2, 1);
    glVertexAttribDivisor(location + 3, 1);
    glBindVertexArray(0);
}

void mesh::draw_instanced(size_t instance_amount, shader& shader_program)
{
    GLuint diffuse_index = 1;
    GLuint specular_index = 1;
    GLuint normal_index = 1;

    for(GLint i = 0; i < textures.size(); ++i)
    {
        glActiveTexture(GL_TEXTURE0 + i);
        std::string number;
        std::string name;
        switch(textures[i].type)
        {
        case texture_type::diffuse:
            name = "texture_diffuse";
            number = std::to_string(diffuse_index++);
            break;
        case texture_type::specular:
            name = "texture_specular";
            number = std::to_string(specular_index++);
            break;
        case texture_type::normal:
            name = "texture_normal";
            number = std::to_string(normal_index++);
            break;
        }
        shader_program.set_uniform(("material_" + name + number).c_str(), i);
        glBindTexture(GL_TEXTURE_2D, textures[i].id);
    }
    glActiveTexture(GL_TEXTURE0);

    glBindVertexArray(VAO);
    glDrawElementsInstanced(GL_TRIANGLES, static_cast<GLsizei>(indices.size()), GL_UNSIGNED_INT, 0, static_cast<GLsizei>(instance_amount));
    glBindVertexArray(0);
}

```

