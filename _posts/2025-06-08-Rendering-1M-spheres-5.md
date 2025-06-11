---
layout: post
title:  "Rendering 1 Million spheres: Part 5 (Frustum Culling & Level of Detail in GPU)"
date:   2025-06-08
---
In this blog, we will understand and implement a compute shader to calculate the Frustum culling and LOD, which will run in GPU.

When we are going to transfer our calculation to compute shader, we also need to pass the input data and then send it back to CPU before calling `glDrawElementInstanced()`. 

We will use **Shader Storage Buffer Objects** (SSBOs) for trasferring this input and output data. SSBOs allows shaders to access large amount of data stored in buffer objects and provides a easy way to share large arrays between CPU and GPU.
We will also need some Atomic Counter Buffer Objects (ACBOs) for keeping LOD selection count. We will reset these ACBOs to 0 before calling `glDispatchCompute` which runs culling and LOD calculations.

We'll create 2 new classes `SSBO` and `ACBO` similar to `VBO` class but with buffer binding target type of `GL_SHADER_STORAGE_BUFFER` and `GL_ATOMIC_COUNTER_BUFFER` instead of `GL_ARRAY_BUFFER`. We will create one SSBO which will contain all instance positions data and 3 empty SSBOs for LOD to hold memory in the GPU.
```cpp
// In SSBO.cpp
SSBO::SSBO(const std::vector<glm::vec3>& instanceData, std::string bufferType)
{
	glGenBuffers(1, &ID);
	glBindBuffer(GL_SHADER_STORAGE_BUFFER, ID);
	if(bufferType == "justForSpace")
		glBufferData(GL_SHADER_STORAGE_BUFFER, instanceData.size() * sizeof(glm::vec3), nullptr, GL_DYNAMIC_DRAW);
	else
		glBufferData(GL_SHADER_STORAGE_BUFFER, instanceData.size() * sizeof(glm::vec3), instanceData.data(), GL_STATIC_DRAW);
}

// In ACBO.cpp
ACBO::ACBO()
{
  glGenBuffers(1, &ID);
  glBindBuffer(GL_ATOMIC_COUNTER_BUFFER, ID);
  glBufferData(GL_ATOMIC_COUNTER_BUFFER, sizeof(GLuint), nullptr, GL_DYNAMIC_DRAW);
}

// In Mesh.cpp
void Mesh::initializeSSBOs(...) {
    instancePosBuffer = new SSBO(instancePositions, "normal");

    highPosBuffer = new SSBO(instancePositions, "justForSpace");
    ...

    highLodCounterBuffer = new ACBO();
    ...
}

Mesh::Mesh(...){
  ...
  initializeSSBOs(...);
  ...
}
```
Now we have initialized our SSBOs and ACBOs, we will define the compute shader with culling and LOD calculation.
In our ShaderClass, we will add new constructor, which can create a new shader program for compute shaders.
```cpp
Shader::Shader(const char* computeShaderFile, const char* shaderType)
{
	Shader::shaderType = shaderType;
	std::string computeCode = get_file_contents(computeShaderFile);
	const char* computeShaderSource = computeCode.c_str();
	
	GLuint computeShader = glCreateShader(GL_COMPUTE_SHADER); // Compute shader object
	glShaderSource(computeShader, 1, &computeShaderSource, NULL); // attaching src to object
	glCompileShader(computeShader);

	compileErrors(computeShader, "CULLING_LOD_COMPUTE");

	ID = glCreateProgram(); // shader program object
	glAttachShader(ID, computeShader);
	glLinkProgram(ID); // wrap-up/link all shaders together
	compileErrors(ID, "PROGRAM");

	glDeleteShader(computeShader);
}
```
Then we will create a new compute shader with name `cullingLOD.comp` with following code. I will add the required comments to make it understand but most of it is very easy to understand.
```GLSL
#version 430

layout(local_size_x = 256) in; // Number of threads per workgroup

// Input buffer: all instances before culling 
// its a standard format with compute shaders and SSBOs to define shader data variables with layout(std430, binding = x)
layout(std430, binding = 0) buffer InstancePositionsData {
    vec3 instancePositions[];
};
...
// High LOD buffers
layout(std430, binding = 3) buffer HighLodPositions {
    vec3 highPositions[];
};
...
// Atomic counter: 
layout(binding = 0) uniform atomic_uint highLodCount;
...

// Pass camera Positions, all frustum planes using uniform
// uniform for camera position
uniform vec3 cameraPosition;

// Uniforms for frustum bounds
uniform float left;
uniform float right;
uniform float bottom;
uniform float top;
uniform float near;
uniform float far;

void main(){
  uint index = gl_GlobalInvocationID.x; // it defines current instancePosition index

  vec3 position = instancePositions[index];

  // Orthographic frustum culling
  if (position.x >= left && position.x <= right && position.y >= bottom && position.y <= top && position.z <= near && position.z >= far) {
      
    highp float distanceSqrd = ((cameraPosition.x - position.x) * (cameraPosition.x - position.x))
        + ((cameraPosition.y - position.y) * (cameraPosition.y - position.y))
        + ((cameraPosition.z - position.z) * (cameraPosition.z - position.z)); // using squared distance to avoid sqrt()

    if (distanceSqrd <= 2500.0) { // High LOD
        uint visibleIndex = atomicCounterIncrement(highLodCount);
        highPositions[visibleIndex] = position;
        ...
    }else if(...){ // Medium LOD
      ...
    }else{ // Low LOD
      ...
    }
  }
}
```
We will create one shader program object with this compute shader in main.cpp.
```cpp
// In main.cpp
...
//--------GPU compute shader for LOD and Culling--------
Shader cullingLODShader("./cullingLOD.comp", "Compute Shader");
cullingLODShader.Activate();
...
```
Now we need to create a new function which should get called inside our rendering loop and that can do pre setup before calling this compute shader.
```cpp
// In Mesh.cpp
void Mesh::gpuCullingLOD(Shader& computeShader, Camera& camera)
{
  highLodCounterBuffer->Clean();
  ...

  highPosBuffer->Clean();
  ...

  computeShader.Activate();
  glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 0, instancePosBuffer->ID);
  ...

  glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 3, highPosBuffer->ID);
  ...

  // atomic counters
  glBindBufferBase(GL_ATOMIC_COUNTER_BUFFER, 0, highLodCounterBuffer);
  ...

  glUniform3f(glGetUniformLocation(computeShader.ID, "cameraPosition"), camera.Position.x, camera.Position.y, camera.Position.z);
  glUniform1f(glGetUniformLocation(computeShader.ID, "left"), camera.left);
  glUniform1f(glGetUniformLocation(computeShader.ID, "right"), camera.right);
  glUniform1f(glGetUniformLocation(computeShader.ID, "bottom"), camera.bottom);
  glUniform1f(glGetUniformLocation(computeShader.ID, "top"), camera.top);
  glUniform1f(glGetUniformLocation(computeShader.ID, "near"), camera.near);
  glUniform1f(glGetUniformLocation(computeShader.ID, "far"), camera.far);
  
  glDispatchCompute((instancePositions.size() + 255) / 256, 1, 1); // calling compute shader to run
  glMemoryBarrier(GL_SHADER_STORAGE_BARRIER_BIT | GL_ATOMIC_COUNTER_BARRIER_BIT); // to prevent data race issues and ensuring SSBO and ACBO visiblity for future operations
  glFinish(); // for synchronization and to ensure that compute shader has finished execution before proceeding
}
```
Now we are set and we have updated data with culling and LOD selection but how to pass it to vertex shader in our rendering pipeline. 
We can copy it to `highInstancePositions` and other vector arrays but this copying of large amount of data for each frame will cost us too much. Can we do something more optimal?

As we already have this calculated data stored in GPU, we should use it directly in our vertex shader. It will directly read it from same memory locations, we just have to tell where to read. But we also have to modify our vertex shader code a little. We will add separate SSBOs for instancePositions and others and then read current instancePosition with `gl_InstanceID`.
```GLSL
// In sphere.vert
...
layout(std430, binding = 1) buffer Positions {
    vec3 positions[];
};
layout(std430, binding = 2) buffer Colors {
    vec3 colors[];
};
...

void main(){
  vec3 instancePos = positions[gl_InstanceID];
	vec3 instanceColor = colors[gl_InstanceID];
  ...
}
```
Now inside our `Mesh::Draw` function, just fetch all lod counts from Atomic Count Buffers, bind respected SSBOs based on current LOD and call glDrawElementsInstanced() with current LOD count.
```cpp
void Mesh::Draw(){
  ...
  glBindBuffer(GL_ATOMIC_COUNTER_BUFFER, highLodCounterBuffer);
  glGetBufferSubData(GL_ATOMIC_COUNTER_BUFFER, 0, sizeof(GLuint), &highLodCount);

  glBindBuffer(GL_ATOMIC_COUNTER_BUFFER, medLodCounterBuffer);
  glGetBufferSubData(GL_ATOMIC_COUNTER_BUFFER, 0, sizeof(GLuint), &medLodCount);

  glBindBuffer(GL_ATOMIC_COUNTER_BUFFER, lowLodCounterBuffer);
  glGetBufferSubData(GL_ATOMIC_COUNTER_BUFFER, 0, sizeof(GLuint), &lowLodCount);
  ...
  if (highLodCount > 0) {
    highDetailVAO.Bind();

    glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 1, highPosBuffer->ID); // Bind high LOD positions
    
    glDrawElementsInstanced(GL_TRIANGLES, highDetailIndices.size(), GL_UNSIGNED_INT, 0, highLodCount);
  }
  ...
}
```
And we are done.

**You should get 60fps frame rate for our application with 1 Million spheres.**

I will continue this blog with other titles which will discuss PBR and IBL in detail and its integration in our application.