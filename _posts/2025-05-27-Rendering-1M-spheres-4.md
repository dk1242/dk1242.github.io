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

<!-- ![Orthographic Frustum Calculation](/assets/images/orthographic_frustum_calculation.png) -->

We can define frustum Width and depth based on our scene. The minimum value we should assign for frustum depth is 150.0f because our spheres are spread from -150 to +150 in z direction. For frustum width too, we should assign it atleast 150.0f.
We will calculate frustum height using the aspect ratio we have defined for our screen.
```cpp
void Camera::UpdateFrustum() {
    float frustumWidth = 180.0f;
	float frustumHeight = 9.0f / 16.0f * frustumWidth; 
	float frustumDepth = 200.0f; 

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
	for (size_t i = 0; i < instancePositions.size(); i++){
		if (isInside(instancePositions[i], camera)) {
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

### Instance VBOs restructuring
There is one more simple optimization we can do that instead of creating Instance VBO with every draw call, we can create it once with *initializeInstanceVBOs()*, which we will call just before entering the rendring loop. Then update these instance VBOs in each draw call. 
```cpp
// In Mesh.h
InstanceVBO *highPosVBO, *medPosVBO...;

// In Mesh.cpp
void Mesh::initializeInstanceVBOs(Camera &camera) {
    for (int i = 0; i < instancePositions.size(); ++i) {
        // Perform frustum culling
        if (isInside(instancePositions[i], camera)) {
            float distanceSquared = glm::distance2(camera.Position, instancePositions[i]);

            // LOD selection based on distance
            if (distanceSquared < 2500.0f) { // High detail
                highDetailInstancePositions.push_back(instancePositions[i]);
            }
            else if (distanceSquared < 10000.0f) { // Medium detail
                mediumDetailInstancePositions.push_back(instancePositions[i]);
            }
            else { // Low detail
                lowDetailInstancePositions.push_back(instancePositions[i]);
            }
        }
    }
    highPosVBO = new InstanceVBO(highDetailInstancePositions);
    medPosVBO = new InstanceVBO(mediumDetailInstancePositions);
    ...
}
// Inside Mesh::Draw() function
void Mesh::Draw(){
	...
	if (!highDetailInstancePositions.empty()) {
		highDetailVAO.Bind();
		highPosVBO->Update(highDetailInstancePositions);

		highDetailVAO.LinkAttrib(*highPosVBO, 1, 3, GL_FLOAT, sizeof(glm::vec3), (void*)(0));
		
		glDrawElementsInstanced(GL_TRIANGLES, highDetailIndices.size(), GL_UNSIGNED_INT, 0, highDetailInstancePositions.size());
	}
	...
}
```
We also need to add one Update function in our **InstanceVBO** class.
```cpp
// Inside InstanceVBO.cpp
void InstanceVBO::Update(std::vector<glm::vec3>& instanceProps)
{
	glBindBuffer(GL_ARRAY_BUFFER, ID);
	glBufferSubData(GL_ARRAY_BUFFER, 0, instanceProps.size() * sizeof(glm::vec3), instanceProps.data());
}
```
Now, if you have observed we have combined Frustum culling and LOD selection in a single loop, which also have given some performance improvement because we removed the culledPosition.push_back(pos) which can save some time for us.

Till this point, I got 250k~260k spheres rendering easily with 60fps.

## Parallel threads calculation
The next approach we can directly implement is to use threads and do these calculations parallely. However, while it sounds straightforward and simple but it is not as it seems. One of the major challenge is thread synchronization, which can become a significant bottleneck in performance when using threads. Synchronization introduces overhead, which can significantly impact performance. As the number of threads increases, the amount of synchronization required also grows. After a point, adding more threads does not improve performance and may even degrade it due to the increased complexity of coordination.

To address this, we need to limit the number of threads which can run parallely and optimize our performance. In my case, after experimenting with different thread counts, I found that setting the number of threads to 4 resulted in improved performance. Other thread counts, whether higher or lower, led to a decline in performance due to either underutilization or excessive synchronization.

I used "<omp.h>" for threads implementation as it was easy to implement without caring for mutex lock and all. I also used private thread local buffers for push_back() and later we combined it with actual buffers to avoid any dirty write. 
```cpp
void updateCullingLOD(Camera& camera){
	omp_set_num_threads(4); // thread count 4 gave best results to me
#pragma omp parallel
    {
        // Private thread-local buffers
        std::vector<glm::vec3> threadHighDetailPositions, threadMediumDetailPositions, threadLowDetailPositions;

#pragma omp for schedule(static)
        for (int i = 0; i < instancePositions.size(); ++i) {
			if (isInside(instancePositions[i], camera)) {
				// LOD selection implementation
				// insert in threadHighDetailPositions...
			}
		}
#pragma omp critical
		{
			highDetailInstancePositions.insert(highDetailInstancePositions.end(), threadHighDetailPositions.begin(), threadHighDetailPositions.end());
			...
		}
	}
}
```
Again we didn't got the expected amount of performance, just an addition of 50k spheres. At this point, I got 60fps for around 300k spheres.

Now I understood the main bottleneck in the cpu side was the updateCullingLOD() where we were doing the 1 Million calculation and handling all those push_back() and everything in one go. Even though it was done parallely by 4 threads but not much improvement in the performance. So, we had to reduce this number for a single call.

### Calculation in chunks (Not a perfect solution but visually no difference)
I tried doing this calculation in chunks like we divide our data in 10 chunks and in each rendering loop we will run updateCullingLOD for 100k and the starting position of loop will offset everytime. When we will complete all chunks calculations, then only we will update the main vectors: highInstancePositions, medInstancePositions and lowInstancePositions. We can implement this by defining the start and end point in our for-loop and also keeping separate staging vectors and then aggregate them when all chunks calculation gets completed.
```cpp
// In Mesh.h
const size_t numChunks = 10; // we will try to reduce it
size_t currentChunk = 0;
size_t chunkSize = 1;

// In Mesh.cpp
Mesh::Mesh(...){
	...
	chunkSize = instancePositions.size() / numChunks;
	...
}
void Mesh::UpdateChunkLODs(Camera& camera){
	size_t start = currentChunk * chunkSize;
	size_t end = std::min(start + chunkSize, instancePositions.size());
	...
	for (int i = start; i < end; ++i) {
    // Perform frustum culling
    if (isInside(instancePositions[i], camera)) {
	...
	}
	highDetailStagingPositions.insert(highDetailStagingPositions.end(), threadHighDetailPositions.begin(), threadHighDetailPositions.end());
    ...
	currentChunk = (currentChunk + 1) % numChunks; 

	// Aggregate staging buffers into main buffers after all chunks processing
	if (currentChunk == 0) {
		this->AggregateBuffers();
	}
}
void Mesh::AggregateBuffers()
{
    highDetailInstancePositions = std::move(highDetailStagingPositions);
    mediumDetailInstancePositions = std::move(mediumDetailStagingPositions);
    ...

    // Clean staging buffers for the next cycle
    highDetailStagingPositions.resize(0);
    mediumDetailStagingPositions.resize(0);
    ...
}
```
Here we got **60 fps for 1000000 spheres** and I can reduce numChunks to 5 too and it's still giving same performance. So, here we are processing 200k spheres in a go and after every 5 cyles we are also updating our data, which is really giving very good visual result and you can never identify that you are seeing **a little old data for some fractions of seconds**. 

Even though it doesn't produce any wrong visual output, we still have to fix it. After doing some research, I found we can do this Culling and LOD selection calculation in GPU which will give a major performance boost becuase GPU can do these calculation parallely with their thousand of cores designed specifically for parallel processing. **GPUs also have higher memory bandwidth so we can directly store data in GPU and process them too very fast**.

I will complete the implementation of **Culling and LOD selection in GPU** in our next blog.