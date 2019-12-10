+++
date="2017-05-30T12:00:00+06:00" 
title="6. opengl使用不同的缓存对象"
categories=["Translation of books"] 
tags=["opengl", "LearningModern3DGraphicsProgramming"]
+++

# opengl使用不同的缓存对象

在[设置颜色](http://www.cnblogs.com/grass-and-moon/p/6684854.html)一章中，我们使用了一个缓存对象来存储点和颜色的信息。那么我们有没有可能，将点和颜色的信息分开存储呢？这在实际应用中也许可以使得各个属性之间保持相互的独立。本章补充内容需要做的事情就是这个。

## 相对于上一章需要改变的内容有

顶点属性和颜色分别独立存储：

``` c
const float vertex[] = {
    0.0f, 0.5f, 0.0f, 1.0f,
    0.5f, -0.366f, 0.0f, 1.0f,
    -0.5f, -0.366f, 0.0f, 1.0f,
};

const float color[] = {
    1.0f, 0.0f, 0.0f, 1.0f,
    0.0f, 1.0f, 0.0f, 1.0f,
    0.0f, 0.0f, 1.0f, 1.0f
};
```

### Vertex Array Object

上一章中，绘制图像所需的数据都存储在Vertex Buffer Object中，然后利用`glVertexAttribPointer`来告诉opengl该缓存对象中的哪个数据段是顶点位置属性，哪些数据段是颜色属性。如果想要将多个属性分别存储于独立的VBO中，那么此处就需要VAO出马上任了。VAO能够用来存储多个VBO对象的对象。它被设计用来存储用于完成对象渲染所需要的信息：这里包括，数据，数据格式，以及不同的数据对应的着色器中的location。

于是将上一章中的`InitializeVertexBuffer`改成`InitializeVAO`

``` c
void InitializeVAO()
{
    glGenVertexArrays(1, &vao);
    glBindVertexArray(vao);

    glGenBuffers(1, &positionBufferObject);

    glBindBuffer(GL_ARRAY_BUFFER, positionBufferObject);
    glBufferData(GL__ARRAY_BUFFER, sizeof(vertexData), vertexData, GL_STATIC_DRAW);
    glVertexAttribPointer(0, 4, GL_FLOAT, GL_FALSE, 0, 0);

    glGenBuffers(1, &colorBufferObject);
    glBindBuffer(GL_ARRAY_BUFFER, colorBufferObject);
    glBufferData(GL_ARRAY_BUFFER, sizeof(colorData), colorData, GL_STATIC_DRAW);
    glVertexAttribPointer(1, 4, GL_FLOAT, GL_FALSE, 0, 0);

    glBindBuffer(GL_ARRAY_BUFFER, 0);
}
```

同时display中的代码也需要进行如下的修改：

``` c
void display()
{
    glClearColor(0.0f, 0.0f, 0.0f, 0.0f);
    glClear(GL_COLOR_BUFFER_BIT);

    program.UseProgram();

    glBindVertexArray(vao);
    glEnableVertexAttribArray(0);
    glEnableVertexAttribArray(1);

    glDrawArrays(GL___TRIANGLES, 0, 3);
    glDisableVertexAttribArray(0);
    glDisableVertexAttribArray(1);

    glUseProgram(0);
    glutSwapBuffers();
}
```
