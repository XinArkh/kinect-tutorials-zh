******************
Kinect 中文教程
******************


:作者: `Edward Zhang <http://ed.ilogues.com/>`_

:译者: `Wu Hsin <https://github.com/XinArkh>`_

:授权: 该文档遵循 `CC BY-SA 4.0 <http://creativecommons.org/licenses/by-sa/4.0/>`_ 许可协议。

本教程面向 Kinect 的初学者，使用 **C++ 语言**\ 构建了 Kinect 的一些入门案例，并使用 OpenGL (GLUT 或 SDL) 来实现可视化。教程分为相互平行的两部分，分别是：

- 基于 **Kinect v1.8 SDK** 的基础操作，适用于基于结构光的 Kinect 传感器，包括 **XBox 360 Kinect** 和 **Kinect for Windows**\ 。

.. image:: ./images/kinectv1.png

- 基于 **Kinect v2 SDK** 的基础操作，适用于采用 Time of Flight (TOF) 深度感知方式的 Kinect 传感器，即 **XBox One Kinect**\ 。

.. image:: ./images/kinectv2.jpg

基本原则
===========
在把玩了一阵子 Kinect 之后，作者发现很多资料都残缺不全。作为一份（给后来者）的参考，本教程应运而生。以下是本教程的指导原则：

- **如无必要，勿费口舌** - 本教程主要侧重于如何使用各种 API。对于其余的窗口创建和显示部分，将会在简单概括后略过。如果你想要深入学习 OpenGL、C++ 等知识，可以去寻找这些方面更加专业的教程。这里不会同时精讲三种 API 和一种语言，这只会让人更加糊涂。
- **精炼代码，主次分明** - 教程中的代码刚好能够让它工作，其余部分基本会忽略。为了避免头文件过多带来的混乱，一部分代码结构会被牺牲，这样你可以专注于相关的功能。

.. note::
   
   该教程是在原有英文教程的基础上，由译者个人维护的\ **非官方**\ 中文教程。\ `点击此处 <https://homes.cs.washington.edu/~edzhang/tutorials/index.html>`_\ 可以查看原版英文教程。


.. toctree::
   :maxdepth: 2
   :caption: Kinect v1.8 C++ SDK 基础教程
   :numbered:

   kinect1/0_Setup
   kinect1/1_Basics
   kinect1/2_Depth
   kinect1/3_PointCloud
   kinect1/4_SkeletalTracking


.. toctree::
   :maxdepth: 2
   :caption: Kinect v2 C++ SDK 基础教程
   :numbered:

   kinect2/0_Setup
   kinect2/1_Basics
   kinect2/2_Depth
   kinect2/3_PointCloud
   kinect2/4_SkeletalTracking


索引
=====

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`
