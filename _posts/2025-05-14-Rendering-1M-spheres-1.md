---
layout: post
title:  "Rendering 1 Million spheres: Part 1 (OpenGL basics)"
date:   2025-05-13
---
Hey everyone

So, I came across one problem statement on Twitter posted by Vipul Vaibhav and tried to implement it. It goes like this
>**Physically Based Rendering 1,000,000 spheres with unique materials supporting dynamic lighting and have sphere selection/highlighting ability while maintaining 60 FPS.**

I didn't have much experience with openGL before. I had written some GLSL shaders earlier, without interacting with openGL directly. So, I took my time to learn openGL and other important concepts like Physically Based Rendering, Image Based Lighting and basic matrix operations.

In this post, I'll write about the basics of openGL and almost everything we need to know before implementing the main solution (obviously I'll cover PBR and IBL in separate post).

**One important thing before we start, the language of all these posts will not be that much formal. I'm keeping it light and conversational. Also, I'm writing all these posts based on my own understanding and it might not match yours.**

I also want to mention that I learned and cleared my basics from [learnopengl.com](https://learnopengl.com/) and [FreeCodeCamp video on openGL](https://youtu.be/45MIykWJ-C4?si=uBAfGH4kaYC-CwCn).
I'll keep mentioning these resources at right places, so you can go and look through those topics in detail.

Let's start now.

## Basic setup
I used Visual Studio 2022 on Windows 11, with an NVIDIA RTX A1000 Laptop GPU and 12 GB RAM.
With this GPU, we can render approximately 1.5-4 million visible vertices/frame even with complex calculations like Physically Based Rendering (PBR) and Image-Based Lighting (IBL), whilw maintaining 60 fps on screen. 
You just check your configurations and the limits of your GPU.

I assume that you have a decent understanding of C++ and Object Oriented Programming concepts, since once we cover the basics, we'll refactor our code to follow OOP principles.

For OpenGL, we are not going to use the OpenGL functions directly as they are written by the GPU providers. Instead we will use GLFW, a library written in C, which provides all basic necessities for rendering and to create OpenGL context and handling windowing and input.

You can build GLFW package using CMake GUI or CLI as described on learnopengl.com's [*Building GLFW* section](https://learnopengl.com/Getting-started/Creating-a-window#:~:text=most%20other%20IDEs.-,Building%20GLFW,-GLFW%20can%20be) or directly use 
a pre-compiled package for your system. We'll also have to link this library with our compilation process. For Visual Studio setup, you can follow [this section from learnopengl.com](https://learnopengl.com/Getting-started/Creating-a-window#:~:text=first%20OpenGL%20application!-,Linking,-In%20order%20for).
```
If you ever want to compile your c++ code with external libraries, you can link them by adding -l<libraryname> during compilation.

eg. 
g++ -o main main.cpp -lglfw3 -lGL 
```
We'll also need one more library: **GLAD**, it helps in loading the actual openGL function implementations it needs and store them in separate function pointers for later use. This is a system specific process, and you can follow it's setup process from [here on learnopengl.com](https://learnopengl.com/Getting-started/Creating-a-window#:~:text=to%2Ddate%20library.-,Setting%20up%20GLAD,-GLAD%20is%20an).

## Creating a window
Let’s start by creating main.cpp file. Inside it, we'll include both GLFW and GLAD, and define main() function. We'll initialize glfw with glfwInit(). Then we'll have to configure it by specifying the openGL version (3.3) and setting openGL profile type(CORE). 
```
int main()
{
    glfwInit();
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
  
    return 0;
}
```
You can follow [this page](https://learnopengl.com/Getting-started/Hello-Window) for a detailed description of the things I'm writing here. SInce the focus of this article is different, I'll have to keep things short and precise.

After setting up glfw, we will create GLFWwindow* object by passing the window's width, height and title. We'll also have to make this window's context current on the calling thread using glfwMakeContextCurrent().
Before that we'll create 2 global const variable for width and height.
```
const int WIDTH = 1920, HEIGHT = 1080;
```
```
GLFWwindow* window = glfwCreateWindow(WIDTH, HEIGHT, "1M spheres", NULL, NULL);
if (window == NULL)
{
    std::cout << "Failed to create GLFW window" << std::endl;
    glfwTerminate();
    return -1;
}
glfwMakeContextCurrent(window);
```
Then we need to initialize **GLAD** in our application by calling 
```
if(!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress))
{
    std::cout << "Failed to initialize GLAD" << std::endl;
    return -1;
}
```
Before we start rendering, we need to set the viewport dimensions for openGL using **glViewport(x, y, width, height)** where (x, y) is the position of the lower-left corner of the viewport, and width and height define its size.

If you want you can also setup a callback, which will resize your viewport whenever the window is resized.

Next, we don't want our application to just render something and exit. We want it to keep running until we close it. For that we can use some event loop kinda thing and what's better than a infinite while loop? 
So, we'll create a while loop with condition (!glfwWindowShouldClose(window)) which will check if it's been instructed for closing the window.
```
while(!glfwWindowShouldClose(window))
{
    glfwSwapBuffers(window);
    glfwPollEvents();    
}
```
* glfwSwapBuffers(window) - This function swaps the color buffer (which is a 2d buffer consisting of color values for each pixel in this window, which together forms a frame). Typically, OpenGL uses double buffering, meaning there are two buffers, we call them front and back buffer. The front buffer is what we see as output image currently displaying on the screen and the back buffer performs rendering. As soon as the back buffer is ready, it's swapped with the front buffer, and the next frame starts drawing on the back buffer again.
* glfwPollEvents() - This function checks if any event is triggered(key press, mouse movement), and call the corresponding functions.

After exiting the loop, we'll call **glfwTerminate()**, which will cleanup the resources. 

Upto this point, you should be able to launch a window - a blank one with black background.

> If you want you can also add a keypress event to close the application by pressing ESC button. Just add the below line at the top of your while loop.
```
if(glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS) glfwSetWindowShouldClose(window, true);
```

For the sake of doing it, let's clear our screen with a specific color. 
```
glClearColor(0.2f, 0.3f, 0.3f, 1.0f); // fills this rgba on screen
glClear(GL_COLOR_BUFFER_BIT); // clears the screen color's buffer and then entire color buffer will be filled again with the same color configured by glClearColor.
```
## Triangles
