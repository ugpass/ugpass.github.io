---
title:      "OpenGL ES&GLKit加载纹理" 
date:       2020-07-28
tags:
    - OpenGL ES 
categories:
    - OpenGL ES 
---

1.需要导入 `GLKit.framework`

2.在`ViewController`中导入

```
#import <GLKit/GLKit.h>

#import <OpenGLES/ES3/gl.h>
#import <OpenGLES/ES3/glext.h>
```

3.如果将`ViewController`的父类改为`GLKViewController`,则同时需要将`Main.storyboard`中的view的类别改为`GLKView`
![修改view类别.png](https://upload-images.jianshu.io/upload_images/1395687-259961490eb4539a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当然也可以不修改`ViewController`的父类，直接在`ViewController`中自定义`GLKView`

```
@property (nonatomic, strong) GLKView *glkView;
```

4.创建EAGLContext，并设置为当前Context

```
//1.配置初始环境
- (void)setUpConfig {
    //创建EAGLContext 并设置为currentContext
    _context = [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES3];
    if (!_context) {
        NSLog(@"_context create failed");
    }
    [EAGLContext setCurrentContext:_context];
    
    //配置glkView
    GLKView *glkView = (GLKView *)self.view;
    glkView.context = _context;
    //开启颜色缓冲区
    glkView.drawableColorFormat = GLKViewDrawableColorFormatRGBA8888;
    //开启深度测试缓冲区
    glkView.drawableDepthFormat = GLKViewDrawableDepthFormatNone;
    
    //设置glkView的背景颜色
    glClearColor(1.0, 0.0, 0.0, 1.0);
}
```

5.设置顶点数据以及纹理数据 拷贝到顶点缓冲区

```
//设置顶点数据以及纹理数据 拷贝到顶点缓冲区
-(void)setUpVertexData
{
    //1.设置顶点数组(顶点坐标,纹理坐标)
    /*
     纹理坐标系取值范围[0,1];原点是左下角(0,0);
     故而(0,0)是纹理图像的左下角, 点(1,1)是右上角.
     */
    GLfloat vertexData[] = {
        
        0.5, -0.5, 0.0f,    1.0f, 0.0f, //右下
        0.5, 0.5,  0.0f,    1.0f, 1.0f, //右上
        -0.5, 0.5, 0.0f,    0.0f, 1.0f, //左上
        
        0.5, -0.5, 0.0f,    1.0f, 0.0f, //右下
        -0.5, 0.5, 0.0f,    0.0f, 1.0f, //左上
        -0.5, -0.5, 0.0f,   0.0f, 0.0f, //左下
    };
 
    /*
     顶点数组: 开发者可以选择设定函数指针，在调用绘制方法的时候，直接由内存传入顶点数据，也就是说这部分数据之前是存储在内存当中的，被称为顶点数组
     
     顶点缓存区: 性能更高的做法是，提前分配一块显存，将顶点数据预先传入到显存当中。这部分的显存，就被称为顶点缓冲区
     */
    
    //2.开辟顶点缓存区
    //(1).创建顶点缓存区标识符ID
    GLuint bufferID;
    glGenBuffers(1, &bufferID);
    //(2).绑定顶点缓存区.(明确作用)
    glBindBuffer(GL_ARRAY_BUFFER, bufferID);
    //(3).将顶点数组的数据copy到顶点缓存区中(GPU显存中)
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertexData), vertexData, GL_STATIC_DRAW);
    
    //3.打开读取通道.
    /*
     (1)在iOS中, 默认情况下，出于性能考虑，所有顶点着色器的属性（Attribute）变量都是关闭的.
     意味着,顶点数据在着色器端(服务端)是不可用的. 即使你已经使用glBufferData方法,将顶点数据从内存拷贝到顶点缓存区中(GPU显存中).
     所以, 必须由glEnableVertexAttribArray 方法打开通道.指定访问属性.才能让顶点着色器能够访问到从CPU复制到GPU的数据.
     注意: 数据在GPU端是否可见，即，着色器能否读取到数据，由是否启用了对应的属性决定，这就是glEnableVertexAttribArray的功能，允许顶点着色器读取GPU（服务器端）数据。
   
    (2)方法简介
    glVertexAttribPointer (GLuint indx, GLint size, GLenum type, GLboolean normalized, GLsizei stride, const GLvoid* ptr)
   
    功能: 上传顶点数据到显存的方法（设置合适的方式从buffer里面读取数据）
    参数列表:
        index,指定要修改的顶点属性的索引值,例如
        size, 每次读取数量。（如position是由3个（x,y,z）组成，而颜色是4个（r,g,b,a）,纹理则是2个.）
        type,指定数组中每个组件的数据类型。可用的符号常量有GL_BYTE, GL_UNSIGNED_BYTE, GL_SHORT,GL_UNSIGNED_SHORT, GL_FIXED, 和 GL_FLOAT，初始值为GL_FLOAT。
        normalized,指定当被访问时，固定点数据值是否应该被归一化（GL_TRUE）或者直接转换为固定点值（GL_FALSE）
        stride,指定连续顶点属性之间的偏移量。如果为0，那么顶点属性会被理解为：它们是紧密排列在一起的。初始值为0
        ptr指定一个指针，指向数组中第一个顶点属性的第一个组件。初始值为0
     */
    
    //顶点坐标数据
    glEnableVertexAttribArray(GLKVertexAttribPosition);
    glVertexAttribPointer(GLKVertexAttribPosition, 3, GL_FLOAT, GL_FALSE, sizeof(GLfloat) * 5, (GLfloat *)NULL + 0);
    
    
    //纹理坐标数据
    glEnableVertexAttribArray(GLKVertexAttribTexCoord0);
    glVertexAttribPointer(GLKVertexAttribTexCoord0, 2, GL_FLOAT, GL_FALSE, sizeof(GLfloat) * 5, (GLfloat *)NULL + 3);
 
}
```

6.加载纹理

```
//加载纹理
-(void)setUpTexture
{
    //1.获取纹理图片路径
    NSString *filePath = [[NSBundle mainBundle] pathForResource:@"mew_interlaced" ofType:@"png"];
    
    //2.设置纹理参数
    //纹理坐标原点是左下角,但是图片显示原点应该是左上角.
    NSDictionary *options = [NSDictionary dictionaryWithObjectsAndKeys:@(1),GLKTextureLoaderOriginBottomLeft, nil];
    
    GLKTextureInfo *textureInfo = [GLKTextureLoader textureWithContentsOfFile:filePath options:options error:nil];
    
    //3.使用苹果GLKit 提供GLKBaseEffect 完成着色器工作(顶点/片元)
    cEffect = [[GLKBaseEffect alloc]init];
    cEffect.texture2d0.enabled = GL_TRUE;
    cEffect.texture2d0.name = textureInfo.name;
}
```

7.实现GLKView代理  绘制视图，如果是自定义`GLKView`需要设置当前控制器为代理

```
#pragma mark -- GLKViewDelegate
//绘制视图的内容
/*
 GLKView对象使其OpenGL ES上下文成为当前上下文，并将其framebuffer绑定为OpenGL ES呈现命令的目标。然后，委托方法应该绘制视图的内容。
*/
- (void)glkView:(GLKView *)view drawInRect:(CGRect)rect
{
    //1.清理颜色缓冲区、深度缓冲区、模型缓冲区
    glClear(GL_COLOR_BUFFER_BIT);
    
    //2.准备绘制
    [cEffect prepareToDraw];
    
    //3.开始绘制
    glDrawArrays(GL_TRIANGLES, 0, 6);
    
}
```