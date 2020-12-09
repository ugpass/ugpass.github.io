---
title:      "OpenGL ES GLSL加载纹理" 
date:       2020-08-02
tags:
    - OpenGL ES 
    - GLSL
categories:
    - OpenGL ES 
    - GLSL
---

使用GLSL语言加载纹理，需要自定义顶点着色器和片源着色器。

GLSL编写的顶点着色器和片元着色器其实是一段代码，也是一段字符串，所以文件名和后缀可以自定义。

通过创建`Empty`文件，修改文件名为`shaderv.vsh` 和  `shaderf.fsh`，分别代表顶点着色器和片元着色器。在GLSL语言编写的文件中，建议不要写中文注释，否则可能发生一些奇怪的错误。

`顶点着色器 shaderv.vsh`代码

```
attribute vec4 position;
attribute vec2 textCoordinate;
varying lowp vec2 varyTextCoord;

void main() {
    varyTextCoord = textCoordinate;
    
    gl_Position = position;
}
```

>  解释：
>  attribute vec4 position; 指的是attribute属性的 4维向量 的顶点坐标。
>  attribute vec2 textCoordinate; 指的是attribute属性的 2维向量 的纹理坐标。
>  varying lowp vec2 varyTextCoord; 指的是通过顶点着色器透传到片元着色器中的 低精度 的 2维向量 的纹理坐标。
>  `varying`修饰符代表可以通过顶点着色器传参到片元着色器，在片元着色器中的接收变量 声明 需要和顶点着色器中 **一模一样**。
>  每一个着色器文件都有一个`main`函数。
>   varyTextCoord = textCoordinate; 将纹理坐标通过`varyTextCoord`变量传递到片元着色器。
>  gl_Position = position; 不对顶点坐标做任何处理，直接传入给OpenGL ES中的内建变量`gl_Position`。

`片元着色器 shaderf.fsh`代码

```
precision highp float;
varying lowp vec2 varyTextCoord;
uniform sampler2D colorMap;

void main() {
    lowp vec4 temp = texture2D(colorMap, varyTextCoord);
    
    gl_FragColor = temp;
}
```

> 解释：
> precision highp float; 在本文件中使用高精度的 `float`。
> varying lowp vec2 varyTextCoord; 声明和顶点着色器中一致，用来接收纹理坐标。
> uniform sampler2D colorMap;  纹理采样器
> texture2D(colorMap, varyTextCoord); 通过纹理采样器，获取纹理每个像素点对应的颜色值，赋值给OpenGL ES中的内建变量`gl_FragColor`。


使用GLSL编写自定义着色器加载纹理需要以下步骤：

  - 创建图层
  - 创建上下文
  - 清空缓冲区
  - 设置renderBuffer
  - 设置frameBuffer
  - 开始绘制

> 以下代码在自定义CustomView继承自UIView中编写。

**0.定义属性**
```
@interface CustomView()

@property (nonatomic, strong) CAEAGLLayer *eaglLayer;//自定义图层

@property (nonatomic, strong) EAGLContext *context;//上下文

@property (nonatomic, assign) GLuint renderBuffer;//渲染缓冲区ID
@property (nonatomic, assign) GLuint frameBuffer;//帧缓冲区ID

@property (nonatomic, assign) GLuint program;//程序ID

@end
```

**1.创建图层**
```
//1.设置图层
- (void)setupLayer {
    self.eaglLayer = (CAEAGLLayer *)self.layer;
    
    [self setContentScaleFactor:[UIScreen mainScreen].scale];
    
    /**
     kEAGLDrawablePropertyColorFormat:颜色缓冲区格式
     kEAGLDrawablePropertyRetainedBacking:绘制后是否保留其内容
     */
    self.eaglLayer.drawableProperties = [NSDictionary dictionaryWithObjectsAndKeys:@false, kEAGLDrawablePropertyRetainedBacking, kEAGLColorFormatRGBA8, kEAGLDrawablePropertyColorFormat, nil];
}
```

需要重写`+ (Class)layerClass;`方法才能强转生效。
```
+ (Class)layerClass {
    return [CAEAGLLayer class];
}
```

**2.创建上下文**
```
//2.设置上下文
- (void)setupContext {
    self.context = [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES2];
    if (!self.context) {
        NSLog(@"create context failed");
        return;
    }
    BOOL ret = [EAGLContext setCurrentContext:self.context];
    if (!ret) {
        NSLog(@"setCurrentContext failed");
        return;
    }
}
```

**3.清空缓冲区**
```
//3.清空缓冲区
- (void)clearRenderAndFrameBuffer {
    //Frame Buffer Object FBO
    //Render Buffer 三类：颜色缓冲区、深度缓冲区、模版缓冲区
    
    glDeleteRenderbuffers(1, &_renderBuffer);
    self.renderBuffer = 0;
    
    glDeleteFramebuffers(1, &_frameBuffer);
    self.frameBuffer = 0;
}
```

**4.设置renderBuffer**
```
//4.设置renderBuffer 
-(void)setupRenderBuffer
{
    //定义一个缓存区ID
    GLuint renderBuffer;
    
    //申请一个缓存区标志
    glGenRenderbuffers(1, &renderBuffer); 
    self.renderBuffer = renderBuffer;
    
    //将标识符绑定到GL_RENDERBUFFER
    glBindRenderbuffer(GL_RENDERBUFFER, self.renderBuffer);
    
    //将可绘制对象drawable object's  CAEAGLLayer的存储绑定到OpenGL ES renderBuffer对象
    [self.context renderbufferStorage:GL_RENDERBUFFER fromDrawable:self.eaglLayer];
}
```

**5.设置frameBuffer**
```
//5.设置frameBuffer
- (void)setupFrameBuffer {
    GLuint frameBuffer;
    
    glGenFramebuffers(1, &frameBuffer);
    
    self.frameBuffer = frameBuffer;
    
    glBindFramebuffer(GL_FRAMEBUFFER, self.frameBuffer);
    
    /*生成帧缓存区之后，则需要将renderbuffer跟framebuffer进行绑定，
     调用glFramebufferRenderbuffer函数进行绑定到对应的附着点上，后面的绘制才能起作用
     */
    //将渲染缓存区frameBuffer 通过glFramebufferRenderbuffer函数绑定到 GL_COLOR_ATTACHMENT0上。
    glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_RENDERBUFFER, self.renderBuffer);
}
```

**6.开始绘制**
```
//6.开始绘制
- (void)renderLayer {
    //设置背景色
    glClearColor(0.45, 0.5, 0, 1);
    //清空颜色缓冲区
    glClear(GL_COLOR_BUFFER_BIT);
    
    CGFloat scale = [UIScreen mainScreen].scale;
    //设置视口
    glViewport(self.frame.origin.x * scale, self.frame.origin.y * scale, self.frame.size.width * scale, self.frame.size.height * scale);
    
    //顶点着色器和片元着色器文件路径
    NSString *vertFilePath = [[NSBundle mainBundle] pathForResource:@"shaderv" ofType:@"vsh"];
    NSString *fragFilePath = [[NSBundle mainBundle] pathForResource:@"shaderf" ofType:@"fsh"];
    
    //加载顶点着色器和纹理着色器 创建program
    self.program = [self loaderShader:vertFilePath withFrag:fragFilePath];
    
    //链接program
    glLinkProgram(self.program);
    
    //获取program链接状态
    GLint linkStatus;
    glGetProgramiv(self.program, GL_LINK_STATUS, &linkStatus);
    if (linkStatus == GL_FALSE) {
        GLchar loginfo[512];
        glGetProgramInfoLog(self.program, sizeof(loginfo), 0, &loginfo[0]);
        NSString *message = [NSString stringWithUTF8String:loginfo];
        NSLog(@"program link error:%@", message);
        return;
    }
    
    //使用program
    glUseProgram(self.program);
    
    //准备顶点数据/纹理坐标
    GLfloat attrArr[] =
     {
         0.5f, -0.5f, -1.0f,     1.0f, 0.0f,
         -0.5f, 0.5f, -1.0f,     0.0f, 1.0f,
         -0.5f, -0.5f, -1.0f,    0.0f, 0.0f,
         
         0.5f, 0.5f, -1.0f,      1.0f, 1.0f,
         -0.5f, 0.5f, -1.0f,     0.0f, 1.0f,
         0.5f, -0.5f, -1.0f,     1.0f, 0.0f,
     };
    
    //将顶点坐标和纹理坐标拷贝到GPU中
    GLuint attrBuffer;
    glGenBuffers(1, &attrBuffer);
    glBindBuffer(GL_ARRAY_BUFFER, attrBuffer);
    glBufferData(GL_ARRAY_BUFFER, sizeof(attrArr), attrArr, GL_DYNAMIC_DRAW);
    
    //获取顶点数据通道ID v.sh positon 打开顶点通道 并设置数据读取方式
    GLuint position = glGetAttribLocation(self.program, "position");
    glEnableVertexAttribArray(position);
    glVertexAttribPointer(position, 3, GL_FLOAT, GL_FALSE, sizeof(GLfloat) * 5, (GLfloat *)NULL + 0);
    
    //打开纹理通道 并设置数据读取方式
    GLuint textCoordinate = glGetAttribLocation(self.program, "textCoordinate");
    glEnableVertexAttribArray(textCoordinate);
    glVertexAttribPointer(textCoordinate, 2, GL_FLOAT, GL_FALSE, sizeof(GLfloat) * 5, (GLfloat *)NULL + 3);

    //加载纹理
    [self setupTexture:@"mew_progressive.jpg"];
    
    //设置纹理采样器
    glUniform1i(glGetUniformLocation(self.program, "colorMap"), 0);
    
    //开始绘制
    glDrawArrays(GL_TRIANGLES, 0, 6);
    
    //从渲染缓冲区显示到屏幕上
    [self.context presentRenderbuffer:GL_RENDERBUFFER];
}
```

**加载纹理**
```
//加载纹理
- (GLuint)setupTexture:(NSString *)filePath {
    //获取图像CGImage
    CGImageRef cgImage = [UIImage imageNamed:filePath].CGImage;
    if (!cgImage) {
        NSLog(@"faile to load image");
        return -1;
    }
    
    //获取图片宽高
    size_t width = CGImageGetWidth(cgImage);
    size_t height = CGImageGetHeight(cgImage);
    
    //获取图片字节数 宽*高*4（RGBA）
    GLubyte *data = (GLubyte *)calloc(width * height * 4, sizeof(GLubyte));
    
    //创建上下文
    /*
     参数1：data,指向要渲染的绘制图像的内存地址
     参数2：width,bitmap的宽度，单位为像素
     参数3：height,bitmap的高度，单位为像素
     参数4：bitPerComponent,内存中像素的每个组件的位数，比如32位RGBA，就设置为8
     参数5：bytesPerRow,bitmap的没一行的内存所占的比特数
     参数6：colorSpace,bitmap上使用的颜色空间  kCGImageAlphaPremultipliedLast：RGBA
     */
    CGContextRef cgContext = CGBitmapContextCreate(
                                                   data,
                                                   width,
                                                   height,
                                                   8,
                                                   width * 4,
                                                   CGImageGetColorSpace(cgImage),
                                                   kCGImageAlphaPremultipliedLast);
    //使用CGContextRef 将图片绘制出来 也是一个解码的过程
    /*
     CGContextDrawImage 使用的是Core Graphics框架，坐标系与UIKit 不一样。UIKit框架的原点在屏幕的左上角，Core Graphics框架的原点在屏幕的左下角。
     CGContextDrawImage
     参数1：绘图上下文
     参数2：rect坐标
     参数3：绘制的图片
     */
    CGRect rect = CGRectMake(0, 0, width, height);
    
    CGContextDrawImage(cgContext, rect, cgImage);
    
    CGContextRelease(cgContext);
    
    //纹理 当只有一个纹理时 纹理ID为0, 多个纹理需要激活
    GLuint textureID;
    glGenTextures(1, &textureID);
    
    glBindTexture(GL_TEXTURE_2D, textureID);
    
    //设置纹理属性
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    
    float fw = width, fh = height;
    //载入纹理2D数据
    /*
     参数1：纹理模式，GL_TEXTURE_1D、GL_TEXTURE_2D、GL_TEXTURE_3D
     参数2：加载的层次，一般设置为0
     参数3：纹理的颜色值GL_RGBA
     参数4：宽
     参数5：高
     参数6：border，边界宽度
     参数7：format
     参数8：type
     参数9：纹理数据
     */
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, fw, fh, 0, GL_RGBA, GL_UNSIGNED_BYTE, data);
    
    free(data);
    return 0;
}
```

**加载着色器**
```
//加载着色器
- (GLuint)loaderShader:(NSString *)vert withFrag:(NSString *)frag {
    //顶点着色器对象 片元着色器对象/句柄
    GLuint verShader, fragShader;
    
    //创建空的program
    GLuint program = glCreateProgram();
    
    //编译
    [self compileShader:&verShader type:GL_VERTEX_SHADER filePath:vert];
    [self compileShader:&fragShader type:GL_FRAGMENT_SHADER filePath:frag];
    
    //把shader附着 到编译好的程序
    glAttachShader(program, verShader);
    glAttachShader(program, fragShader);
    
    //附着之后 就可以删除
    glDeleteShader(verShader);
    glDeleteShader(fragShader);
    
    return program;
}
```

**编译program**
```
//编译
- (void)compileShader:(GLuint *)shader type:(GLenum)type filePath:(NSString *)filePath {
    //读取路径
    NSString *content = [NSString stringWithContentsOfFile:filePath encoding:NSUTF8StringEncoding error:nil];
    const GLchar *source = [content UTF8String];
    
    //创建对应类型的shader
    *shader = glCreateShader(type);
    
    //将着色器附着到 着色器对象上
    glShaderSource(*shader, 1, &source, NULL);
    
    //编译
    glCompileShader(*shader);
}
```

demo地址：[GLSL加载图片]([https://github.com/ugpass/GLSL_Hello_World](https://github.com/ugpass/GLSL_Hello_World)
)

> `glVertexAttribPointer`函数最后一个参数，从哪里开始访问数据的 `NULL`， 如果不加`(GLfloat *)`，将会导致纹理加载不上去！！！

> 由于CoreGraphics坐标系与UIKit不一致，导致图片显示为倒的，之后介绍五种旋转图片的方法。