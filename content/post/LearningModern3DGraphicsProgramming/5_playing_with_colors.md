+++
date="2017-05-30T12:00:00+06:00" 
title="5. 设置颜色"
categories=["Translation of books"] 
tags=["opengl", "LearningModern3DGraphicsProgramming"]
+++

# 设置颜色

这一章会对上一章中绘制的三角形进行颜色的设定。而不是单纯的设置一个单一的颜色，这里我们会使用两种方式来对这个三角形设置颜色的变化。这些方法有使用片段位点来计算颜色，和前一个顶点数据来计算颜色。

## 片段位置显示

正如我们在引言中提到的，片段的数据中的一部分包括片段在屏幕上的位置。因此，如果我们想要在三角形表面上设定变化的颜色，我们可以访问当前片段着色器内的数据，并用来计算最终的颜色。

在这一章中，我们对纹理的加载，项目对象的创建进行了封装。具体代码如下：

```
// TODO add source code
#ifndef _GLSL_PROGRAM_
#define _GLSL_PROGRAM_

#include <vector>
#include <fstream>
#include <string>
#include <sstream>

class GLSLProgram
{
public:
    bool AddShader(GLenum shaderType, const std::string& shaderFileName)
    {
        GLuint shader = glCreateShader(shaderType);

        std::ifstream ifile(shaderFileName.c_str());
        std::stringstream buffer;
        buffer << ifile.rdbuf();
        std::string contents(buffer.str());
        const char* pContents =  contents.c_str();

        glShaderSource(shader, 1, &pContents, NULL);

        glCompileShader(shader);
        GLint status;
        STATUS, &status);
        if (status == GL_FALSE)
        {
            GLint infoLogLength;
            glGetShaderiv(shader, GL_INFO_LOG_LENGTH, &infoLogLength);
            GLchar *strInfoLog = new GLchar[infoLogLength+1];
            glGetShaderInfoLog(shader, infoLogLength, NULL, strInfoLog);
            fprintf(stderr, "Compile failure in %d shader: \n%s\n", shaderType, strInfoLog);
            delete[] strInfoLog;
            return false;
        }

        shaderList_.push_back(shader);
        return true;
    }

    bool LinkProgram()
    {
        bool result(true);
        program_ = glCreateProgram();
        for (size_t i=0; i<shaderList_.size(); ++i)
        {
            glAttachShader(program_, shaderList_[i]);
        }

        glLinkProgram(program_);

        GLint status;
        glGetProgramiv(program_, GL_LINK_STATUS, &status);
        if (status == GL__FALSE)
        {
            GLint infoLogLength;
            glGetProgramiv(program_, GL_INFO_LOG_LENGTH, &infoLogLength);
            printf("Log length: %d\n", infoLogLength);
            GLchar *strInfoLog = new GLchar[infoLogLength+1];
            glGetProgramInfoLog(program_, infoLogLength, NULL, strInfoLog);
            fprintf(stderr, "Linker failure: %s\n", strInfoLog);
            delete[] strInfoLog;
            result = false;
        }

        for (size_t i=0; i<shaderList_.size(); i++)
        {
            glDetachShader(program_, shaderList_[i]);
            glDeleteShader(shaderList_[i]);
        }
        return result;
    }

    void UseProgram()
    {
        glUseProgram(program_);
    }

    GLuint GetProgramID()
    {
        return program_;
    }

private:
    std::vector<GLuint> shaderList_;
    GLuint program_;
};

#endif // _GLSL_PROGRAM_

```

本章中用到的片段位点的着色器是`FragPosition.vert`和`FragPosition.frag`。顶点着色器和上一章用到的着色器相同。片段着色器颜色输出的部分有了相应的改动。

```
#version 330
out vec4 output

void main()
{
    float lerpValue = gl_FragCoord.y / 500.0f;
    outputColor = mix(vec4(1.0f,1.0f,1.0f,1.0f), vec4(0.2f,0.2f,0.2f,1.0f), lerpValue);
}
```

`gl_FragCoord`是片段着色器内建的变量类型。它是一个vec3的变量，具有x,y,z三个组成成分。其中x，y表示的是窗口坐标，所以这些值的绝对值会随着窗口分辨率的改变而改变。该窗口坐标系的原点位于坐下方。因此三角形靠近下边的顶点的y值要比上边点的y值小。

上面的片段着色器中的代码是根据窗口位置的y值进行设定的。500表示默认的窗口的高度。在不改变窗口大小的情况下，lerpValue的值在[0,1]范围内。1表示在窗口的顶点，0表示在窗口的底部。

main函数中的第二行用这个lerpValue"对两个vec4进行线性插值。`mix`是众多着色器语言提供的函数之一。与mix类似的函数可以对vec中的每个分量进行相同的处理。因此，参数的维度必须是相同的。

`mix`函数执行了现行插值运算。如果第三个参数是0,那么返回的值就是第一个入参，如果第三个参数是1,那么返回的就是第二个参数，如果第三个参数是0到1之间的值，那么返回的值，就在第一个参数和第二个参数之间。

**Note**

> 第三个参数必须在，[0,1]范围。但是，GLSL并不会替你进行范围的检查。如果这个值不是在[0,1]的范围内，那么返回的结果是没有定义的。在opengl中”没有定义“的意思是，”返回的值可能不是你想要的“。

我们可以得到如下的结果。

![](./image/figure2_1_fragment_position.png) **Figrue 2.1** Fragment Position

在这个例子中，越靠近下面的位置更接近白色，越靠近上面的位置，越接近黑色。除了片段着色器，想对于第一章，并没有改变太多的代码。


## 顶点属性

在片段着色器中使用片段的位置是非常有用的，但是对于控制三角形的颜色并不是最好的选择。一个更好的方式是，分别设置三角形各个顶点的颜色。这一节中使用了该方式。

我们想要改变对传递给系统的数据。我们希望事情发生的顺序如下：

1. 对于我们传递给顶点着色器的每一个顶点的位置，都传入特定的颜色值。
2. 每一个顶点着色器的输出，我们同样希望输出与顶点着色器接受到的相同的颜色。
3. 在片段着色器中，我们将从顶点着色器中输出的颜色值作为片段着色器的输出颜色。

对与上面的流程，你很有可能会有一些问题，如第2，3步骤是怎么工作的。我们会按照opengl的pipeline进行介绍。

### 多顶点数组和属性

为了完成第一个步骤，我们需要对我们的顶点数组进行一定的更改，现在数据看起来是这样的：

```
const float vertexData[] = {
    0.0f,    0.5f, 0.0f, 1.0f,
    0.5f, -0.366f, 0.0f, 1.0f,
    -0.5f, -0.366f, 0.0f, 1.0f,
    1.0f,    0.0f, 0.0f, 1.0f,
    0.0f,    1.0f, 0.0f, 1.0f,
    0.0f,    0.0f, 1.0f, 1.0f, 
}
```

首先，我们需要从底层理解数组。单个字节是c/c++可以表示的最小的单位。一个字节有8个位(一个位可以是0或1)，一个字节可以表示的一个数字的范围是[0,255]。一个float类型有4个字节的存储空间。任何一个float都会占用内存连续的4个字节。

4个字节的组合，在glsl中表示成vec4，确切的是一个4个float值的序列。因此，vec4占用了16个字节。

`vertexData`是一个占用了更多空间的float数组。我们使用它的方式是将其当成两个数组。每连续的4个float可以表示成vec4，前三个vec4表示顶点的位置，后三个vec4表示的是对应顶点位置的颜色。

在内存中，vertexData数组看起来是这个样子的：

![](./image/figure2_2_vertex_array_memory_map.png)  **Figure 2.2 Vertex Array Memory Map**

图中上面的两个表示了基本数据类型的存储结构，其中每一个小格子表示一个字节。最下面一行的图表显示了整个数组的存储结构。每一个小格子表示一个float数值。左边的一半用来表示顶点的位置，右边的一半用来表示颜色。

这里我们拥有的是两个相互临近的数组。其中一个数组在内存中的起始位置是`&vertexData[0]`，另一个数组的起始位置是`&vertexData[12]`。

vertexData中存储的数据会被存储到缓存对象中。下面的代码我们在之前见过：

**Example2.3 Buffer Object Initialization**

```
void InitializeVertexBuffer()
{
    glGenBuffers(1, &vertexBufferObject);
    
    glBindBuffer(GL_ARRAY_BUFFER, vertexBufferObject);
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertexData), vertexData, GL_STATIC_DRAW);
    glBindBuffer(GL_ARRAY_BUFFER, 0);
}
```

上面的代码并没有改变，因为数组的大小是通过`sizeof`直接计算得到的。因此，当我们想数组中添加元素时，这个大小会自然而然的增大。

同时，你也许会注意到，三角形的定点和上一节中的代码不同了，上一节中是一个直角等腰三角形，现在的是等边三角形。

通过上面的代码，我们将顶点和颜色都传到了缓存对象中了。现在我们需要告诉opengl如何去识别这两个不同类型的数组。

**Example 2.4 Rendering the Scene**

```
void dispaly()
{
    glClearColor(0.0f, 0.0f, 0.0f, 0.0f);
    glClear(GL_COLOR_BUFFER_BIT);

    glUseProgram(theProgram);

    glBindBuffer(GL_ARRAY_BUFFER, vertexBufferObject);
    glEnableVertexAttribArray(0);
    glEnableVertexAttribArray(1);
    glVertexAttribPointer(0, 4, GL_FLOAT, GL_FALSE, 0, 0);
    glVertexAttribPointer(1, 4, GL_FLOAT, GL_FALSE, 0, (void*)48);
    glDrawArrays(GL_TRIANGLES, 0, 3);

    glDisableVertexAttribArray(0);
    glDisableVertexAttribArray(1);
    glUseProgram(0);

    glutSwapBuffers();
    glutPostRedisplay();
}
```

由于我们有两段数据，我们有两个顶点属性。对于每个顶点属性，我们都必须调用`glEnableVertexAttribArray`来开启特定的属性。入参是顶点着色器中通过`layout(location)`设定的属性位置。

然后，我们针对每个我们想使用的属性调用`glVertexAttribPointer`。两个该函数的调用唯一的区别是设置的是那个属性位置，以及最后一个入参。最后一个入参是该属性的参数开始的数组的偏移量。在这个例子中，这个偏移量是float的字节数4,乘以vec4中有的float个数，乘以顶点的个数3,为48。

**Note**

> 如果你好奇为什么是(void*)48，而不是48,因为这是接口遗留问题。`glVertexAttrib"Pointer"`之所以以Pointer结尾，因为最后一个参数是指针。或者说在以前至少是这样的。因此，我们需要显式的将48转换成指针类型。

在这之后，我们使用`glDrawArrays`来进行渲染，然后使得数组不起作用，使用接口`glDisableVertexAttribArray`。

#### 关于绘制，更详细的介绍

在上一章中，我们跳过了glDrawArrays的详细介绍。这里，让我们更进一步来看一下。

设定到opengl不同的属性数组是在渲染的时候被读取的。在我们的示例中，我们有两个数组。每一个数组有一个缓存对象和一个这个数组在缓存对象中的偏移量，但是这些数组的大小还没有被设定。如果你看c++的伪代码，会有如下的形式：

```
GLbyte *bufferObject = (void*){0.0f,0.5f,0.0f,1.0f,0.5f,-0.366f, ...};
float* positionAttribArray[4] = (float *[4])(&(bufferObject+0));
float *colorAttribArray[4] = (float *[4])(&(bufferObject+48));
```

positionAttribArray中的每个元素还有4个成分，这个大小和顶点着色器中的vec4大小相等。`glVertexAttribPointer`第二个参数是4的原因正是这个。没一个成分都是float数据；这也决定了第三个参数是`GL_FLOAT`。这个数组从bufferObject获取数据，这个也正是glVertexAttriPointer调用的时候绑定的缓存区对象。偏移量0,正好是`glVertexAttribPointer`的最后一个参数。

对于`colorAttribArray`也是类似的说明，除了偏移量是48。

使用上述伪代码中的表述方式，`glDrawArrays`可以表达成：

```
void glDrawArrays(GLenum type, GLint start, GLint count)
{
    for (GLint element=start; element<start+count; element++)
    {
        VertexSahder(positionAttribArray[element], colorAttribArray[element]);
    }
}
```

这意味着顶点着色器执行的次数是`count`，并且对应的数据是从`start`开始的`count`个元素。

从缓存对象到顶点着色器的数据流类似如下：

![](./image/figure2_3_multiple_vertex_attributes.png)  **Multiple Vertex Attributes**

和原来一样，每三个顶点可以组成一个三角形。

### 顶点着色器

**Example 2.7 Multi-input Vertex Shader**

```
#version 330
layout (location=0) in vec4 position;
layout (location=1) in vec4 color;

smooth out vec4 theColor;

void main()
{
    gl_Position = position;
    theColor = color;
}
```

这里有三行是新出现的内容。

定义了一个全局的`color`变量，作为顶点着色器的输入。因此，这个着色器，除了输入了position以外，还将color作为了输入参数。position属性被赋予0的属性位置。color被赋予1的属性位置。

上面两行仅仅将参数输入到顶点着色器，我们还需要将特定数据输出。为了完成这个目的，我们首先需要定义一个输出参数，这个使用`out`关键字实现的，这里输出参数`theColor`的类型是vec4.

`smooth`是与插值相关的关键词。在后续的内容中，我们会详细的讨论他。

当然，除了简单的定义输出变量之外，我们还在main函数中对这个输出变量进行了赋值。在着色器代码中，我们通常需要一些复杂的计算才能够确定特定位点的颜色，这里，为了简化问题，我们仅仅使用了输入作为了颜色的输出。用户定义的输出变量对于系统而言是没有严格的意义的。仅仅当接下去的着色器使用他们的时候，他们才是有意义的。

### 片段着色器

新的片段着色器代码如下：

```
#version 330
smooth in vec4 theColor;

out vec4 outputColor;

void main()
{
    outputColor = theColor;
}
```

这个片段着色器我们定义了一个输入变量。这个输入参数的名字并和上一个着色器中输出参数的名字保持一致,类型也同样保持一致，这并不是一种巧合，而是特殊的规定。我们尝试将顶点着色器中的信息传递给片段着色器。为了达成这个目的，我们需要保证着色器输出阶段的名字和类型必须和输入阶段的名字和类型保持一致。并且还需要使用相同的插值限定词（smooth）。

这里对于opengl需要将顶点着色器和片段着色器链接到一块儿有个比较好的理由，就是如果名字，类型或插值限定词不一致，那么opengl在链接的时候就会抛出错误。

这里片段着色器中，将从顶点着色器中传入的颜色直接设置给输出参数。

### 片段着色器插值

这里有个基础的交互问题。我们的顶点着色其会被运行三次。这个执行可以产生3个输出点和3个输出颜色。这三个输出点被用来构建和光栅化三角形，产生一系列的片段。

片段着色器的运行次数不是3。对于每个生成的片段都会运行。一个三角形能够产生的片段着色器的数量，依赖于视线的分辨率，以及在当前屏幕上三角形覆盖了多少的面积。一个变长为1的等边三角形的面积大约为0.433。整个屏幕的面积(x,y in [-1,1])的面积为4，因此三角形的面积大概占了屏幕面积的10分之一。如果窗口的大小为500x500,拥有的像素为250000个像素，那么它的十分之一就是25000。因此，我们的片段着色器大概会被执行2500次。

这里有一些不同的地方。如果，顶点着色器完全和片段着色器对应，那么顶点着色器仅仅有三个颜色值，其它的24997个颜色从什么地方来的呢？

这个答案就是片段插值。

通过使用`smooth`的插值方式，我们告诉opengl要对`smooth`标识的变量进行特殊处理。除了没一个片段着色器接收从顶点着色器传过来的值之外，每个片段着色器还接收到了三个输出值之间的混合值。越接近三个顶点的片段，受到越多的顶点输出值的贡献。

到目前为止，这样的插值方式是顶点着色器和片段着色器之间最为常用的方式。如果你没有提供插值关键词，那么`smooth`就是默认的方式。除了`smooth`之外还有：noperspective和flat。

如果在这一节的练习中，将smooth改成noperspective，你并不会看到差别。这并不是说明这两个方式之间没有区别，而是我们这个练习过为简单，无法使他们差异化。这两种方式之间的差别是微妙的，在后续的内容中我们会有更详细的说明。

`flat`插值方式，其实是不使用插值。他就是说，每个片段只是简单的获取三个顶点着色器输出的第一个值，而不是使用插值。这种方式得到的三角形的颜色是均一的，平面的，因此被称为`flat`。

每个光栅话的三角形都有属于自己的三个输出用来计算三角形片段的值。因此，如果我们渲染两个三角形，一个三角形的渲染并不会影响到另一个三角形的渲染。也就是说，两个三角形之间是独立的。

从相同的顶点数据，来绘制多个三角形也是有可能的。在后续的内容中我们会进行讨论。

### 最后的图

![](./image/figure2_4_interpolated_vertex_colors.png)  **figure2_4_interpolated_vertex_colors.png**

## 回顾

在这一章中，我们学到了以下内容：

- 数据是通过缓存对象和属性数组传递给顶点着色器的。这些数据构成了三角形。
- 这个gl_FragCoord是glsl内建变量，用来获得当前窗口坐标系中该片段的位置。
- 在顶点着色器中使用output变量，可以将输入参数传递给片段着色器。
- 基于顶点着色器和片段着色器设定的插值关键词，数据可以三角形表面插值的方式从顶点着色器传递到片段着色器。

### 进一步的学习

- 在FragPosition例子中，改变viewport。将视口改变成窗口的上半部分，或下半部分。看一下，这将如何影响到三角形。
- 将FragPosition和VertexColor例子结合到一起，使用插值得到的颜色和FragPosition确定的颜色的乘积作为最后的颜色。

### GLSL函数

`vec min(vec initial, vec final, float alpha);`

对initial,final进行插值操作。alpha为0时，结果是initial，alpha为1时，结果为final。用公式进行表示就是`result=initial+(final-initial)*alpha`

## 代码

### 通用代码

为了更少的更改每一章的代码，将需要更具每个章节定制的函数提取到了common.h中，如下：

```
#ifndef _COMMON_H_
#define ___COMMON_H_

void init();
void display();
void reshape(int w, int h);
void mouseClick(int button, int state, int x, int y);
void mouseMotion(int x, int y);
void keyboard(unsigned char key, int x, int y);
extern int windowWidth;
extern int windowHeight;

#endif
```

以及通用的main.cpp如下：

```
#include <GL/glew.h>
#include <GL/glut.h>
#include <iostream>

#include "common.h"


int main(int argc, char*argv[])
{
    glutInit(&argc, argv);

    glutInitDisplayMode(GLUT_DOUBLE | GLUT_DEPTH);
    glutInitWindowSize(windowWidth, windowHeight);
    glutInitWindowPosition(300,200);
    glutCreateWindow(argv[0]);

    GLenum err = glewInit();
    if (err != GLEW_OK)
    {
        std::cout << "glewInit failed: " << glewGetErrorString(err) << std::endl;
    }

    init();
    glutDisplayFunc(display);
    glutReshapeFunc(reshape);
    glutMouseFunc(mouseClick);
    glutMotionFunc(mouseMotion);
    glutKeyboardFunc(keyboard);
    glutMainLoop();
}
```

### FragPosition代码

fragPosition例子的代码如下：

```
#include <GL/glew.h>
#include <GL/glut.h>

#include "glsl_program.h"
#include "common.h"

int windowWidth=500;
int windowHeight=500;

GLuint positionBufferObject;
const float vertexPositions[] = {
    0.75f, 0.75f, 0.0f, 1.0f,
    0.75f, -0.75f, 0.0f, 1.0f,
    -0.75f, -0.75f, 0.0f, 1.0f,
};

GLSLProgram program;

void InitializeVertexBuffer()
{
    glGenBuffers(1, &positionBufferObject);

    glBindBuffer(GL_ARRAY_BUFFER, positionBufferObject);
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertexPositions), vertexPositions, GL_STATIC_DRAW);
    glBindBuffer(GL_ARRAY_BUFFER, 0);
}

void InitializeProgram()
{
    program.AddShader(GL_VERTEX_SHADER, "./FragPosition.vert");
    program.AddShader(GL_FRAGMENT_SHADER, "./FragPosition.frag");
    program.LinkProgram();
}

void init()
{
    InitializeProgram();
    InitializeVertexBuffer();
}

void display()
{
    glClearColor(0.0f,0.0f,0.0f,0.0f);
    glClear(GL_COLOR_BUFFER_BIT);

    program.UseProgram();

    glBindBuffer(GL__ARRAY_BUFFER, positionBufferObject);
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(0,4,GL_FLOAT,GL_FALSE,0,0);
    glDrawArrays(GL_TRIANGLES, 0, 3);
    glDisableVertexAttribArray(0);
    ARRAY_BUFFER, 0);
    glUseProgram(0);
    glutSwapBuffers();
}

void reshape(int w, int h)
{
    glViewport(0, 0, w, h);
}

void mouseClick(int button, int state, int x, int y)
{

}

void mouseMotion(int x, int y)
{

}

void keyboard(unsigned char key, int x, int y)
{

}
```

fragPosition顶点着色器和片段着色器分别如下：

```
/// FragPosition.vert
#version 450
layout(location = 0) in vec4 position;
void main()
{
    gl_Position = position;
}

/// FragPosition.frag
#version 450

out vec4 outputColor;

void main()
{
    float lerpValue = gl_FragCoord.y/500.f;
    outputColor = mix(vec4(1.0f,1.0f,1.0f,1.0f), vec4(0.2f,0.2f,0.2f,1.0f), lerpValue);
}
```

### Vertex Color 代码

vertex_color.cpp代码

```
#include <GL/glew.h>
#include <GL/glut.h>

#include "glsl_program.h"
#include "common.h"

int windowWidth=500;
int windowHeight=500;

GLuint positionBufferObject;
const float vertexData[] = {
    0.0f,    0.5f, 0.0f, 1.0f,
    0.5f, -0.366f, 0.0f, 1.0f,
    -0.5f, -0.366f, 0.0f, 1.0f,
    1.0f,    0.0f, 0.0f, 1.0f,
    0.0f,    1.0f, 0.0f, 1.0f,
    0.0f,    0.0f, 1.0f, 1.0f, 
};

GLSLProgram program;

void InitializeVertexBuffer()
{
    glGenBuffers(1, &positionBufferObject);

    glBindBuffer(GL_ARRAY_BUFFER, positionBufferObject);
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertexData), vertexData, GL_STATIC_DRAW);
    glBindBuffer(GL_ARRAY_BUFFER, 0);
}

void InitializeProgram()
{
    program.AddShader(GL_VERTEX_SHADER, "./VertexColor.vert");
    program.AddShader(GL_FRAGMENT_SHADER, "./VertexColor.frag");
    program.LinkProgram();
}

void init()
{
    InitializeProgram();
    InitializeVertexBuffer();
}

void display()
{
    glClearColor(0.0f,0.0f,0.0f,0.0f);
    glClear(GL_COLOR_BUFFER_BIT);

    program.UseProgram();

    glBindBuffer(GL__ARRAY_BUFFER, positionBufferObject);
    glEnableVertexAttribArray(0);
    glEnableVertexAttribArray(1);
    glVertexAttribPointer(0,4,GL_FLOAT,GL_FALSE,0,0);
    glVertexAttribPointer(1,4,GL_FLOAT,GL_FALSE,0,(void*)48);
    TRIANGLES, 0, 3);
    glDisableVertexAttribArray(0);
    glDisableVertexAttribArray(1);
    glBindBuffer(GL_ARRAY_BUFFER, 0);
    glUseProgram(0);
    glutSwapBuffers();
}

void reshape(int w, int h)
{
    glViewport(0, 0, w, h);
}

void mouseClick(int button, int state, int x, int y)
{

}

void mouseMotion(int x, int y)
{

}

void keyboard(unsigned char key, int x, int y)
{

}
```

color vertex 顶点着色器和片段着色器的代码见文中。
