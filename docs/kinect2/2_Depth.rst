Kinect 深度数据
=====================

:目标: 学习如何从一台 Kinect 中获取深度数据，并了解数据的格式是什么样的。

:源码: `点此查看 <https://github.com/XinArkh/kinect-tutorials-zh/tree/master/src/kinect2/2_Depth>`_    :download:`2_Depth.zip <../../src/kinect2/2_Depth.zip>`


概述
------

本章教程非常短——与前一章获取 RGB 数据的代码相比只有两处重要的改动。

另外，为了方便将注意力集中在 Kinect 的细节上，我们重构了 SDL 和 GLUT 组件，将它们放置在了不同的文件中。


Kinect 代码
----------------


Kinect 初始化
++++++++++++++++

要从 Kinect 中获取深度数据，只需要改变一下数据帧输入源、数据帧读取器和数据帧的类型。同时还要注意，深度相机的分辨率是512*424，与彩色相机 (1920*1080) 是不同的。

.. code:: cpp

    IDepthFrameReader* reader;     // Kinect depth data source

    bool initKinect() {
        if (FAILED(GetDefaultKinectSensor(&sensor))) {
            return false;
        }
        if (sensor) {
            sensor->Open();
            IDepthFrameSource* framesource = NULL;
            sensor->get_DepthFrameSource(&framesource);
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


从 Kinect 中获取深度数据帧
++++++++++++++++++++++++++++++++

我们将以灰度图像的形式显示 Kinect 的深度图像。每个像素的值将是该像素点到 Kinect 的距离 (mm) 除以 256 的余数。

.. code:: cpp

    void getKinectData(GLubyte* dest) {
        IDepthFrame* frame = NULL;
        if (SUCCEEDED(reader->AcquireLatestFrame(&frame))) {
            unsigned int sz;
            unsigned short* buf;
            frame->AccessUnderlyingBuffer(&sz, &buf);

            const unsigned short* curr = (const unsigned short*)buf;
            const unsigned short* dataEnd = curr + (width*height);

            while (curr < dataEnd) {
                // Get depth in millimeters
                unsigned short depth = (*curr++);

                // Draw a grayscale image of the depth:
                // B,G,R are all set to depth%256, alpha set to 1.
                for (int i = 0; i < 3; ++i)
                    *dest++ = (BYTE)depth % 256;
                *dest++ = 0xff;
            }
        }
        if (frame) frame->Release();
    }

注意，与彩色帧不同，我们只想访问来自帧的原始数据。为此，我们使用\ ``frame->AccessUnderlyingBuffer``\ 来获得指向数据的指针。

以上就是所有 Kinect 部分的代码！剩下的就是如何把它搬上屏幕。

显示框架
-----------

这里重构了代码，将 GLUT 和 SDL 部分打包进了\ ``init()``\ 函数、\ ``draw()``\ 函数和\ ``execute()``\ 函数中。

要使用它们，只需要在\ ``main.cpp``\ 的最上方包含\ ``glut.h``\ 或\ ``sdl.h``\ 。确保项目属性中有正确的包含文件路径和链接文件路径。

结束！构建并运行，确保你的 Kinect 已经插入。你应该会看到一个包含 Kinect 所拍摄内容的灰度视频流窗口。