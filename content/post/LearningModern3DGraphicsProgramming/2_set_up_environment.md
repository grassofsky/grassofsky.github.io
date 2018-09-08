+++
date="2017-05-30T12:00:00+06:00" 
title="2. 环境设置"
categories=["Translation of books"] 
tags=["opengl", "LearningModern3DGraphicsProgramming"]
+++

# 环境设置

由于本书中的例子，均是基于OpenGL实现的，因此你的工作环境需要能够运行OpenGL，为了读者能够更好的运行原文中的示例，此处简单地介绍了linux和windows下OpenGL环境的配置。需要配置的是除了OpenGL基础环境外，还需要freeglut和glew。具体的配置见下面的内容。

## linux

由于译者使用的linux版本为mint 18 sarah，此处就以mint系统为例进行linux下的环境配置。

~~~
sudo apt-get install build-essential libgl1-mesa-dev git libglu1-mesa-dev
sudo apt-get install libglew-dev freeglut3-dev

// 使用glxinfo查看OpenGL支持的版本，如下所示
~$ glxinfo | grep OpenGL
OpenGL vendor string: NVIDIA Corporation
OpenGL renderer string: GeForce 940MX/PCIe/SSE2
OpenGL core profile version string: 4.5.0 NVIDIA 367.57
OpenGL core profile shading language version string: 4.50 NVIDIA
OpenGL core profile context flags: (none)
OpenGL core profile profile mask: core profile
OpenGL core profile extensions:
OpenGL version string: 4.5.0 NVIDIA 367.57
OpenGL shading language version string: 4.50 NVIDIA
OpenGL context flags: (none)
OpenGL profile mask: (none)
OpenGL extensions:
OpenGL ES profile version string: OpenGL ES 3.2 NVIDIA 367.57
OpenGL ES profile shading language version string: OpenGL ES GLSL ES 3.20
OpenGL ES profile extensions:
~~~

也可以参考cnblogs他人的博文：[Linux下OpenGL开发--准备篇](http://www.cnblogs.com/antistone/archive/2008/03/18/1111403.html)

## windows
 
可以参考cnblogs他人的博文：[搭建OpenGL环境-Windows/VS2013](http://www.cnblogs.com/lhyz/p/4178004.html)


