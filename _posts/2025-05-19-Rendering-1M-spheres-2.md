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
We can understand Textures as just a 2D image which we can paste over our 3D object. Till now we were filling a color over an object by directly passing the RGB values but now we will sample them from a texture and apply those RGB values over that object.

But for that we need to define which part of texture (texture coordinate) maps to which vertex. Each vertex should have texture coordinate associated with it. Texture coordinates have range between 0 and 1 in both x and y axis. The retrieval of texture color using texture coorinates is defined as **Sampling**.

Generally, we defines our texture coordinates in between range of 0 and 1, to completely wrap the texture over the object. But if we define them outside of this range of 0 and 1, by default, they will start repeating. Lets suppose if I define them between 0 and 5, the texture will repeat for 5 times by defualt but we can also assign different behaviours like mirrored repeat, clamping to border and clamping to edge.
We can assign all these properties using `glTexParameteri()`.

For texture filtering, we have two options like `GL_NEAREST` and `GL_LINEAR`. GL_NEAREST is default texture filtering method which results in blocked pattern and will clearly define the edges while GL_LINEAR gives a smooth pattern for edges. We will decide what to use will totally depend on our requirement. 
Generally, we set it based on like when scaling up (magnifying), use GL_LINEAR and for downscaled texture use GL_NEAREST.

We also have option to create mipmaps, which allows us to create different texture for different sizes (mostly of lesser resolutions). It enables us to not use a large resolution texture for the objects which are far from our camera. It will also help in improving the performance.

### Creating a Texture
To upload an image for generating a texture, we will use a library called **stb_image.h**. You can download it from [here](https://github.com/nothings/stb/blob/master/stb_image.h) and then save it to your project. Then create a new cpp file with name `stb.cpp` and include following code:
```cpp
#define STB_IMAGE_IMPLEMENTATION
#include "stb_image.h"
```
Now we will start by creating a Texture class for our application including same bind, unbind and delete function. We also need to set our texture to shader, for that we'll create one more function.
```cpp
class Texture {
public:
	GLuint ID = 0;
    
	Texture(const char* image, GLenum slot, GLenum format, GLenum pixelType);
	
	// Assigns a texture unit to a Shader
	void texUnit(Shader& shader, const char* uniform, GLuint unit);
	// Binds a texture
	void Bind() const;
	// Unbinds a texture
	void Unbind();
	// Deletes a texture
	void Delete() const;
};
```
We will declare our Texture constructor such that it will load our image and create a texture. Textures are generated with `glTexImage2D()` and we need to pass following parameters to it.
* First argument - texture target, which is `GL_TEXTURE_2D` for now
* Second arg - the mipmap level, for which we want to generate this texture
* Third arg - the format in which we want to save texture, it's `RGB` because our image contains that data only
* Fourth and Fifth argument defines the width and height
* Sixth argument will always be 0.
* 7th arg tells the format (RGB) and data type (`GL_UNSIGNED_BYTE`) of our image
* 8th arg is the actual image data

```cpp
Texture::Texture(const char* imagePath, GLenum slot, GLenum format, GLenum pixelType)
{
	int widthImg, heightImg, numColCh;
	stbi_set_flip_vertically_on_load(true);
	unsigned char* bytes = stbi_load(imagePath, &widthImg, &heightImg, &numColCh, 0);
	if (!bytes) {
		std::cerr << "Failed to load HDR texture at path: " << imagePath << std::endl;
		return;
	}
	else {
		std::cout << imagePath << " loaded correctly.\n";
	}
    // genearating and binding texture 
	glGenTextures(1, &ID); // first argument is the number of textures we want to generate, 2nd arg is the address where we want to store them
	glBindTexture(GL_TEXTURE_2D, ID);

    //setting parameters
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_MIRRORED_REPEAT);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);

	glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, widthImg, heightImg, 0, format, pixelType, bytes); 
	glGenerateMipmap(GL_TEXTURE_2D);

	stbi_image_free(bytes);
	glBindTexture(GL_TEXTURE_2D, 0); // unbinding
}
...
void Texture::texUnit(Shader& shader, const char* uniform, GLuint unit)
{
	GLuint uniTex0 = glGetUniformLocation(shader.ID, uniform);
	shader.Activate();
	glUniform1i(uniTex0, unit);
}
```

### Applying Textures
We will pass the texture coordinates like following
```cpp
std::vector<glm::vec2> texCoords = {
    glm::vec2(0.0f, 0.0f),
    glm::vec2(1.0f, 0.0f),
    glm::vec2(0.0f, 1.0f),
    glm::vec2(1.0f, 1.0f)
};
```
Then we need to pass these coordinates to the shader code too. For that we need to create a new VBO and `VBO(glm::vec2)` constructor which supports glm::vec2, bind it with the pre existing VAO and finally link it.
```cpp
mainVAO.LinkAttrib(texCoordVBO, 1, 2, GL_FLOAT, sizeof(glm::vec2), (void*)(0));
```
In shader side, we need to define a layout with location=1 for texture Coordinates and pass this to fragment shader. In fragment shader, we will assign FragColor with texture () function provided by GLSL, in which we need to pass texture sampler and texture coordinates. This texture function return the corresponding color at the texture coordinates, considering the parameters we set.
```GLSL
// In vertex shader
layout (location = 1) in vec2 aTexCoord;
out vec2 TexCoord;
void main(){
    ...
    TexCoord = aTexCoord;
    ...
}
// in fragment shader
in vec2 TexCoord;

uniform sampler2D sampleTexture;
void main()
{
    FragColor = texture(sampleTexture, TexCoord);
}
```
In last, we just need to activate, bind the texture and set the uniform for `sampleTexture`.
```cpp
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, myTexture.ID);

//for setting the texture in shader
mytexture.texUnit(sphereShader, "sampleTexture", 0);
```

Now, you should be able to see a texture over your rectangle.
![alt text](/assets/images/rectTexture.png)