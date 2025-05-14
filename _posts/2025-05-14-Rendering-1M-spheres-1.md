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

Let's start now.

## Basic setup
I used Visual Studio 22 on Windows 11.