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

## Different Coordinates systems
We usually defines our objects coordinates in a world space, means the object positions can range from -INF to INF in all 3 directions, but our Normalized Device Coordinates ranges from -1.0 to 1.0 in both x and y axis on our screen. There is a sequence we need to follow for transforming the objects from world space to NDC. There are 5 different coordinate systems through which we'll pass:
1. Local space - Local space defines the coordinates space which is local to our object like the origin of our object, any transformation like rotationetc. In ideal case, the origin should be in the center of the object but sometimes we need it at the corners (suppose to rotate it from the corner). These transformation details like scaling, rotation and translation, we can store in the local space, which we will be passed to global space through a model matrix consisting of all these local space details.
2. World/Global space - When we import multiple objects in our application, they will set their position at origin (0, 0, 0) by default. But we dont want this behaviour, so we will change their positions in world space to create our desired scene. We can also perform differnt operations like rotation and scaling in the world space too. It will not affect the local space properties for that object (means if we have 2 instance of same object, the transformations on one will not affect the other).
3. View/Camera/Eye space - When we'll place a camera in our world, it can see only some specific part like in the back, it cannot see. So we need to perform some transformations which will transform the coordinates to our camera space. We'll discuss about these transformation in the **Camera** section
4. Clip space - As we want our final coordinates in between -1.0 and 1.0, we will use a projection matrix that specifies a range of coordinates in each direction, and the objects lying outside this range will not be clipped and mapped in between -1.0 and 1.0 . If we visualize this projection matrix, it will look like a 3d container or frustum, which will hold the objects inside and whatever is outside of it, will not be rendered on the screen. This projection matrix converts 3d coordinates to device coordinates which can be mapped to NDC. 
5. Screen space - In last, the clip space coordinates will be converted to screen space coorinates using viewport transform. The coordinates of range -1.0 and 1.0 will changed to the range define by glViewport i.e. 0 to WIDTH in x-axis and 0 to HEIGHT in y-axis.

So, finally a coordinate in clip space will get transformed as

$
V_{clip} = M_{projection} * M_{view} * M_{model} * V_{local}
$
## Camera
To define a camera in 3D space, we need some particular vectors like 
1. Position vector - it defines the current position of our camera (x, y, z)
2. Direction vector - it defines the direction in which we are currently targeting. It can be calculated by subtracting the position vector and target Position vector and then normalize it.
```cpp
glm::vec3 cameraDirection = glm::normalize(cameraPos - cameraTarget);
```
3. Up axis - It tell the up direction for camera space. As we are taking the up direction in y-axis, it will be (0, 1, 0);
```cpp
glm::vec3 Up = glm::vec3(0.0f, 1.0f, 0.0f);
```
4. Right axis - It represent the positive x-axis of camera space. We can calculate it by doing a cross product between Up vector and Camera Direction vector, it will result in a perpendicular vector to both.
```cpp
glm::vec3 RightDir = glm::normalize(glm::cross(Orientation, Up));
```

Now we need to define two matrices, view and projection matrix, which helps us in converting world coordinates to clip space coordinates. 

The View matrix can be calculated by using `glm::lookAt(position, target, Up)` function.
The projection matrix can be defined in two ways as we have two different types of camera systems - perspective camera (how humans perceives world) and orthographic camera(when everything will have same size with different distance). 
* Perspective projection - Perspective projection is very similar to how humans perceives the world like distant objects will look smaller than the near objects. We will also try to convert our world space coordinates in such a way that it will produce the same result. If you remember we had a `w` parameter from gl_Position, it will increase with the distance from the camera and then the (x, y, z) coordinates will be divided by the corresponding w value, which will result in NDC.
A perspective matrix can be defined using `glm::perspective(FOVangle, aspectRatio, near, far)`. We can visualize a perspective frustum like the following diagram:

![Perspective frustum ](/assets/images/perspective_frustum.png)

* Orthographic projection - Orthographic projection is just like when every object will be of same size regardless of the distance. In this case, the w value will always be 1.0. It can be defined using `glm::ortho(left, right, bottom, top, near, far);`. We can visualize an orthographic frustum like following image:

![Orthographic frustum](/assets/images/orthographic_frustum.png)

Before start using camera here, We'll create a Camera class which have functions to support handling inputs for cmaera movement, updating the viewProjection/camera matrix for shader.
```cpp
class Camera {
public:
	glm::vec3 Position = glm::vec3(0.0f, 0.0f, 10.0f);
	glm::vec3 Orientation = glm::vec3(0.0f, 0.0f, -1.0f);
	glm::vec3 Up = glm::vec3(0.0f, 1.0f, 0.0f);
	glm::vec3 RightDir = glm::normalize(glm::cross(Orientation, Up));
	
	glm::mat4 view;
	glm::mat4 projection;
	glm::mat4 cameraMatrix = glm::mat4(1.0f);

	int width;
	int height;

	Camera(int width, int height, glm::vec3 position);

	// Updates the camera(viewProjection) matrix for the Vertex Shader
	void updateMatrix(float FOVdeg, float nearPlane, float farPlane);
	// Exports the camera matrix to a shader
	void Matrix(Shader& shader, const char* uniform);
	// Handles camera inputs
	void Inputs(GLFWwindow* window);
};
```
We can update cameraMatrix in updateMatrix function and handle keyboard inputs for camera movement in Inputs function.
```cpp
void Camera::updateMatrix(float FOVdeg, float nearPlane, float farPlane)
{
	// Initializes matrices since otherwise they will be the null matrix
	view = glm::mat4(1.0f);
	projection = glm::mat4(1.0f);

	// Makes camera look in the right direction from the right position
	view = glm::lookAt(Position, Position + Orientation, Up); //Position + orientation will give the direction
	// Adds perspective to the scene
	projection = glm::perspective(glm::radians(FOVdeg), (float)width / height, nearPlane, farPlane); // using a perspective camera

	// Sets new camera matrix
	cameraMatrix = projection * view;
}
void Camera::Inputs(GLFWwindow* window)
{
	if (glfwGetKey(window, GLFW_KEY_W) == GLFW_PRESS)
	{
		Position += speed * Orientation; // changes position in the orientation (inside screen, -z axis) direction
	}
	if (glfwGetKey(window, GLFW_KEY_A) == GLFW_PRESS)
	{
		Position += speed * -glm::normalize(glm::cross(Orientation, Up)); // changes position in the left (-x axis) direction
	}
	if (glfwGetKey(window, GLFW_KEY_S) == GLFW_PRESS)
	{
		Position += speed * -Orientation;  // changes position in the opposite (out of screen) orientation (z axis) direction
	}
	if (glfwGetKey(window, GLFW_KEY_D) == GLFW_PRESS)
	{
		Position += speed * glm::normalize(glm::cross(Orientation, Up)); // changes position in the right (x axis) direction
	}
	if (glfwGetKey(window, GLFW_KEY_SPACE) == GLFW_PRESS)
	{
		Position += speed * Up;  // change position in the Up (y axis) direction
	}
	if (glfwGetKey(window, GLFW_KEY_LEFT_CONTROL) == GLFW_PRESS)
	{
		Position += speed * -Up; // change position in the Down (-y axis) direction
	}
}
```
We also have to set this new cameraMatrix in vertex shader uniform using Matrix function.
```cpp
void Camera::Matrix(Shader& shader, const char* uniform)
{
	glUniformMatrix4fv(glGetUniformLocation(shader.ID, uniform), 1, GL_FALSE, glm::value_ptr(cameraMatrix));
}
```
We also need to do some changes in our existing shader code to properly use the camera functionality in our application.
```GLSL
// Vertex shader
// add 2 extra uniforms
uniform mat4 camMatrix;
uniform mat4 model;
...
void main(){
    ...
    gl_Position = camMatrix * vec4(model, 1.0);
}
```
In last, just call Input and updateMatrix function inside your rendering while loop. 
```cpp
while(true){
    ...
    camera.Inputs(window);
    camera.updateMatrix(45.0f, 0.1f, 300.0f);
    ...
}
// Inside Mesh::Draw function
void Mesh::Draw(Shader& shader, Camera& camera){
    shader.Activate();
    ...
    camera.Matrix(shader, "camMatrix");
    ...
}
```
Now you should be able to move in the scene with camera.