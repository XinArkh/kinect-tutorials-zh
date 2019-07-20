Kinect 点云
==============


:目标: 学习如何对齐彩色和深度图像，以获得彩色点云。

:源码: `点此查看 <https://github.com/XinArkh/roslibpy-docs-zh>`_    :download:`1_Basics.zip <../../src/kinect1/3_PointCloud.zip>`


概述
-------

在本章教程中，我们想采取一些新步骤。最有趣的部分就是，我们现在正在处理 3D 数据！但是，创建一个交互式系统的工作量对我们的教程来说有些多了，所以我们只是创建一个简单地旋转点云。本章教程分为三部分：首先，我们简单讨论一下为什么点云可能要比你想像的更难；其次，我们将展示如何使用 Kinect SDK 获取正确的数据；最后，我们将会展示一些可以降低图像展示难度的 OpenGL技巧。


深度和 RGB 坐标系
--------------------


Kinect 坐标系
++++++++++++++++

Kinect 使用以 Kinect 为中心的笛卡尔坐标系统，\ *Y* 轴朝上，\*Z* 轴朝前，\ *X* 轴朝左。

.. image:: ../images/3/3_0.png


对齐
++++++

最简单的生成点云的方法是直接重叠深度和彩色图像，使深度像素 *(x, y)* 与图像像素 *(x, y)* 一一匹配。然而，这样生成的深度图像质量非常低，对象的边界和颜色无法对齐。这是因为 RGB 相机和深度相机位于 Kinect 上面的不同位置；显然，它们看到的东西并不一致！通常，我们需要对两个相机进行某种对齐操作（正式术语是「注册」， *registration*\ ），来从一个坐标空间映射到另一个坐标空间。幸运的是，微软粑粑已经帮我们做好这件事了，我们唯一需要做的就是调用正确的函数。

**直接重叠 RGB 和深度**

.. image:: ../images/3/3_1.png

**“注册”后的 RGB 和深度**

.. image:: ../images/3/3_1_2.png

要注意，计算机视觉和机器人学方面的研究人员并不喜欢这个内置注册函数的输出质量，所以他们经常用 `OpenCV <http://opencv.org/>`_ 之类的工具手动进行校正。


Kinect 代码
--------------

以下很多代码仅仅是对前面两章教程的综合。


Kinect 初始化
+++++++++++++++++

初始化部分没有新内容。我们需要两个图像流，一个用于深度图像，一个用于彩色图像。

.. code:: cpp

    HANDLE rgbStream;
    HANDLE depthStream;
    INuiSensor* sensor;

    bool initKinect() {
        // Get a working kinect sensor
        int numSensors;
        if (NuiGetSensorCount(&numSensors) < 0 || numSensors < 1) return false;
        if (NuiCreateSensorByIndex(0, &sensor) < 0) return false;

        // Initialize sensor
        sensor->NuiInitialize(NUI_INITIALIZE_FLAG_USES_DEPTH | NUI_INITIALIZE_FLAG_USES_COLOR);

        sensor->NuiImageStreamOpen(
            NUI_IMAGE_TYPE_DEPTH,                     // Depth camera or rgb camera?
            NUI_IMAGE_RESOLUTION_640x480,             // Image resolution
            0,      // Image stream flags, e.g. near mode
            2,      // Number of frames to buffer
            NULL,   // Event handle
            &depthStream);
        sensor->NuiImageStreamOpen(
            NUI_IMAGE_TYPE_COLOR,                     // Depth camera or rgb camera?
            NUI_IMAGE_RESOLUTION_640x480,             // Image resolution
            0,      // Image stream flags, e.g. near mode
            2,      // Number of frames to buffer
            NULL,   // Event handle
            &rgbStream);
        return sensor;
    }


从 Kinect 中获取深度数据
++++++++++++++++++++++++++++++

Kinect SDK 提供了一个函数，告诉你 RGB 图像的哪个像素对应深度图像中的特定点。我们将把这个信息保存在另一个全局数组\ ``depthToRgbMap``\ 中。特别地，我们按照每一个深度像素的顺序，将对应的彩色像素的行和列（即 *x* 和 *y* 坐标）也保存下来。

现在，我们开始处理 3D 数据。我们将深度图像帧想像成空间中的一束点，而不是一个 640*480 的图像。所以在函数\ ``getDepthData()``\ 中，我们将用每个点的坐标（而不是深度）来填充我们的缓存区。这意味着对于\ ``float``\ 类型的坐标来说，我们需要填充的缓存区需要达到\ ``width*height*3*sizeof(float)``\ 的大小。

.. code:: cpp

    // Global Variables
    long depthToRgbMap[width*height*2];
    // ...
    void getDepthData(GLubyte* dest) {
    // ...
            const USHORT* curr = (const USHORT*) LockedRect.pBits;
            float* fdest = (float*) dest;
            long* depth2rgb = (long*) depthToRgbMap;
            for (int j = 0; j < height; ++j) {
                for (int i = 0; i < width; ++i) {
                    // Get depth of pixel in millimeters
                    USHORT depth = NuiDepthPixelToDepth(*curr);
                    // Store coordinates of the point corresponding to this pixel
                    Vector4 pos = NuiTransformDepthImageToSkeleton(i,j,*curr);
                    *fdest++ = pos.x/pos.w;
                    *fdest++ = pos.y/pos.w;
                    *fdest++ = pos.z/pos.w;
                    // Store the index into the color array corresponding to this pixel
                    NuiImageGetColorPixelCoordinatesFromDepthPixelAtResolution(
                        NUI_IMAGE_RESOLUTION_640x480, // color frame resolution
                        NUI_IMAGE_RESOLUTION_640x480, // depth frame resolution
                        NULL,                         // pan/zoom of color image (IGNORE THIS)
                        i, j, *curr,                  // Column, row, and depth in depth image
                        depth2rgb, depth2rgb+1        // Output: column and row (x,y) in the color image
                    );
                    depth2rgb += 2;
                    *curr++;
                }
            }
    // ...

这里有很多东西需要解释！

- \ ``Vector4``\ 是微软在齐次坐标系下的的 3D 点类型。如果你的线性代数生疏了，不用担心齐次坐标——只要把它当作一个具有 *x*\ 、\ *y*\ 、\ *z* 坐标的三维点即可。在\ `这个页面 <http://sunshine2k.blogspot.com/2011/12/reason-for-homogeneous-4d-coordinates.html>`_\ 可以找到一个简短的说明。

- \ ``NuiTransformDepthImageToSkeleton()``\ 返回某一特定深度像素的 3D 坐标，坐标系是上面提到的 Kinect 坐标系。这个函数还有一个版本，可以接受一个附加的分辨率参数。

- \ ``NuiImageGetColorPixelCoordinatesFromDepthPixelAtResolution()``\ 接受深度像素（深度图像中的行、列和深度），输出彩色图像中的行和列。\ `API 参考页面 <http://msdn.microsoft.com/en-us/library/jj663857.aspx>`_\ 见此。

.. note::

    **译者注**：同样地，上面的 API 页面已经失效，替代的网页快照\ `见此 <https://web.archive.org/web/20140425111041/http://msdn.microsoft.com/en-us/library/jj663857.aspx>`_\ 。

