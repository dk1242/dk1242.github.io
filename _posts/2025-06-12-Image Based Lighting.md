---
layout: post
title:  "Image Based Lighting"
date:   2025-06-12
---
Image Based Lighting is a technique used in computer graphics and rendering to light objects using the information from the images forming a sorrounding environment as one big light source. IBL uses an image, usually a high dynamic range image (HDRI) or a cubemap environment image, captured from real world to simulate realistic lighting environment.

Before starting first take a look at reflactance equation.

$$
\begin{align*}
L_o(p, \omega_o) = \int_{\Omega} \left( k_d \frac{c}{\pi} + k_s \frac{DFG}{4(\omega_o \cdot n)(\omega_i \cdot n)}  \right) L_i(p, \omega_i) (n \cdot \omega_i) \, d\omega_i
\end{align*}
$$

IBL considers whole environment lighting not just the direct lighting, which makes it possible for objects to look more physically accurate. But with this environment lighting when we have to do the calculation for radiance using reflectance equation, we need to calculate the integral with multiple incoming light directions becuase of potential radiance from all directions. While for direct lighting, we already knew the limited light positions and their respected directions which make it easier to calculate because of radiance from only one direction.

With an environment cubemap, we can visualize each texel (texture pixel) as one single light source and by sampling this cubemap with `texture()` for the direction $\omega_i$, we can get scene's radiance from that direction.
```GLSL
vec3 radiance = texture(skybox, R).rgb;
```
We have to do this smapling for all possible directions $\omega_i$ across the hemisphere $\Omega$ which will become very expensive with each fragment shader call. We can try with precomputing most of the required calculations to solve this integral efficiently.

We can extend the reflectance equation in 2 parts for diffuse and specular IBL separately.

$$
\begin{align*}
L_o(p, \omega_o) = \int_{\Omega} (k_d \frac{c}{\pi}) L_i(p, \omega_i) (n \cdot \omega_i) \, d\omega_i + \int_{\Omega}(k_s \frac{DFG}{4(\omega_o \cdot n)(\omega_i \cdot n)}) L_i(p, \omega_i) (n \cdot \omega_i) \, d\omega_i
\end{align*}
$$

### Diffuse IBL
In the integral for Diffuse IBL, we can move out the constant part $k_d\frac{c}{\pi}$ from the integration. For pre computation, we will assume our $p$ is always at the center of environment cubemap. It will give us an integral which totally depends on $\omega_i$. Now, we can compute a new cubemap which will store the result of diffuse integral for all possible outgoing directions $\omega_o$ using **convolution**.

**Convolution** involves calculating something for each entry in a dataset while factoring in contributions from all other entries within the dataset. Here, for every sample direction in the cubemap, we account for all other sample directions across the hemisphere $\Omega$. 

To perform convolution on this environment cubemap, we will solve the integral for each output direction $\omega_o$ by sampling multiple incoming directions $\omega_i$. Then the radiance from these sampled directions will be averaged out. The hemisphere $\Omega$ used for sampling $\omega_i$ is always aligned with particular $\omega_o$ being convolved.

This pre computed cubemap represents the sum of all indirect diffuse light from the scene that interacts with the surface aligned along the direction $\omega_o$. This type of cubemap is commonly reffered as an irradiance map, as it enables direct sampling of scene's irradiance for any direction $\omega_o$.

### Cubemap
For generating a cubemap, we can either go with a skybox which will have set of 6 images for 6 cube planes or a HDR image. But for implementing PBR, we must have to use HDR images because with normal LDR image skybox, it will have RGB values between 0.0 and 1.0 while HDR have color values outside 0.0 and 1.0 range to give lights with correct intensity.
I have use below HDR image, you can choose whatever you want from freely available HDR images online.
![HDR image used](/assets/images/hdrsample.jpeg)
This image may look a little distorted at some points but its just because this is projected from sphere onto a flat plane. So that we can visualize and store it in a single image known as equirectangular map. 

Now we have to convert this into an environment cubemap. We will load it using our old standard library called stb_image.h which supports loading of .hdr images directly as an array of floating point values. We just have to create new constructor in our pre-existing Texture class which we created in [3D basics](https://dk1242.github.io/2025/05/19/Rendering-1M-spheres-2-3D-basics.html#:~:text=one%20more%20function.-,class%20Texture,-%7B%0Apublic%3A) blog. This constructor will support loading of .hdr image.
```cpp
Texture::Texture(const char* imagePath, GLenum slot)
{
	type = "HDR";

	int width, height, numChannels;
	stbi_set_flip_vertically_on_load(true);
	float* bytes = stbi_loadf(imagePath, &width, &height, &numChannels, 0);

	if (!bytes) {
		std::cerr << "Failed to load HDR texture at path: " << imagePath << std::endl;
		return;
	}
	else {
		std::cout << imagePath << " loaded correctly.\n";
	}

	glGenTextures(1, &ID);
	glActiveTexture(slot);
	glBindTexture(GL_TEXTURE_2D, ID);

	// Load HDR bytes into the texture
	glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB16F, width, height, 0, GL_RGB, GL_FLOAT, bytes);

	// Set texture parameters
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

	stbi_image_free(bytes);
	glBindTexture(GL_TEXTURE_2D, 0);
}
```
```cpp
// In main.cpp
Texture hdrTexture("./textures/night.hdr", GL_TEXTURE0);
```
After loading this HDR image and creating its Texture object, we need to create its cubemap. To convert an equirectangle image to cubemap, we need to first render a cube and project the equirectangle map on all cube's faces and then take 6 images of this cube's sides as cubemap face.

We will create a new class TextureUtilities where we can write different functions like GenerateCubemap(), GenrateIrraidanceMap() and others. We will need a common **cube** (skybox) mesh in this class which will be used for generating these maps and a common View Projection Camera matrix.
```cpp
class TextureUtilities {
public:
    TextureUtilities();

    glm::mat4 captureProjection; // projection Matrix
    std::vector<glm::mat4> captureViews; // vector array of View Matrix (for 6 cube faces)
    std::vector<glm::mat4> cameraMatrix; // Camera Matrix (View x Projection Matrix)

    std::vector<glm::vec3> skyboxVertices = {
        // positions          
        glm::vec3(1.0f,  1.0f, 1.0f),
        glm::vec3(1.0f, -1.0f, 1.0f),
        glm::vec3(-1.0f, -1.0f, 1.0f),
        glm::vec3(-1.0f, 1.0f, 1.0f),
        glm::vec3(1.0f, 1.0f, -1.0f),
        glm::vec3(1.0f, -1.0f, -1.0f),
        glm::vec3(-1.0f, -1.0f, -1.0f),
        glm::vec3(-1.0f, 1.0f, -1.0f),
    };
    std::vector<GLuint> skyboxIndices = {
        0, 1, 2,
        2, 3, 0,

        0, 3, 4,
        3, 4, 7,

        2, 3, 6,
        3, 6, 7,

        0, 1, 5,
        0, 4, 5,

        1, 2, 5,
        2, 5, 6,

        4, 5, 6,
        4, 6, 7
    };
    Mesh* skymap;

	GLuint GenerateCubemap(GLuint &hdrTexture, Shader& shader);
	GLuint GenerateIrradianceMap(GLuint &cubemapTexture, Shader& irradianceShader);
	GLuint GeneratePrefilteredCubemap(GLuint &cubemapTexture, Shader& prefilterShader, GLuint maxMipLevels);
	GLuint GenerateBRDFLUT(Shader &brdfShader);
};
```
Inside `TextureUtilities()` constructor, we will assign `skymap` Mesh* with `new Mesh(skyboxVertices, skyboxIndices)` and also initialize the `cameraMatrix`.
```cpp
TextureUtilities::TextureUtilities()
{
    captureProjection = glm::perspective(glm::radians(90.0f), 1.0f, 0.1f, 10.0f); // fov of 90 degrees to capture the entire face
    
    captureViews.push_back(glm::lookAt(glm::vec3(0.0f), glm::vec3(1.0f, 0.0f, 0.0f), glm::vec3(0.0f, -1.0f, 0.0f))); // Positive X (+X) axis
    captureViews.push_back(glm::lookAt(glm::vec3(0.0f), glm::vec3(-1.0f,  0.0f,  0.0f), glm::vec3(0.0f, -1.0f,  0.0f))); // Negative X (-X) axis
    captureViews.push_back(glm::lookAt(glm::vec3(0.0f), glm::vec3(0.0f, 1.0f, 0.0f), glm::vec3(0.0f, 0.0f, 1.0f))); // Positive Y (+Y) axis
    captureViews.push_back(glm::lookAt(glm::vec3(0.0f), glm::vec3(0.0f, -1.0f, 0.0f), glm::vec3(0.0f, 0.0f, -1.0f))); // Negative Y (-Y) axis
    captureViews.push_back(glm::lookAt(glm::vec3(0.0f), glm::vec3(0.0f, 0.0f, 1.0f), glm::vec3(0.0f, -1.0f, 0.0f))); // Positive Z (+Z) axis
    captureViews.push_back(glm::lookAt(glm::vec3(0.0f), glm::vec3(0.0f, 0.0f, -1.0f), glm::vec3(0.0f, -1.0f, 0.0f)));  // Negative Z (-Z) axis

    cameraMatrix.resize(6, glm::mat4(1.0f));

    for (int i = 0; i < 6; i++) {
        cameraMatrix[i] = captureProjection * captureViews[i];
    }
    
    skymap = new Mesh(skyboxVertices, skyboxIndices);
}
```
Before we start writing `GenerateCubeMap()`, we need to create new vertex and fragment shaders which will help in projecting equirectangle map onto each side of cubemap.
Our vertex shader will not modify anything, it will just pass the local positions to fragment shaders as it is.
```GLSL
// cubeMapShader.vert
#version 330 core
layout (location = 0) in vec3 aPos;

out vec3 localPos;

uniform mat4 camMatrix; // View x Projection Matrix

void main()
{
    localPos = aPos;
    gl_Position = camMatrix * vec4(aPos, 1.0);
}
```
In fragment shader, to project equirectangular map onto the cube faces, we will convert `localPos` 3D vector into spherical UV coordinate system and will use it to sample the texture.
```GLSL
#version 330 core

in vec3 localPos; // local position passed from vertex shader
out vec4 FragColor;

uniform sampler2D hdrTexture; // equirectangular map

const vec2 invAtan = vec2(0.1591, 0.3183); // constants used for inverse trigonometric scaling to map spherical coordinates to UV space

vec2 SampleSphericalMap(vec3 v)
{
    vec2 uv = vec2(atan(v.z, v.x), asin(v.y)); // atan calculates longitude and asin calulates latitude
    uv *= invAtan; // scaling to fit the UV space
    uv += 0.5; // offset for maintaining the UV range
    return uv;
}

void main() {
    vec2 uv = SampleSphericalMap(normalize(localPos)); // calculate uv coordinates
    vec3 color = texture(hdrTexture, uv).rgb; // sample the hdrtexture to retrieve color at the uv direction
    
    FragColor = vec4(color, 1.0);
}
```
Now we just have to create a shaderClass object with these 2 shaders and pass it to GenerateCubemap() function.
```cpp
Shader cubemapShader("./cubemapShader.vert", "./cubemapShader.frag", "cubemap");
cubemapShader.Activate();
...
GLuint cubemapTexture = textureUtilitiesObject.GenerateCubemap(hdrTexture.ID, cubemapShader);
```
Now we can start writing `GenerateCubemap` function which will genrate a cubemap from provided HDR texture and this shader. We will start with `cubemapTexture` initialization, allocating memory for each cube face, setting some standard texture parameters. Then we will setup the framebuffer and renderbuffer objects and bind the hdr texture and shader. At last, we will render this texture onto each cube map face.
```cpp
GLuint TextureUtilities::GenerateCubemap(GLuint& hdrTexture, Shader &shader)
{
    GLuint cubemapTexture;
    glGenTextures(1, &cubemapTexture);
    glBindTexture(GL_TEXTURE_CUBE_MAP, cubemapTexture); // GL_TEXTURE_CUBE_MAP for storing 6 2d images corresponding to cube faces

    for (GLuint i = 0; i < 6; ++i) {
        glTexImage2D(GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, 0, GL_RGB16F, 512, 512, 0, GL_RGB, GL_FLOAT, nullptr); //Memory allocation
    }

    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    ... // other parameters

    GLuint captureFBO, captureRBO;
    glGenFramebuffers(1, &captureFBO);
    glGenRenderbuffers(1, &captureRBO); // render buffer will store the depth information

    glBindFramebuffer(GL_FRAMEBUFFER, captureFBO);
    glBindRenderbuffer(GL_RENDERBUFFER, captureRBO);
    glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH_COMPONENT24, 512, 512);
    glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_RENDERBUFFER, captureRBO);
  
    glViewport(0, 0, 512, 512);
    
    shader.Activate();
    glActiveTexture(GL_TEXTURE0);
    glBindTexture(GL_TEXTURE_2D, hdrTexture);
    glUniform1i(glGetUniformLocation(shader.ID, "hdrTexture"), 0); // assigning shader uniform hdrTexture 

    glBindFramebuffer(GL_FRAMEBUFFER, captureFBO);
    
        
    for (GLuint i = 0; i < 6; ++i) {
        glUniformMatrix4fv(glGetUniformLocation(shader.ID, "camMatrix"), 1, GL_FALSE, glm::value_ptr(cameraMatrix[i])); // setting uniform cameraMatrix with specific cameraMatrix[i]
        
        glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, cubemapTexture, 0); // attaching frame buffer with current cubemap face(+/-X, +/-Y, +/-Z) and texture image
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
        
        skymap->Draw(shader);
    }

    glBindFramebuffer(GL_FRAMEBUFFER, 0);
    return cubemapTexture;
}
```
The `Draw` function is defined as
```cpp
void Mesh::Draw(Shader& shader) const
{
    mainVAO.Bind();

    glDrawElements(GL_TRIANGLES, GLsizei(indices.size()), GL_UNSIGNED_INT, 0);
}
```
It should genearate a cubemap like following:
![Cube map images](/assets/images/cubemap.png)
I took this screenshot from Nvidia's Nsight Graphics 2025.2.0 Frame debugger window. It's a great tool for debugging, resource utilization info, profiling and other activities.

We can also render this cubemap to form our skymap. For that we will create new shaders. The vertex shader, skymap.vert will just pass the localPos to fragment shader.
```GLSL
#version 330 core
layout (location = 0) in vec3 aPos;

uniform mat4 camMatrix;

out vec3 localPos;

void main()
{
    localPos = aPos;
    vec4 pos = camMatrix * vec4(aPos, 1.0);
    gl_Position = pos.xyww; // Depth is set far away 
}
```
The fragment shader will sample the cubemap with localPos and then will apply the standard [gamma correction](https://learnopengl.com/Advanced-Lighting/Gamma-Correction).
```GLSL
#version 330 core
in vec3 localPos;

uniform samplerCube skybox;

out vec4 FragColor;

void main()
{
    vec3 color = texture(skybox, localPos).rgb;

    color = color / (color + vec3(1.0));
    color = pow(color, vec3(1.0/2.2)); 

    FragColor = vec4(color, 1.0);
}
```
Now, create a skybox's ShaderClass object and a separate Mesh for rendering.
```cpp
// In main.cpp
//*****************-------SkyMap----------*****************

Shader skymapShader("./skymap.vert", "./skymap.frag", "skybox");
skymapShader.Activate();

std::vector<glm::vec3> skyboxVertices = {
    // positions          
    glm::vec3(300.0f,  300.0f, 300.0f),
    glm::vec3(300.0f, -300.0f, 300.0f),
    ...
};
std::vector<GLuint> skyboxIndices = {
    0, 1, 2,
    2, 3, 0,
    ...
};

Mesh skymap(skyboxVertices, skyboxIndices);
...
while(true){
    ...
    glDepthFunc(GL_LEQUAL); // to pass when incoming depth is less or equal to stored
    skymap.Draw(skymapShader, camera, cubemapTexture);
    glDepthFunc(GL_LESS);
    ...
}
```
It should render like a skybox with continous looking environment.
![Skybox](/assets/images/skybox.png)

So, we are done with generating a cubemap from a HDR image and its rendering in our application. Now we have to generate the pre discussed Irradiance map.

### Irradiance Map
