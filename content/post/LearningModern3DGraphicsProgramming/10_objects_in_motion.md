+++
date="2017-07-06T12:00:00+06:00" 
title="10. 对象的运动"
categories=["Translation of books"] 
draft=true
tags=["opengl", "LearningModern3DGraphicsProgramming"]
+++

**原文**: [https://paroj.github.io/gltut/](https://paroj.github.io/gltut/) 

# 第六章 对象的运动

在这一章中，我们会看到很多的对对象进行变换的方式，包括在不同的位置对物体进行渲染，和对物体进行旋转。以及给出为什么使用矩阵的方式实现这些变换。

## 空间

这本书中遍布了不同的坐标系空间。我们已经见到了基于opengl的坐标系，如标准化坐标系，裁剪坐标系，和窗口坐标系。同时我们也见到了用户定义的空间坐标系如相机坐标系。但是我们还没有讨论，空间坐标系到底是什么玩意儿。

空间坐标系包含以下特性：

- 维度，可以是二维，三维，四维或更高维度。
- 用于定义空间中向量的基向量，即坐标轴。这些基向量并不需要互相正交。
- 空间中的原点。空间中所有点的起源。
- 空间的范围。

空间中的任意点都可以用基向量与常数乘积后的和来表示。几何的表示如下图：

![](/image/figure6-1-two-2d-coordinate-systems.png)

上图中的两个坐标系都有各自的基向量，分别用红色和绿色表示。点在两个坐标系中的表示均为（2,2）。

坐标系空间的数学表达式如下：

**Equation 6.1 Coordinate System**

![](/image/equation6-1-coordinate-system.png)

几何版本通过基向量和原点已经足够处理问题了。原点仅仅是一个位置，基向量通过绘制的方式表现出来的。但是，在数值表示中，这些具体的数据表示什么实际的意思呢？一个点是一个坐标。这说明，坐标的位置是相对于坐标系统而而言的。对于基向量也是一样的。从本质上来讲，任何东西都是相对的。

### 变换

在先前章节中处理透视投影的时候，我们已经知道如何将一个坐标系空间中的点转换到另一个坐标系空间中，比如，将相机坐标系空间中的对象转换到裁剪坐标系中。将一个空间中的坐标变换为另一个空间中的坐标的行为被称为transformation，转换。坐标的实际意义是没有发生变化的，所有改变的仅仅是坐标系，以及相对于这个坐标系的坐标。

我们已经见了很多坐标系统的变换。opengl实现了从裁剪坐标系到标准化坐标系的变换，以及标准化坐标系到窗口坐标系的变换。我们的着色器实现了相机坐标系到裁剪坐标系的变换，这种变化的实现是通过矩阵实现的。透视投影仅仅是一种特殊的变换。

这一章会介绍到不同的变换操作，以及如何通过opengl进行实现。

### 模型空间

在这之前，我们首先定义一个新的坐标系：模型坐标系。这是用户定义的空间，但是和相机空间不同的是，模型空间并没有唯一的定义。它是每一个独立对象所特有的。通过缓存对象传递给定点着色器的顶点事实上是在模型空间中的。

模型空间有无数的变形。每一个需要渲染的对象通常有自己的模型空间，即使这些坐标系的不同之处仅仅是原点的不同。对象的模型空间通常是让编程者和模型设计者更方便的使用。

本章讨论的是模型空间到相机空间的变换操作。我们的着色器已经知道了如何处理相机空间中的数据，需要的也仅仅是通过模型空间到相机空间的变换。

## 平移

最简单的空间变换操作是平移。在前面的章节中，我们已经使用了平移变换。下面的代码就是在顶点着色器中出现的代码：

```
vec4 cameraPos = position + vec4(offset.x, offset.y, 0.0, 0.0);
```

此处的平移量，就是初始化空间中的原点想对于目标空间中点的偏移量。由于空间中的坐标都是相对于原点而言的。平移需要做的就是添加一个向量。这个向量是当前坐标系中的原点相对于目标坐标系的向量。

**Figure 6.2 Coordinate System Translation in 2D**

![](/image/figure6-2-coordinate-system-translation-in-2d.png)

平移是一个非常简单的操作，我们没有必要把它想得非常复杂。对于平移的实现，最好的工具就是：矩阵。当然，我们可以简单的使用一个三维向量传输偏移量的方式来实现平移。但是矩阵隐含了其他的一些优势。

我们将会遇到的所有的位置向量都是四维向量，最后一个维度的w值，一直是1.0，在透视投影中，我们已经见识了使用四个维度的好处。

如果我们希望对点没有任何变换，那么此时可以使用单位矩阵进行实现。单位矩阵如下：

**Equation 6.2 Identity Matrix**

$$\begin{Bmatrix} 1 & 0 & 0 & 0 \\ 0 & 1 & 0 & 0 \\ 0 & 0 & 1 & 0 \\ 0 & 0 & 0 & 1 \end{Bmatrix}$$

将上面的单位矩阵转换为平移矩阵只需要在矩阵的第四列设定对应的偏移量即可。

**Equation 6.3 Translation Matrix**

$$Translation = \begin{Bmatrix} 1 & 0 & 0 & x \\ 0 & 1 & 0 & y \\ 0 & 0 & 1 & z \\ 0 & 0 & 0 & 1 \end$$

这一章的例子开始，将使用glm实现矩阵的相关运算。

**Example 6.1 Translation Shader Initialization**

```
void InitializeProgram()
{
    std::vector<GLuint> shaderList;
    
    shaderList.push_back(Framework::LoadShader(GL_VERTEX_SHADER,
        "PosColorLocalTransform.vert"));
    shaderList.push_back(Framework::LoadShader(GL_FRAGMENT_SHADER,
        "ColorPassthrough.frag"));
    
    theProgram = Framework::CreateProgram(shaderList);
    
    positionAttrib = glGetAttribLocation(theProgram, "position");
    colorAttrib = glGetAttribLocation(theProgram, "color");
    
    modelToCameraMatrixUnif = glGetUniformLocation(theProgram,
        "modelToCameraMatrix");
    cameraToClipMatrixUnif = glGetUniformLocation(theProgram,
        "cameraToClipMatrix");
    
    float fzNear = 1.0f; float fzFar = 45.0f;
    
    cameraToClipMatrix[0].x = fFrustumScale;
    cameraToClipMatrix[1].y = fFrustumScale;
    cameraToClipMatrix[2].z = (fzFar + fzNear) / (fzNear - fzFar);
    cameraToClipMatrix[2].w = -1.0f;
    cameraToClipMatrix[3].z = (2 * fzFar * fzNear) / (fzNear - fzFar);
    
    glUseProgram(theProgram);
    glUniformMatrix4fv(cameraToClipMatrixUnif, 1, GL_FALSE,
        glm::value_ptr(cameraToClipMatrix));
    glUseProgram(0);
}
```

GLM是一个针对向量和矩阵运算的数学库。它的使用方式和glsl中向量和矩阵的运算方式类似。

矩阵`cameraToClipMatrix`定义成glm::mat4的类型。与glsl中类似都是基于列排序的。glm::value_ptr用来获取矩阵的指针。这个在将数据传递给着色器的时候是有用的。

此处fFrustumScale是通过计算得到的。

**Example 6.2 Frustum Scale Computation**

```
float CalcFrustumScale(float fFovDeg)
{
    const float degToRad = 3.14159f * 2.0f / 360.0f;
    float fFovRad = fFovDeg *  degToRad;
    return 1.0f / tan(fFovRad/2.0f);
}
const float fFrustumScale = CalcFrustumScale(45.0f);
```

## 缩放

另一个变换为缩放，即坐标系中的基向量变长或者变短。

**Figure 6.4 Coordinate System Scaling in 2D**

![](/image/figure6-4-coordinate-system-scaling-in-2d.png)

不同的基向量有相同的缩放比则被称为均一缩放，反之被成为非均一的。缩放用矩阵的形式表现如下：

**Equation 6.4 Scaling Transformation Matrix**

![](/image/equation6-4-scaling-transformation-matrix.png)

本例子中实现缩放矩阵计算的函数如下：

```
glm::mat4 ConstructMatrix(float fElapsedTime)
{
    glm::vec3 theScale = CalcScale(fElapsedTime);
    glm::mat4 theMat(1.0f);
    theMat[0].x = theScale.x;
    theMat[1].y = theScale.y;
    theMat[2].z = theScale.z;
    theMat[2] = glm::vec4(offset, 1.0f);

    return theMat;
}
```

在此处你也许会好奇，缩放系数可不可以是负数？是可以的，当是负数的时候物体原来的方向就会发生变化，可以在例子中，将缩放系数更改为负数，进行查看。

## 旋转

旋转变化的结果就是将一个物体在一个空间中的方位变换到了另一个空间中的方位。可以想象，在旋转的过程中，空间的坐标系随着物体一起旋转，那么物体相对于坐标系的位置是没有发生变化的，变化的仅仅是空间坐标系的方位。另一中理解方式是，空间没有随着变换，仅仅是物体在该空间中的方位发生了变换。

旋转看上去类似这个样子的：

**Figure 6.6 Coordinate Rotation in 2D**

![](/image/figure6-6-coordinate-rotation-in-2d.png)

一个点通过变换矩阵的变换计算如下：

**Equation 6.5 Vectorized Matrix Multiplication**

![](/image/equation6-5-vectorized-matrix-multiplication.png)

上述的公式中可将，x，y，z三个值乘的三个向量正是空间的三个轴的向量。w乘的向量可以作为一个偏移量。

一个空间到另一个空间的变换最终的含义是：将原始空间坐标系中的基向量和原点，在另一个目标坐标系空间中的重新表达。

因此，旋转矩阵并不单单意味着旋转，更深层次的理解应该是方位矩阵。这一点的掌握对之后的复杂变换的理解非常有用。下面给出了针对不同轴的旋转矩阵。

**Equation 6.6 Axial Rotation Matrices**

![](/image/equation6-6-axial-rotation-matrices.png)

更加通用的旋转矩阵，针对特定的旋转轴进行特定角度的旋转，如下：

**Equation 6.7 Angle/Axis Rotation Matrix**

![](/image/equation6-7-angle-axis-rotation-matrix.png)

旋转矩阵的构建如下：

**Example 6.4 Rotation Transformation Building**

```
glm::mat4 ConstructMatrix(float fElapsedTime)
{
    const glm::mat3 &rotMatrix = CalcRotation(fElapsedTime);
    glm::mat4 theMat(rotMatrix);
    theMat[3] = glm::vec4(offset, 1.0f);
    return theMat;
}
```

显示结果为：

**Figure 6.7 Rotation Project**

![](figure6-7-rotation-project.png)

## 矩阵的乐趣

在所有先前的变换的例子中都有平移变换。讲到缩放变换的时候并不是单纯的缩放变换，而是包含了平移。基本的变换均是由三种基本变换组合而成的。

任何变换操作的组合都是可能的，只要根据需求进行组合即可。

连续的变换可以通过连续的乘法实现。举个例子，S是一个纯粹的缩放矩阵，T是纯粹的平移矩阵，R是纯粹的旋转矩阵。那么着色器中的组合变换为：

```
vec4 temp;
temp = T * position;
temp = R * temp;
temp = S * temp;
gl_Position = cameraToClipMatrix * temp;
```

上述的代码在功能上实现了变换的组合，但并不是非常的合理，在着色器中对变换进行组合，计算速度并不是很快，对于每个顶点都会有四个向量与矩阵之间的乘法。

矩阵乘法是不可交换的，S×R与R×S是不相等的。但是，矩阵乘法是可以组合的，`(S*R)*T=S*(R*T)`。 但是如果让着色器去计算矩阵与矩阵之间的乘法，还不如保持上述的计算方式，同时又考虑到不同的顶点的变换矩阵是相同的，因此，可以现将所有的变换矩阵相乘，计算得到一个总的变换矩阵，然后再传递给着色器。

这里是使用矩阵进行变换运算的一个主要的原因。你可以将任意多个的变换进行组合，然后再传递给着色器，让opengl进行顶点变换的运算。

### 变换顺序

正如前面提到的，矩阵乘法是没有交换性的。如下：

**Equation 6.8 Order of Transformation**

![](/image/equation6-8-order-of-transformation.png)

如果用图来进行说明，则如下：

**Figure 6.8 Transform Order Diagram**

![](/image/figure6-8-transform-order-diagram.png)

你必须理解一些发生在S和T之间特殊的事情。在平移后再进行缩放，其实已经不是针对模型空间的缩放，而是针对平移后空间的缩放，要知道，每一个变换矩阵可以理解成一个对应的变换空间。

对于旋转矩阵而言也有类似的现象。旋转总是针对当前空间的原点进行的，因此比较合理的顺序是，旋转在平移之前，而缩放通常又在旋转之前。如果，将顺序反过来，那么缩放不再是针对模型空间进行的了。对于均匀的缩放，也许看不出什么问题，但是对于非均匀的缩放，就能够很明显的看出两种变换的不同了，正如上图中所示。

### 层级模型

在更加复杂的场景中，通常会需要一个相对于另一个模型空间变换的变换。这在想要让一个对象B捡起另一个对象A的时候会变得有用。被捡起的对象会随着捡它的对象的移动而移动。由多个渲染对象的多次变换组成的模型为层级模型。在这样的层级模型中，任何一个成员的变换都是一系列父级成员的变换，加上自身模型变换的结果。在这样的变换中，模型与其他模型之间存在父子关系。

为了方便讨论，层级模型中的每个组成的完整变换被称为结点（node）。每个结点都是有一些列的特殊变换组成的，基本都包含了缩放，平移，旋转等。有一点需要注意的是此处的完整变换是先对于其父亲的空间而言的，而不是相机空间。

因此，如果一个结点的变换是（3,0,4），表示在父空间的基础上，进行了这么多的平移。

接下来的例子给出了手臂的模型。这个例子是交互的。不同node之间的角度是可以通过键盘调整的。对应的命令如下：

**Table 6.1 Hierarchy Tutorial Key Commands**

![](/image/table6-1-hierarchy-tutorial-key-commands.png)

**Figure 6.9 Hierarchy Project**

![](/image/figure6-9-hierarchy-project.png)

这个例子的结构是很有意思的，同时还给出了一些处理层级模型的重要的数据结构。

Hierarchy类存储了我们节点层级信息，它存储了没一个结点的相对位置，以及角度信息，大小信息，每个立方体的大小。

绘制函数如下：

**Eample 6.5 Hierarchy::Draw**

```
void Draw()
{
    MatrixStack modelToCameraStack;

    glUseProgram(theProgram);
    glBindVertexArray(vao);

    modelToCameraStack.Translate(posBase);
    modelToCameraStack.RotateY(angBase);

    // Draw Left base.
    {
        modelToCameraStack.Push();
        modelToCameraStack.Translate(posBaseLeft);
        modelToCameraStack.Scale(glm::vec3(1.0f, 1.0f, scaleBaseZ));
        glUniformMatrix4fv(modelToCameraMatrixUnif, 1, GL_FALSE,
            glm::value_ptr(modelToCameraStack.Top()));
        glDrawElements(GL_TRIANGLES, ARRAY_COUNT(indexData),
            GL_UNSIGNED_SHORT, 0);
        modelToCameraStack.Pop();
    }

    //Draw right base.
    {
        modelToCameraStack.Push();
        modelToCameraStack.Translate(posBaseRight);
        modelToCameraStack.Scale(glm::vec3(1.0f, 1.0f, scaleBaseZ));
        glUniformMatrix4fv(modelToCameraMatrixUnif, 1, GL_FALSE,
            glm::value_ptr(modelToCameraStack.Top()));
        glDrawElements(GL_TRIANGLES, ARRAY_COUNT(indexData),
            GL_UNSIGNED_SHORT, 0);
        modelToCameraStack.Pop();
    }
    
    //Draw main arm.
    DrawUpperArm(modelToCameraStack);
    
    glBindVertexArray(0);
    glUseProgram(0);
}
```

MatrixStack
