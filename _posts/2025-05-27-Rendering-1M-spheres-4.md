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
	for (size_t i = 0; i < instancePositions.size(); i++) {
		if (isInside(instancePositions[i], camera)) {
			{
				culledPositions.push_back(instancePositions[i]);
			}
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

