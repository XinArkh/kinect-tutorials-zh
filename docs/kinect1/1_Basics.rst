Kinect 基础
==============


:目标: 学习如何初始化一台 Kinect，并从中获取 RGB 类型的数据。

:源码: `点此查看 <https://github.com/XinArkh/kinect-tutorials-zh/tree/master/src/kinect1/1_Basics>`_    :download:`1_Basics.zip <../../src/kinect1/1_Basics.zip>`


概述
-------

我们有两段与 Kinect 相关的代码，下文将详细讨论这些内容，然后对展示部分的代码做一个简要介绍。


包含文件，常量和全局变量
-----------------------------


包含文件
+++++++++++

大多数的文件名就解释清楚了它自己。使用 Kinect 基本都会用到三个头文件：\ ``NuiApi.h``\ ，\ ``NuiImageCamera.h``\ 和\ ``NuiSensor.h``\ 。

为了让 Kinect 的包含文件正常工作，你还需要在前面添加\ ``Ole2.h``\ 和\ ``Windows.h``\ 。同时也不要忘了添加可视化相关的包含文件。

.. code:: cpp

    // GLUT
    #include <Windows.h>
    #include <Ole2.h>

    #include <gl/GL.h>
    #include <gl/GLU.h>
    #include <gl/glut.h>

    #include <NuiApi.h>
    #include <NuiImageCamera.h>
    #include <NuiSensor.h>

.. code:: cpp

    // SDL
    #include <Windows.h>
    #include <Ole2.h>

    #include <SDL_opengl.h>
    #include <SDL.h>

    #include <NuiApi.h>
    #include <NuiImageCamera.h>
    #include <NuiSensor.h>


常量和全局变量
++++++++++++++++++

我们把窗口的宽度和高度定义为 640*480，这恰好也是 Kinect 摄像头输入的图像尺寸。

注意，\ ``data``\ 数组将会保存一份我们从 Kinect 中获取的图像的副本，以便我们将其作为纹理 (texture)。有经验的 OpenGL 用户也可以使用帧缓存对象 (Frame Buffer Object) 来代替它。

.. code:: cpp

    #define width 640
    #define height 480

    // OpenGL Variables
    GLuint textureId;              // ID of the texture to contain Kinect RGB Data
    GLubyte data[width*height*4];  // BGRA array containing the texture data

    // Kinect variables
    HANDLE rgbStream;              // The identifier of the Kinect's RGB Camera
    INuiSensor* sensor;            // The kinect sensor


Kinect 代码
---------------


Kinect 初始化
+++++++++++++++


下面是我们的第一段与 Kinect 相关的代码。\ ``initKinect()``\ 函数的作用是初始化一个 Kinect 设备。它包含了两个部分：首先，我们需要找到一个已连接到电脑的 Kinect 传感器，然后将其初始化并准备从中读取数据。

.. code:: cpp

    bool initKinect() {
        // Get a working kinect sensor
        int numSensors;
        if (NuiGetSensorCount(&numSensors) < 0 || numSensors < 1) return false;
        if (NuiCreateSensorByIndex(0, &sensor) < 0) return false;

        // Initialize sensor
        sensor->NuiInitialize(NUI_INITIALIZE_FLAG_USES_DEPTH | NUI_INITIALIZE_FLAG_USES_COLOR);
        sensor->NuiImageStreamOpen(
            NUI_IMAGE_TYPE_COLOR,            // Depth camera or rgb camera?
            NUI_IMAGE_RESOLUTION_640x480,    // Image resolution
            0,      // Image stream flags, e.g. near mode
            2,      // Number of frames to buffer
            NULL,   // Event handle
            &rgbStream);
        return sensor;
    }

以下是一些注意事项：

- 通常情况下，我们需要对上面所有函数的返回值都倍加小心，并且考虑到存在不止一个 Kinect 传感器的情况；但是为了简单起见，我们在这里只是尝试使用第一个连接到的 Kinect 传感器。

- \ ``NuiInitialize()``\ 方法接受一组标志 (flags)，来指定我们感兴趣的传感器特性。在本例中我们选择颜色和深度相机输入；此外，还有音频输入和骨骼关节点输入等选项。更多细节请参考\ `官方 API <http://msdn.microsoft.com/en-us/library/hh855368#NUI_INITIALIZE>`_\ 。

- \ ``NuiImageStreamOpen()``\ 方法有一些迷惑性。它初始化了一个\ **句柄** (\ ``HANDLE``\ )，我们稍后可以用它来获取图像帧。这个函数可以用来设置 RGB 彩色图像流或深度图像流，这取决于函数的第一个参数。现在你可以暂时忽略第 3 和第 5 个参数。设置分辨率为 640*480，缓冲区大小为一个整数。最后一个参数是一个指向句柄的指针，我们将用它来获取图像帧。更多细节请参考\ `官方文档 <http://msdn.microsoft.com/en-us/library/nuiimagecamera.nuiimagestreamopen>`_\ 。

.. note::

    **译者注**：在这里忍不住要对微软说句脏话，旧版 Kinect 下架之后连文档也清理得一干二净。目前的官网的过期版本存档中保存的是 `Kinect for Windows SDK 2.0 版本 <https://docs.microsoft.com/en-us/previous-versions/windows/kinect/dn799271(v=ieb.10)>`_\ ，SDK v1 部分已经被完全移除，所以\ **上面提供的两个官网链接其实都是打不开的**\ 。

    译者经过多方搜索，在这里提供一个聊胜于无的 SDK v1 文档查阅途径，即通过网页快照存档网站 Wayback Machine 来查阅\ `官方文档的历史快照 <https://web.archive.org/web/20130906183129/http://msdn.microsoft.com/en-us/library/hh855347.aspx>`_\ 。

    举个例子，上面提到的打不开的两个页面，在网页快照中分别如下：

    - \ ``NuiInitialize()``\ 方法\ `官方 API 的网页快照 <https://web.archive.org/web/20120915103909/http://msdn.microsoft.com/en-us/library/hh855368#NUI_INITIALIZE>`_\ 
    - \ ``NuiImageStreamOpen()``\ 方法\ `官方文档的网页快照 <https://web.archive.org/web/20120528191325/http://msdn.microsoft.com/en-us/library/nuiimagecamera.nuiimagestreamopen>`_\ 
    
    几个时间节点：

    - 微软官网最后发布的一代 Kinect SDK 版本号为 1.8.0.595，发布日期为 2013 年 9 月 13 日，在此之后文档更新应基本停止。
    - 2014 年 10 月微软发布第二代 Kinect for Windows。
    - 微软官网大约在 17 年至 18 年前后彻底撤掉了 Kinect SDK v1 文档的存档，网页快照不再记录。

    另外还需注意，网页快照的抓取时间是有一定随机性的，有时一些页面可以查看，但另一些页面可能会报错，对于报错的情况，可以换一个时间节点查看。


从 Kinect 中获取 RGB 帧
++++++++++++++++++++++++++++++

要从传感器获取帧，我们必须抓取并锁定它，这样在读取时它就不会损坏。

.. code:: cpp

    void getKinectData(GLubyte* dest) {
        NUI_IMAGE_FRAME imageFrame;
        NUI_LOCKED_RECT LockedRect;
        if (sensor->NuiImageStreamGetNextFrame(rgbStream, 0, &imageFrame) < 0) return;
        INuiFrameTexture* texture = imageFrame.pFrameTexture;
        texture->LockRect(0, &LockedRect, NULL, 0);

在这段代码中有三种类型：\ ``NUI_IMAGE_FRAME``\ 是一个包含了所有关于该帧的元数据的结构——编号、分辨率等；\ ``NUI_LOCKED_RECT``\ 包含一个指向实际数据的指针；\ ``INuiFrameTexture``\ 管理帧数据。首先，我们从前面初始化的句柄中获取一个\ ``NUI_IMAGE_FRAME``\ ，然后我们得到一个\ ``INuiFrameTexture``\ ，这样我们就可以使用\ ``NUI_LOCKED_RECT``\ 从它里面获取像素数据。

现在，我们可以将数据复制到我们自己设定的内存位置。\ ``LockedRect``\ 的\ ``Pitch``\ 方法返回的是帧的每一行中有多少字节；对该值进行简单的检查可以确保该帧不是空的。

.. code:: cpp

    if (LockedRect.Pitch != 0)
    {
        const BYTE* curr = (const BYTE*) LockedRect.pBits;
        const BYTE* dataEnd = curr + (width*height)*4;

        while (curr < dataEnd) {
            *dest++ = *curr++;
        }
    }

Kinect 数据是 BGRA 格式的，所以我们可以直接将其复制到我们的缓冲区中，作为 OpenGL 纹理使用。

最后，我们必须释放数据帧，这样 Kinect 才能再次使用它。

.. code:: cpp

        texture->UnlockRect(0);
        sensor->NuiImageStreamReleaseFrame(rgbStream, &imageFrame);
    }

以下是一些注意事项：

- 与前面相同，我们仍然没有检查所有的返回代码。在你的程序中，安全起见可以进一步完善这一部分。

- 如果你调用这个更新函数太快了，那么 Kinect 的更新速率可能会跟不上。在这种情况下，\ ``NuiImageStreamGetNextFrame()``\ 将会返回一个负值。\ ``NuiImageStreamGetNextFrame()``\ 的第二个参数指定了在返回失败之前等待新帧的时长（单位为毫秒）。

- 需要重申的是，工作流程遵循以下顺序：
    #. 获取一个帧：\ ``sensor->NuiImageStreamGetNextFrame()``\ 。第一个参数是我们前面初始化的句柄，最后一个参数是指向将要接收帧数据的\ ``NUI_IMAGE_FRAME``\ 结构的指针。第二个参数允许你设定一定时间等待一个新的帧（见上文）。
    #. 锁定像素数据：\ ``imageFrame.pFrameTexture->LockRect()``\ 。第二个参数是指向\ ``NUI_LOCKED_RECT``\ 结构的指针；除此之外，所有的参数必须是 0 或 *NULL*。
    #. （借助\ ``LockedRect.pBits``\ ）复制数据。
    #. 解锁像素数据：\ ``imageFrame.pFrameTexture->UnlockRect()``\ 。参数必须为 0。
    #. 释放帧：\ ``sensor->NuiImageStreamReleaseFrame()``\ 。第一个参数是图像流的句柄，第二个参数是指向我们要释放的图像帧的指针。

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