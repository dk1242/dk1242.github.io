---
layout: post
title:  "Rendering 1 Million spheres: Part 2 (3D basics with OpenGL)"
date:   2025-05-19
---
This post is the continuation of my previous [post](https://dk1242.github.io/2025/05/13/Rendering-1M-spheres-1.html) which is the first part of this series. In this blog, we will cover the textures, 3D basics including lighting and camera.

But before we start, I want to modularize our existing code, so it can follow the OOP principles. 

### Code Refactoring
Till now we are using VBO, VAO, EBO and Shaders. So, we'll create classes for all these objects. Then we'll create a common class `Mesh` to put all these Buffer objects together.

For Buffer objects, we majorly need one unsigned integer variable to reference it, one Bind function, one Unbind function and one last Delete function just for the cleanup.
```cpp
class VBO
{
public:
	// Reference ID of the Buffer Object
	GLuint ID;
	// Constructor that generates a Buffer Object and binds it to some data
	VBO(...);

	void Bind() const; // call glBindBuffer(GL_ARRAY_BUFFER, ID);
	void Unbind(); // call glBindBuffer(GL_ARRAY_BUFFER, 0);
	void Delete() const; // call glDeleteBuffers(1, &ID);
};	
```
In case of VAO, we also need one extra function to link a VBO attribute to VAO.
```cpp
void VAO::LinkAttrib(VBO& VBO, GLuint layout, GLuint numComponents, GLenum type, GLsizeiptr stride, void* offset){
    VBO.Bind();
    glVertexAttribPointer(layout, numComponents, type, GL_FALSE, stride, offset);
    glEnableVertexAttribArray(layout);
    VBO.Unbind();
}
```

Now, we need to create a ShaderClass to handle vertex and fragment shader objects creation, their compilation and then shader program object creation and compilation. Also we need to handle the shader Activation.
```cpp

std::string get_file_contents(const char* filename); // reads the file and returns its content in string

class Shader {
public:
	GLuint ID;
	std::string shaderName;
	Shader(const char* vertexFilePath, const char* fragmentFilePath, const char* shaderName);

	void Activate() const;
	void Delete() const;
private:
	void compileErrors(unsigned int shader, const char* type);
};
```
We can create ShaderClass object like 

```cpp
Shader sphereShader("./sphere.vert", "./sphere_pbr.frag", "PBR");
```
We also have to create a Mesh class which will combine all this.
```cpp
class Mesh {
public:
    std::vector <glm::vec3> vertices;
    std::vector <GLuint> indices;
    
    VAO mainVAO;

	Mesh(std::vector <glm::vec3>& vertices, std::vector <GLuint>& indices);

    void Draw(Shader& shader);
}
```
I'll also define `Mesh()` and `Draw()` functions just for your reference.
```cpp
// Mesh.cpp file
Mesh::Mesh(std::vector<glm::vec3>& vertices, std::vector<GLuint>& indices) // Constructor for Mesh class
{
    Mesh::vertices = vertices;
    Mesh::indices = indices;

    mainVAO.Bind();
    VBO mainVBO(vertices);
    EBO mainEBO(indices);
    mainVAO.LinkAttrib(mainVBO, 0, 3, GL_FLOAT, sizeof(glm::vec3), (void*)(0));
    mainVAO.Unbind();

    mainVBO.Unbind();
    mainEBO.Unbind();
}

void Mesh::Draw(Shader& shader) // Draw function for Mesh Object
{
    shader.Activate();
    mainVAO.Bind();

    glDrawElements(GL_TRIANGLES, indices.size(), GL_UNSIGNED_INT, 0);
}
```
So, we will initialize a Mesh `rectangle` like
```cpp
std::vector<glm::vec3> vertices = {
    glm::vec3(0.5f,  0.5f, 0.0f),
    glm::vec3(0.5f, -0.5f, 0.0f),
    glm::vec3(-0.5f, -0.5f, 0.0f),
    glm::vec3(-0.5f,  0.5f, 0.0f)
};
std::vector<GLuint> indices = {
    0, 1, 3,   // first triangle
    1, 2, 3    // second triangle
};
Mesh rectangle(vertices, indices);
...
...
while(true){
    ...
    ...
    rectangle.Draw(sampleShader);
    ...
    ...
}
```
> Todo: Add link to source code

### Some more basics of Shaders 
Till now we know how to pass data to vertex shader. But to pass data from vertex shader to fragment shader, we need to declare a variable in vertex shader with `out` and then declare it with same name in fragment shader but with `in`.
```glsl
// inside vertex shader
out vec4 color;

// inside fragment shader
in vec4 color;
```
There is one more way to pass data to shaders and that is `Uniforms`. But they works a little differently as they are global for one particular shader progrma, can be accessed in both vertex and fragment shaders, can be accessed and updated at any stage of application.
We can declare them directly with `uniform` like `uniform vec4 color;`
For setting there values from opengl, we need to call some particular functions defined for each particular data type. Better we add them directly inside our SgaderClass class.
```cpp
void setBool(const std::string &name, bool value) const
{         
    glUniform1i(glGetUniformLocation(ID, name.c_str()), (int)value); 
}
void setInt(const std::string &name, int value) const
{ 
    glUniform1i(glGetUniformLocation(ID, name.c_str()), value); // glUniform1i for integer
}
void setFloat(const std::string &name, float value) const
{ 
    glUniform1f(glGetUniformLocation(ID, name.c_str()), value); // glUniform1f for float
} 
// glUniform1fv: for a float vector/array.
// glUniform4f : for 4 floats.
```

## Textures