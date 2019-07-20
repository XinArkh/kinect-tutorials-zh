Kinect 深度数据
=====================

:目标: 学习如何从一台 Kinect 中获取深度数据，并了解数据的格式是什么样的。

:源码: `点此查看 <https://github.com/XinArkh/roslibpy-docs-zh>`_    :download:`1_Basics.zip <../../src/kinect1/2_Depth.zip>`


概述
------

本章教程非常短——与前一章获取 RGB 数据的代码相比只有两处重要的改动。

另外，SDL 和 GLUT 组件被重构放置在了不同的文件中，以便将注意力集中在 Kinect 的细节上。


Kinect 代码
----------------


Kinect 初始化
++++++++++++++++

要从 Kinect 中获取深度数据，只需要改变一下\ ``NuiImageStreamOpen()``\ 中的参数。第一个参数是\ ``NUI_IMAGE_TYPE_DEPTH``\ ，它告诉 Kinect 我们需要深度图像而不是 RGB 图像。（为了逻辑上的清晰，我们也对应修改了句柄的名称，来反映这一点。）

在使用 Kinect for Windows 时，与 XBox Kinect 相比还多了一个\ **近距离模式**\ 。顾名思义，近距离模式允许 Kinect 设备在更近的距离下工作（约 50cm 到 200cm），而没有近距离模式的设备采集距离一般为 80cm 到 400cm。

要激活近距离模式，将第三个参数设置为\ ``NUI_IMAGE_STREAM_FLAG_ENABLE_NEAR_MODE``\ 即可。

.. code:: cpp

    // NEW VARIABLE
    HANDLE depthStream;

    bool initKinect() {
        // Get a working kinect sensor
        int numSensors;
        if (NuiGetSensorCount(&numSensors) < 0 || numSensors < 1) return false;
        if (NuiCreateSensorByIndex(0, &sensor) < 0) return false;

        // Initialize sensor
        sensor->NuiInitialize(NUI_INITIALIZE_FLAG_USES_DEPTH | NUI_INITIALIZE_FLAG_USES_COLOR);

            // --------------- START CHANGED CODE -----------------
        sensor->NuiImageStreamOpen(
            NUI_IMAGE_TYPE_DEPTH,                     // Depth camera or rgb camera?
            NUI_IMAGE_RESOLUTION_640x480,             // Image resolution
            NUI_IMAGE_STREAM_FLAG_ENABLE_NEAR_MODE,   // Image stream flags, e.g. near mode
            2,      // Number of frames to buffer
            NULL,   // Event handle
            &depthStream);
            // --------------- END CHANGED CODE -----------------
        return sensor;
    }

有关近距离模式的更多信息，请参阅\ `此官方博客文章 <https://blogs.msdn.microsoft.com/kinectforwindows/2012/01/20/near-mode-what-it-is-and-isnt/>`_\ 。


从 Kinect 中获取深度数据帧
++++++++++++++++++++++++++++++++

我们将以灰度图像的形式显示 Kinect 的深度图像。每个像素的值将是该像素点到 Kinect 的距离 (mm) 除以 256 的余数。

这里唯一要注意的事情是\ ``NuiDepthPixelToDepth()``\ 函数——实际深度图中的每个像素都含有深度信息和玩家信息（即哪个玩家是与这个像素关联的）。调用此函数返回的是该像素处的深度 (mm)。

深度数据是 16 位的，所以我们使用 \ ``USHORT``\ 类型来读取它。

.. code:: cpp

            const USHORT* curr = (const USHORT*) LockedRect.pBits;
            const USHORT* dataEnd = curr + (width*height);

            while (curr < dataEnd) {
                // Get depth in millimeters
                USHORT depth = NuiDepthPixelToDepth(*curr++);

                // Draw a grayscale image of the depth:
                // B,G,R are all set to depth%256, alpha set to 1.
                for (int i = 0; i < 3; ++i)
                    *dest++ = (BYTE) depth%256;
                *dest++ = 0xff;
            }

以上就是所有 Kinect 部分的代码！剩下的就是如何把它搬上屏幕。

显示框架
-----------

这里重构了代码，将 GLUT 和 SDL 部分打包进了\ ``init()``\ 函数、\ ``draw()``\ 函数和\ ``execute()``\ 函数中。

要使用它们，只需要在\ ``main.cpp``\ 的最上方包含\ ``glut.h``\ 或\ ``sdl.h``\ 。确保项目属性中有正确的包含文件路径和链接文件路径。

结束！构建并运行，确保你的 Kinect 已经插入。你应该会看到一个包含 Kinect 所拍摄内容的灰度视频流窗口。