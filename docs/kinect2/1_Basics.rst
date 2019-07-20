Kinect 基础
==============


:目标: 学习如何初始化一台 Kinect，并从中获取 RGB 类型的数据。

:源码: `点此查看 <https://github.com/XinArkh/kinect-tutorials-zh/tree/master/src/kinect2/1_Basics>`_    :download:`1_Basics.zip <../../src/kinect2/1_Basics.zip>`


概述
-------

我们有两段与 Kinect 相关的代码，下文将详细讨论这些内容，然后对展示部分的代码做一个简要介绍。


包含文件，常量和全局变量
-----------------------------


包含文件
+++++++++++

文件名基本已经解释清楚了它自己：\ ``Kinect.h``\ ，即为 Kinect 的主要头文件。

为了让 Kinect 的包含文件正常工作，你还需要在前面添加\ ``Ole2.h``\ 和\ ``Windows.h``\ 。同时也不要忘了添加可视化相关的包含文件。

.. code:: cpp

    // GLUT
    #include <Windows.h>
    #include <Ole2.h>

    #include <gl/GL.h>
    #include <gl/GLU.h>
    #include <gl/glut.h>

    #include <Kinect.h>

.. code:: cpp

    // SDL
    #include <Windows.h>
    #include <Ole2.h>

    #include <SDL_opengl.h>
    #include <SDL.h>

    #include <Kinect.h>


常量和全局变量
++++++++++++++++++

我们把窗口的宽度和高度定义为 1920*1080，这恰好也是 Kinect 摄像头输入的图像尺寸。

注意，\ ``data``\ 数组将会保存一份我们从 Kinect 中获取的图像的副本，以便我们将其作为纹理 (texture)。有经验的 OpenGL 用户也可以使用帧缓存对象 (Frame Buffer Object) 来代替它。

.. code:: cpp

    #define width 1920
    #define height 1080

    // OpenGL Variables
    GLuint textureId;              // ID of the texture to contain Kinect RGB Data
    GLubyte data[width*height*4];  // BGRA array containing the texture data

    // Kinect variables
    IKinectSensor* sensor;         // Kinect sensor
    IColorFrameReader* reader;     // Kinect color data source


Kinect 代码
---------------


Kinect 初始化
+++++++++++++++


下面是我们的第一段与 Kinect 相关的代码。\ ``initKinect()``\ 函数的作用是初始化一个 Kinect 设备。它包含了两个部分：首先，我们需要找到一个已连接到电脑的 Kinect 传感器，然后将其初始化并准备从中读取数据。

.. code:: cpp

    bool initKinect() {
        if (FAILED(GetDefaultKinectSensor(&sensor))) {
            return false;
        }
        if (sensor) {
            sensor->Open();

            IColorFrameSource* framesource = NULL;
            sensor->get_ColorFrameSource(&framesource);
            framesource->OpenReader(&reader);
            if (framesource) {
                framesource->Release();
                framesource = NULL;
            }
            return true;
        } else {
            return false;
        }
    }

以下是一些注意事项：

- 通常情况下，我们需要对上面所有函数的返回值都倍加小心；但是为了简单起见，我们在这里忽略了一些内容。

- 注意数据流请求的一般模式：
    #. 编写一个适当类型的数据帧输入源 (framesource)，包括彩色 (Color)、深度 (Depth)、肢体 (Body) 等。
    #. 通过传感器接口来请求数据帧输入源。
    #. 从数据帧输入源中打开读取器。
    #. 安全地释放数据帧输入源。
    #. 从读取器中请求数据。


从 Kinect 中获取 RGB 帧
++++++++++++++++++++++++++++++

.. code:: cpp

    void getKinectData(GLubyte* dest) {
        IColorFrame* frame = NULL;
        if (SUCCEEDED(reader->AcquireLatestFrame(&frame))) {
            frame->CopyConvertedFrameDataToArray(width*height*4, data, ColorImageFormat_Bgra);
        }
        if (frame) frame->Release();
    }

这个函数非常简单。我们从数据源中轮询 (poll for) 一个帧，如果有可用的帧，我们就将它以适当的格式复制到纹理 (texture) 数组中。不要忘记在这之后还要释放掉数据帧。

注意，原始彩色帧可能是 YUV 或其它类似的格式，因此需要一些处理将其转换为可用的 RGB/BGR 格式。你也可以使用\ ``frame->AccessUnderlyingBuffer()``\ 访问原始数据，我们将在下一章节的教程中使用它。

数据帧的元数据也是可以访问的。下面是一些你可能感兴趣的事情:

- 相机设置，如曝光 (exposure) 和增益 (gain)，可以通过\ ``IColorCameraSettings``\ 来访问：

.. code:: cpp

        IColorCameraSettings* camerasettings;
        frame->get_ColorCameraSettings(&camerasettings);
        float gain;
        TIMESPAN exposure;
        camerasettings->get_Gain(&gain);
        camerasettings->get_ExposureTime(&exposure);
        // ...etc.

- 维度和视野范围可以通过\ ``IFrameDescription``\ 访问：

.. code:: cpp

        IFrameDescription* description;
        frame->get_FrameDescription(&description);
        int height, width;
        float xfov, yfov;
        description->get_Height(&height);
        description->get_Width(&width);
        description->get_HorizontalFieldOfView(&xfov);
        description->get_VerticalFieldOfView(&yfov);
        // ...etc.

这也适用于深度数据帧和红外数据帧。

更多细节可以访问\ `这个页面 <https://msdn.microsoft.com/en-us/library/microsoft.kinect.kinect.icolorframe.aspx>`_\ 。

以上就是所有 Kinect 部分的代码！剩下的就是如何把它搬上屏幕。


窗口化，事件处理和主循环
-------------------------------

本节将会解释与 GLUT——或 SDL——相关的代码，包括了窗口初始化、事件处理和主更新循环。

具体的初始化代码取决于使用那种实现方式 (GLUT 或 SDL)。它只是使用适当的 API 初始化一个窗口，失败时返回 *false*\ 。GLUT 版本的实现还会通过指定\ ``draw()``\ 函数在每次循环迭代中被调用来设置主循环。

主循环在\ ``execute()``\ 函数中启动。在 GLUT中，循环是在后台处理的，所以我们需要做的就是调用\ ``glutMainLoop()``\ 函数。在 SDL 中，我们编写自己的循环。在每个循环中，我们在屏幕上绘制新的帧，这个处理是在\ ``drawKinect()``\ 函数中完成的。

如果你希望通过 GLUT 或 SDL 进行更复杂的窗口和循环管理，或学习更多关于这些函数的知识，网络上也有很多其它的参考资料。

**GLUT**

.. code:: cpp

    void draw() {
       drawKinectData();
       glutSwapBuffers();
    }

    void execute() {
        glutMainLoop();
    }

    bool init(int argc, char* argv[]) {
        glutInit(&argc, argv);
        glutInitDisplayMode(GLUT_DEPTH | GLUT_DOUBLE | GLUT_RGBA);
        glutInitWindowSize(width,height);
        glutCreateWindow("Kinect SDK Tutorial");
        glutDisplayFunc(draw);
        glutIdleFunc(draw);
        return true;
    }


**SDL**

.. code:: cpp

    void execute() {
        SDL_Event ev;
        bool running = true;
        while (running) {
            while (SDL_PollEvent(&ev)) {
                if (ev.type == SDL_QUIT) running = false;
            }
            drawKinectData();
            SDL_GL_SwapBuffers();
        }
    }

    bool init(int argc, char* argv[]) {
        SDL_Init(SDL_INIT_EVERYTHING);
        SDL_Surface* screen =
            SDL_SetVideoMode(width, height, 32, SDL_HWSURFACE | SDL_GL_DOUBLEBUFFER | SDL_OPENGL);
        return screen;
    }


通过 OpenGL 显示
------------------


初始化
++++++++

代码中描述了三个步骤——设置纹理以包含图像帧，准备 OpenGL 来绘制纹理，以及设置摄像机视点（对 2D 图像使用正投影）。

.. code:: cpp

        // Initialize textures
        glGenTextures(1, &textureId);
        glBindTexture(GL_TEXTURE_2D, textureId);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
        glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA8, width, height,
                     0, GL_BGRA, GL_UNSIGNED_BYTE, (GLvoid*) data);
        glBindTexture(GL_TEXTURE_2D, 0);

        // OpenGL setup
        glClearColor(0,0,0,0);
        glClearDepth(1.0f);
        glEnable(GL_TEXTURE_2D);

        // Camera setup
        glViewport(0, 0, width, height);
        glMatrixMode(GL_PROJECTION);
        glLoadIdentity();
        glOrtho(0, width, height, 0, 1, -1);
        glMatrixMode(GL_MODELVIEW);
        glLoadIdentity();

显然，我们应该用一个函数把上面的片段包起来，这里为了方便直接把它塞进了\ ``main()``\ 函数中。

.. code:: cpp

    int main(int argc, char* argv[]) {
        if (!init(argc, argv)) return 1;
        if (!initKinect()) return 1;

        /* ...OpenGL texture and camera initialization... */

        // Main loop
        execute();
        return 0;
    }


将图像帧画到屏幕上
++++++++++++++++++++++

这部分是很常规的代码。首先将 Kinect 数据复制到我们的缓存区中，然后指定我们的纹理使用这个缓冲区。

.. code:: cpp

    void drawKinectData() {
        glBindTexture(GL_TEXTURE_2D, textureId);
        getKinectData(data);
        glTexSubImage2D(GL_TEXTURE_2D, 0, 0, 0, width, height, GL_BGRA, GL_UNSIGNED_BYTE, (GLvoid*)data);

然后，绘制一个以图像帧为纹理的方框。

.. code:: cpp

        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
        glBegin(GL_QUADS);
            glTexCoord2f(0.0f, 0.0f);
            glVertex3f(0, 0, 0);
            glTexCoord2f(1.0f, 0.0f);
            glVertex3f(width, 0, 0);
            glTexCoord2f(1.0f, 1.0f);
            glVertex3f(width, height, 0.0f);
            glTexCoord2f(0.0f, 1.0f);
            glVertex3f(0, height, 0.0f);
        glEnd();
    }

结束！构建并运行，确保你的 Kinect 已经插入。你应该会看到一个包含 Kinect 所拍摄内容的视频流窗口。