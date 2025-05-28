---
layout: post
title:  "Rendering 1 Million spheres: Part 4 (Frustum Culling & Level of Detail)"
date:   2025-05-27
---
In this blog, we will go through the 2 most important and standard optimizations methods including Frustum Culling and Level of Detail.

# Frustum Culling
Frustum culling is an optimization technique which can improve the rendering performance by excluding the objects outside the camera's viewing frustum from the rendering pipeline.
The camera's viewing frustum depends on the type of camera, we are using. As we are using a perspective camera, frustum should also be of perspective type. But there's one issue with perspective frustum here that when we move our camera, it will remove and include the spheres in between the movement which doesn't look good.

![Perspective frustum](/assets/images/perspective_frustum_working.png)

It'll be good if these spheres will get added and removed at the near and far plane only, which will produce a much better experience. We can produce this effect using **an orthographic type frustum while using a perpspective camera**. 

![Orthographic frustum](/assets/images/orthographic_frustum_working.png)

### Orthographic Frustum
Before we start doing frustum culling, we first have to calculate the frustum planes positions. As we already have defined the near and far plane distance for camera, our frustum depth will be (far - near). 

![Orthographic Frustum Calculation](/assets/images/orthographic_frustum_calculation.png)

We can calculate frustum Height by applying a little trigonometry. So, tan(fov/2) will be equal to half of frustum height divided by frustum depth. You can understand it through above image. 
In last, we can calculate frustum width using the aspect ratio we defined for our screen.
```cpp
void Camera::UpdateFrustum() {
    float aspectRatio = 16.0f/9.0f;
	float frustumDepth = 150.0f; 
    float frustumHeight = 2 * tan(glm::radians(45.0f / 2.0f)) * frustumDepth;
	float frustumWidth = aspectRatio * frustumHeight;

	this->left	  = Position.x - frustumWidth / 2.0f;
	this->right	  = Position.x + frustumWidth / 2.0f;

	this->bottom  = Position.y - frustumHeight / 2.0f;
	this->top	  = Position.y + frustumHeight / 2.0f;

	this->near	  = Position.z;
	this->far	  = Position.z - frustumDepth;
}
```
We will call `updateFrustum()` inside `updateMatrix()` after calculating the cameraMatrix in our rendering loop. 

After this we need to check if a sphere is lying inside or outside our frustum. For this, we will just check that our sphere center as a point is inside all 6 frustum planes or not.
```cpp
static bool isInside(const glm::vec3& position, Camera& camera) {
    return position.x >= camera.left && position.x <= camera.right &&
        position.y >= camera.bottom && position.y <= camera.top &&
        position.z <= camera.near && position.z >= camera.far;
}
```
We will also need one separate funtion which will call isInside() for all instancePositions and if inside, it will push to culledPositions vector. Then we will use this `culledPositions.size()` instead of `numInstances`. 
```cpp
void Mesh::doCulling(Camera& camera){
	for (size_t i = 0; i < instancePositions.size(); i++) 
	{
		if (isInside(instancePositions[i], camera)) 
		{
			culledPositions.push_back(instancePositions[i]);
		}
	}
}
...
void Mesh::Draw(...){
	...
	glDrawElementsInstanced(GL_TRIANGLES, indices.size(), GL_UNSIGNED_INT, 0, culledPositions.size());
}
```
Then we will call this `doCulling()` function before calling the `baseSphere.Draw(shaderProgram, camera, numInstances)` inside our rendering loop. 

Now, try to increase the numInstances value, start from at least 100k where screen is rendering with 60 fps. 
For me till 175k spheres, it's rendering with ~60fps

# Level of Detail (LOD)
Level of Detail is a technique we use to optimize the rendering performance by dynamically changing the complexity of our 3d Models based on their distance from the camera. So, the closer an object is to the viewer, higher the details of that model and vice versa.

In our application, we can implement this by dividing our frustum depth in 3 parts based on the distance from the camera. The first part which is nearest to our camera will have the spheres with the highest number of vertices, then the middle with medium number of vertices and the last section with the lowest number of vertices. 
Implementing LOD will improve the performance as there will be reduction in computations inside GPU (less number of vertices to run vertex and fragment shader code).
We will generate vertices in 3 different vectors for LOD and the corresponding indices too in the same way. Inside Mesh class we will need a different constructor which will assign 3 different VAOs for LOD.
```cpp
// main.cpp 
std::vector<glm::vec3> highDetailVertices;
std::vector<GLuint> highDetailIndices;

std::vector<glm::vec3> mediumDetailVertices;
std::vector<GLuint> mediumDetailIndices;

std::vector<glm::vec3> lowDetailVertices;
std::vector<GLuint> lowDetailIndices;

// control the segments values based on your system specs, for me the grpahic quality was good with following segment values
const int highDetailLatSegments = 30;
const int highDetailLonSegments = 30;

const int mediumDetailLatSegments = 25;
const int mediumDetailLonSegments = 25;

const int lowDetailLatSegments = 5; // we can drop it to 5 because they looks very small on the screen and don't need very high details
const int lowDetailLonSegments = 5;

for (int i = 0; i <= highDetailLatSegments; ++i) {
	float theta = i * 3.14159265359f / highDetailLatSegments;
	for (int j = 0; j <= highDetailLonSegments; ++j) {
		float phi = j * 2.0f * 3.14159265359f / highDetailLonSegments;
		
		float x = (radius)*sin(theta) * cos(phi);
		float y = (radius)*sin(theta) * sin(phi);
		float z = (radius)*cos(theta);
		
		highDetailVertices.push_back(glm::vec3(x, y, z));
	}
}
for (int i = 0; i < highDetailLatSegments; ++i) {
	for (int j = 0; j < highDetailLonSegments; ++j) {
		unsigned int first = (i * (highDetailLonSegments + 1)) + j;
		unsigned int second = first + highDetailLonSegments + 1;

		highDetailIndices.push_back(first);
		highDetailIndices.push_back(second);
		highDetailIndices.push_back(first + 1);

		highDetailIndices.push_back(second);
		highDetailIndices.push_back(second + 1);
		highDetailIndices.push_back(first + 1);
	}
}
...
// Similary do for the mediumDetailVertices and lowDetailVertices
...
Mesh baseSphere(highDetailVertices, highDetailIndices, mediumDetailVertices, mediumDetailIndices,
	lowDetailVertices, lowDetailIndices, instancePositions);
```
```cpp
Mesh::Mesh(std::vector<glm::vec3>& highVertices, std::vector<GLuint>& highIndices,
    std::vector<glm::vec3>& medVertices, std::vector<GLuint>& medIndices, 
    std::vector<glm::vec3>& lowVertices, std::vector<GLuint>& lowIndices,
    std::vector<glm::vec3>& instancePositions)
{
	...
	highDetailVAO.Bind();
	VBO highVBO(highVertices);
	EBO highEBO(highIndices);
	highDetailVAO.LinkAttrib(highVBO, 0, 3, GL_FLOAT, sizeof(glm::vec3), (void*)(0));
	highDetailVAO.Unbind();
	highVBO.Unbind();
	highVBO.Unbind();

	...
	// repeat for other medium and low 
	...
}
```
We also need 3 different instancePositions vector based on the LOD, which we will update after calculating the culledPositions value. For now, we have divided our frustum depth (which is 150 units) in 3 parts (50 units for each LOD).
```cpp
for (size_t i = 0; i < culledPositions.size(); i++) {
	float distance = glm::length(camera.Position - culledPositions[i]);

	if (distance < 50.0f) {
		highDetailInstancePositions.push_back(culledPositions[i]);
	}
	else if (distance < 100.0f) {
		mediumDetailInstancePositions.push_back(culledPositions[i]);
	}
	else {
		lowDetailInstancePositions.push_back(culledPositions[i]);
	}
}
```
Now, for drawing the spheres we need different LODs Instance Positions which we can pass through different draw calls. Here, we just have 3 LODs, so, with 3 draw calls instead of 1, it will not cause a significant drop in our rendering performance.
Inside our Draw function, we will create 3 instance VBO for 3 different LODs and call the `glDrawElementsInstanced` with the current LOD instancePositions size.
```cpp
void Mesh::Draw(...){
	...
	if (!highDetailInstancePositions.empty()) {
		highDetailVAO.Bind();
		InstanceVBO highPosVBO(highDetailInstancePositions);
		highDetailVAO.LinkAttrib(highPosVBO, 1, 3, GL_FLOAT, sizeof(glm::vec3), (void*)(0));
		
		glDrawElementsInstanced(GL_TRIANGLES, highDetailIndices.size(), GL_UNSIGNED_INT, 0, highDetailInstancePositions.size());

		highPosVBO.Unbind();
		highPosVBO.Delete();
	}

	if (!mediumDetailInstancePositions.empty()) {
		mediumDetailVAO.Bind();
		InstanceVBO medPosVBO(mediumDetailInstancePositions);
		mediumDetailVAO.LinkAttrib(medPosVBO, 1, 3, GL_FLOAT, sizeof(glm::vec3), (void*)(0));
		
		glDrawElementsInstanced(GL_TRIANGLES, mediumDetailIndices.size(), GL_UNSIGNED_INT, 0, mediumDetailInstancePositions.size());
		medPosVBO.Unbind();
		medPosVBO.Delete();
	}

	if (!lowDetailInstancePositions.empty()) {
		lowDetailVAO.Bind();
		InstanceVBO lowPosVBO(lowDetailInstancePositions);
		lowDetailVAO.LinkAttrib(lowPosVBO, 1, 3, GL_FLOAT, sizeof(glm::vec3), (void*)(0));
		
		glDrawElementsInstanced(GL_TRIANGLES, lowDetailIndices.size(), GL_UNSIGNED_INT, 0, lowDetailInstancePositions.size());
		lowPosVBO.Unbind();
		lowPosVBO.Delete();
	}
}
```
Now try to increase the `numInstances` value and check the screen refresh rate. For me till 200k, it gave me ~60fps.

So, we've reached 200k with 60fps but still 1 Million looks a long distance to go. 

