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

**One importing thing before we start, the language of all these posts will not be that much formal. I'm keeping it light and conversational. Also, I'm writing all these posts based on my own understanding and it might not match yours.**

I also want to mention that I learned and cleared my basics from [learnopengl.com](https://learnopengl.com/) and [FreeCodeCamp video on openGL](https://youtu.be/45MIykWJ-C4?si=uBAfGH4kaYC-CwCn).
I'll keep mentioning these resources at right places, so you can go and look through those topics in detail.

Let's start now.

## Basic setup
I used Visual Studio 22 on Windows 11 with NVIDIA RTX A1000 Laptop GPU and 12 GB RAM.
With this GPU, we can draw ~1.5-4 million visible vertices/frame after including complex calculations like PBR and IBL for maintaining 60 fps on screen. 
You just check your configurations and the limits of your GPU.

I hope you have decent experience in working with C++ and Object Oriented Programming Systems concepts. As, after covering the basics we'll refactor our code to follow OOPS concepts.

For OpenGL, we are not going to use the OpenGL functions directly as they are written by the GPU providers. But we will use GLFW, a library written in C, which provides us all basic necessities for rendering and to create OpenGL context.

You can build GLFW package using CMake GUI/CLI as described on [learnopengl.com's Building GLFW section](https://learnopengl.com/Getting-started/Creating-a-window#:~:text=most%20other%20IDEs.-,Building%20GLFW,-GLFW%20can%20be) or directly use 
the suitable package for your system. We'll also have to link this library with our compilation process. For Visual Studio setup, you can follow [this section from learnopengl.com](https://learnopengl.com/Getting-started/Creating-a-window#:~:text=first%20OpenGL%20application!-,Linking,-In%20order%20for).
```
If you ever want to compile your c++ code with external libraries, you can link them by adding -l with library name.

eg. 
g++ -o main main.cpp -lglfw3 -lGL 
```
We'll also need one more library **GLAD**, it helps in locating the openGL function implementation it needs and store them in separate function pointers for later use.
It is a system specific process and it's setup process you can find [here on learnopengl.com](https://learnopengl.com/Getting-started/Creating-a-window#:~:text=to%2Ddate%20library.-,Setting%20up%20GLAD,-GLAD%20is%20an).

## Creating a window
Let's create main.cpp file in you project, include glfw and glad and create main() function. We'll initialize glfw with glfwInit() and we'll have to configure it like setting up the openGL version (3.3) and openGL profile type(CORE). 
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
You can follow [this page](https://learnopengl.com/Getting-started/Hello-Window) for detailed description of the things I'm writing here. As the intented topic for this article is different, so I have to keep things short and precise.

After setting up glfw, we will create GLFWwindow object by passing the window's width, height and title. We also have to make this window context current on the calling thread.
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
Then we need to initialize *glad* in our application by calling 
```
if(!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress))
    {
        std::cout << "Failed to initialize GLAD" << std::endl;
        return -1;
    }
```
Before we start rendering, we need to set the viewport dimensions for openGL using **glViewport(x1, y1, x2, y2)** where (x1, y1) is the location of lower left corner point and (x2, y2) is upper right corner point.
If you want you can setup a callback, which will resize your viewport when you will resize the window.

We dont want our application to just render something and exit. We need it to be running until we close it. For that we will some event loop kinda thing and what's better than a infinite while loop. 
So, we'll create a while loop with condition (!glfwWindowShouldClose(window)) which will check if it's been instructed fot closing the window.
```
while(!glfwWindowShouldClose(window))
{
    glfwSwapBuffers(window);
    glfwPollEvents();    
}
```
* glfwSwapBuffers(window) - It will swap the color buffer (which is a 2d buffer consisting of color values for each pixel in this window, which combiningly makes a frame). Usually there are 2 buffers, we call them front and back buffer. The front buffer is what we see as output image while in back buffer, all the rendering commands draws it. As soon as the back buffer is ready, we swap the back buffer to front buffer and new frame drawing starts to the back buffer.
* glfwPollEvents() - It will listen if any event is triggered and call the corresponding functions.

After exiting the loop, we'll need to call **glfwTerminate()**, which will cleanup the resources. 

Upto this point, you should be able to create a blank black color window.

> If you want you can add a keypress event which will close the application by pressing ESC button. Just add the below line at the top of your while loop.
```
if(glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS) glfwSetWindowShouldClose(window, true);
```

