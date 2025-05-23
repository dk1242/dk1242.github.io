---
layout: post
title:  "Rendering 1 Million spheres: Part 3 (Circle, Sphere and Spheres)"
date:   2025-05-23
---
## Circle
Before we start for spheres, I want to show how can we draw a circle. The easiest way to draw a circle is to draw multiple triangles and arrange them in a circular manner, just like in the following image:

![alt text](/assets/images/circle_with_triangle.png)

To arrange these triangles in such a manner that it will form a circle, we need to find the vertex positions accordingly. We will use mathematical equations to find them. If we plot our 2D circle on a xy plane, any point on the perimeter of circle can be calculated using some trigonometry.

![alt text](/assets/images/circlevertex.png)
$$
x = r \cdot cos(θ) \\
y = r \cdot sin(θ)
$$
Now we know how to find all vertices positions in the circle, we just need to figure out how to arrange them in indices array. For that, just visualize we are drawing all triangles by ourselves, then how would you do it. I'll start from center of circle then move to 2nd vertex $(r.cos(θ), r.sin(θ))$ and in last $(r.cos(θ+Δθ), r.sin(θ+Δθ))$. We will repeat it for whole 360° while Δθ is the amount of angle in each segment. We can define number of segment as we want but just remember with high number of segments, the circle will be smoother but the number of vertices will also increase.
```cpp
vector<glm::vec3> circleVertices;
vector<GLuint> circleInd;

const int numOfSegments = 50;
const float radius = 0.5f;

circleVertices.push_back(glm::vec3(0.0f, 0.0f, 0.0f));
for (int i = 0; i <= numOfSegments; i++) {
	float angle = 2.0f * 3.14f * (i / numOfSegments); 
	float x = radius * cos(angle);
	float y = radius * sin(angle);
	circleVertices.push_back(glm::vec3(x, y, 0.0f));
}

for (int i = 1; i <= numOfSegments; ++i) {
	circleInd.push_back(0);        // Circle Center vertex
	circleInd.push_back(i);        // Current vertex
	circleInd.push_back(i + 1);    // Next vertex
}

circleInd.push_back(0);            // Center vertex
circleInd.push_back(numOfSegments); // Last vertex
circleInd.push_back(1);           // Closing the circle

Mesh circle(circleVertices, circleInd);
```
Before we start drawing we also need to create a simple Model matrix for the circle and assign it in shader uniform, so it can easily work with camera. 
```cpp
glm::vec3 circlePos = glm::vec3(0.0f, 0.0f, 0.0f);
glm::mat4 circleModel = glm::mat4(1.0f); // an Indentity matrix with no translations, no rotations and no scaling 
circleModel = glm::translate(circleModel, circlePos); // it will translate the model with circlePos, last column will contain circlePos (x, y, z), visulally it will shift the model to this circlePos
glUniformMatrix4fv(glGetUniformLocation(shaderProgram.ID, "model"), 1, GL_FALSE, glm::value_ptr(circleModel));
```
Now just call `circle.Draw(shader, camera)` inside rendering loop and run your application. It should render the circle like following image.

![alt text](/assets/images/circle.png)

## Sphere
Next we need to render a Sphere. We will follow the same steps as above. First we need to figure out how to calculate the vertices positions and then how to arrange them so the triangles will arrange themselves in a spherical position. 
Before we start, I want to clear that we are going to draw triangles over the surface only, not inside. It will be a hollow sphere with triangles attached over its surface.

We can calculate the vertices by visualizing a sphere in following way.

![alt text](/assets/images/sphereVertex.png)

In the above image, first assume there is a line between center and the vertex point. Now, $θ$ is the angle between y-axis and that virtual line. So, the y value for that vertex will be $r.cos(θ)$. Then, if we draw the $sin(θ)$ component of that line, it will fall on xz plane and $Φ$ will be the angle between this line and z-axis.
So, the z-value will be $r.sin(θ).cos(Φ)$ and the x-value will be $r.sin(θ).sin(Φ)$.
$$
x = r \cdot sin(θ) \cdot sin(Φ) \\
y = r \cdot cos(Φ)\\
z = r \cdot sin(θ) \cdot cos(Φ) \\
$$
This way we will define all our vertex points and to push them in indices array, just think of it as sphere grid and traverse it row(level) wise.
```cpp
vector<glm::vec3> sphereVertices;
vector<GLuint> sphereIndices;

const int latitudeSegments = 25;  // Number of vertical divisions
const int longitudeSegments = 25; // Number of horizontal divisions
const float radius = 2.5f;

for (int i = 0; i <= latitudeSegments; ++i) {
	float theta = i * 3.14159265359f / latitudeSegments; // Polar angle (0 to π), we only need to cover the half part because other half will be covered becuase of whole traversal in horizontal cut
	for (int j = 0; j <= longitudeSegments; ++j) {
		float phi = j * 2.0f * 3.14159265359f / longitudeSegments; // Azimuthal angle (0 to 2π)
		// Vertex position
		float x = (radius) * sin(theta) * sin(phi);
		float y = (radius) * cos(theta);
		float z = (radius) * sin(theta) * cos(phi);

		sphereVertices.push_back(glm::vec3(x, y, z));
	}
}
for (int i = 0; i < latitudeSegments; ++i) {
	for (int j = 0; j < longitudeSegments; ++j) {
		// Calculate indices for two triangles per grid cell
        // i defining the vertical level starting from top
        // j tells the horizontal level but it traverse it in a circluar way
		unsigned int first = (i * (longitudeSegments + 1)) + j; // index of the current vertex in the sphere grid
		unsigned int second = first + longitudeSegments + 1;    // index of the vertex directly below the current vertex

		sphereIndices.push_back(first);         // Current vertex
		sphereIndices.push_back(second);        // Vertex directly below
		sphereIndices.push_back(first + 1);     // Vertex in the right

		sphereIndices.push_back(second);        // Vertex directly below
		sphereIndices.push_back(second + 1);    // Vertex below and to the right
		sphereIndices.push_back(first + 1);     // Vertex in the right
	}
}

Mesh sphere(sphereVertices, sphereIndices);
```
Then set sphere model in shader and call Draw() inside the rendering loop. We dont need to change anything in the shader code.
```cpp
...
glm::vec3 spherePos = glm::vec3(0.0f, 0.0f, 0.0f);
glm::mat4 sphereModel = glm::mat4(1.0f);
sphereModel = glm::translate(sphereModel, spherePos);
glUniformMatrix4fv(glGetUniformLocation(shaderProgram.ID, "model"), 1, GL_FALSE, glm::value_ptr(sphereModel));
...
sphere.Draw(shaderProgram, camera);
```

Now, we are done with rendering a sphere but you may not be able to see it as sphere because we have not enabled normals and lighting yet. So, to visualize it enable `glPolygonMode` which will draw the triangle without filling them with colors.
```cpp
glPolygonMode(GL_FRONT_AND_BACK, GL_LINE);
```
![alt text](/assets/images/spherePolygons.png)

## Spheres
Now, we know how to draw a Sphere, we will try to scale it.