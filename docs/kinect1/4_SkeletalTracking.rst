Kinect 骨骼追踪
===================


:目标: 学习如何从 Kinect 获取骨骼 (skeleton) 跟踪数据，尤其是关节点 (joints) 位置。

:源码: `点此查看 <https://github.com/XinArkh/kinect-tutorials-zh/tree/master/src/kinect1/4_SkeletalTracking>`_    :download:`4_SkeletalTracking.zip <../../src/kinect1/4_SkeletalTracking.zip>`


概述
-------

本章教程相当简单，展示了如何获取在 Kinect 视角下的基本人体信息。我们将会展示如何提取身体关节的 3D 位置，这样的信息经过进一步处理，可以实现从简单的绘制骨骼、到复杂的姿势识别在内的各种事情。

为此，我们将从我们在\ :ref:`上一章 <PointCloud>`\ 中创建的框架开始，并添加一些内容。


Kinect 代码
--------------


全局变量
+++++++++++

我们将维护一个全局变量，来保存最近捕捉到的身体的所有关节点。

.. code:: cpp

    // Body tracking variables
    Vector4 skeletonPosition[NUI_SKELETON_POSITION_COUNT];

总共有\ ``NUI_SKELETON_POSITION_COUNT = 20``\ 个关节点会被 Kinect 追踪，具体可以看\ `这里 <https://msdn.microsoft.com/en-us/library/nuiskeleton.nui_skeleton_position_index.aspx>`_\ 。每个关节点用一个\ ``Vector4``\ 类型来描述其在相机坐标系下的 3D 位置。例如，要看右肘的位置，可以使用\ ``skeletonPosition[NUI_SKELETON_POSITION_ELBOW_RIGHT]``\ 。

.. note::

    **译者注**：上面的链接已失效，替代的网页快照\ `可以看这里 <https://web.archive.org/web/20130423073048/http://msdn.microsoft.com/en-us/library/nuiskeleton.nui_skeleton_position_index.aspx>`_\ 。


Kinect 初始化
++++++++++++++++

我们在 Kinect 初始化中添加了两个新的部分。首先，我们在初始化 Kinect 时请求了骨骼追踪数据，然后，我们启用了追踪。

.. code: cpp

    bool initKinect() {
        // ...
        sensor->NuiInitialize(
                NUI_INITIALIZE_FLAG_USES_DEPTH_AND_PLAYER_INDEX
              | NUI_INITIALIZE_FLAG_USES_COLOR
              | NUI_INITIALIZE_FLAG_USES_SKELETON);


        sensor->NuiSkeletonTrackingEnable(
            NULL,
            0     // NUI_SKELETON_TRACKING_FLAG_ENABLE_SEATED_SUPPORT for only upper body
        );
        // ...
    }

默认情况下，Kinect 希望人们能够面对传感器站立，以便更好地跟踪。在函数\ ``NuiSkeletonTrackingEnable()``\ 中，通过设置第二个参数，也可以指示设备跟踪对面坐着的人（比如坐在沙发上)。


从 Kinect 中获取关节数据
++++++++++++++++++++++++++++++

获取骨骼数据要比获取彩色数据和深度数据更简单——它没有锁定或其它繁琐的操作。我们只需从传感器获取一串\ ``NUI_SKELETON_FRAME``\ 类型的数据。关节点追踪的噪声可能会很大，所以为了降低关节点位置随时间的抖动，我们还调用了一个内置的平滑函数。

.. code:: cpp

    void getSkeletalData() {
        NUI_SKELETON_FRAME skeletonFrame = {0};
        if (sensor->NuiSkeletonGetNextFrame(0, &skeletonFrame) >= 0) {
            sensor->NuiTransformSmooth(&skeletonFrame, NULL);
            // Process skeletal frame (see below)...
        }
    }

Kinect 可以同时追踪最多\ ``NUI_SKELETON_COUNT``\ 个用户（在 SDK 中，\ ``NUI_SKELETON_COUNT == 6``\ ）。骨骼数据结构\ ``NUI_SKELETON_DATA``\ 可以在帧的\ ``SkeletonData``\ 数组字段中访问。注意，每个骨骼不一定指向 Kinect 可以看到的实际的人，第一个被跟踪的身体也不一定是数组中的第一个元素。因此，我们需要检查数组中的每个元素是否是被跟踪的身体。通过检查骨骼的追踪状态来做这件事。

.. code:: cpp

            // Loop over all sensed skeletons
            for (int z = 0; z < NUI_SKELETON_COUNT; ++z) {
                const NUI_SKELETON_DATA& skeleton = skeletonFrame.SkeletonData[z];
                // Check the state of the skeleton
                if (skeleton.eTrackingState == NUI_SKELETON_TRACKED) {
                    // Get skeleton data (see below)...
                }
            }

一旦我们有了一个有效的跟踪骨骼，我们就可以将所有的关节点数据复制到我们的关节点位置数组中。鉴于 Kinect 可能会跟丢一部分关节点（比如用户的手臂藏到了背后，或遇到其它遮挡），我们还单独检查每个关节的追踪状态。我们使用\ ``Vector4``\ 关节位置的 *w* 坐标来记录它是否是一个有效的跟踪关节。

.. code:: cpp

                // For the first tracked skeleton
                {
                    // Copy the joint positions into our array
                    for (int i = 0; i < NUI_SKELETON_POSITION_COUNT; ++i) {
                        skeletonPosition[i] = skeleton.SkeletonPositions[i];
                        if (skeleton.eSkeletonPositionTrackingState[i] == NUI_SKELETON_POSITION_NOT_TRACKED) {
                            skeletonPosition[i].w = 0;
                        }
                    }
                    return; // Only take the data for one skeleton
                }


OpenGL 显示
---------------

我们用关节点数组中的坐标绘制一些简单的线条，来显示人体的上肢。也就是说，我们要从右肩到右肘画一条线，然后从右肘到右手腕画一条线；左边也是一样的道理。当然，只有在 Kinect 检测到人的情况下才需要画线，所以我们要先检查向量的 *w* 坐标是否有效。

.. code:: cpp

    void drawKinectData() {
        // ...
        const Vector4& lh = skeletonPosition[NUI_SKELETON_POSITION_HAND_LEFT];
        const Vector4& le = skeletonPosition[NUI_SKELETON_POSITION_ELBOW_LEFT];
        const Vector4& ls = skeletonPosition[NUI_SKELETON_POSITION_SHOULDER_LEFT];
        const Vector4& rh = skeletonPosition[NUI_SKELETON_POSITION_HAND_RIGHT];
        const Vector4& re = skeletonPosition[NUI_SKELETON_POSITION_ELBOW_RIGHT];
        const Vector4& rs = skeletonPosition[NUI_SKELETON_POSITION_SHOULDER_RIGHT];
        glBegin(GL_LINES);
            glColor3f(1.f, 0.f, 0.f);
            if (lh.w > 0 && le.w > 0 && ls.w > 0) {
                // lower left arm
                glVertex3f(lh.x, lh.y, lh.z);
                glVertex3f(le.x, le.y, le.z);
                // upper left arm
                glVertex3f(le.x, le.y, le.z);
                glVertex3f(ls.x, ls.y, ls.z);
            }
            if (rh.w > 0 && re.w > 0 && rs.w > 0) {
                // lower right arm
                glVertex3f(rh.x, rh.y, rh.z);
                glVertex3f(re.x, re.y, re.z);
                // upper right arm
                glVertex3f(re.x, re.y, re.z);
                glVertex3f(rs.x, rs.y, rs.z);
            }
        glEnd();
    }

结束！构建并运行，确保你的 Kinect 已经插入。你应该会看到一个包含 Kinect 所拍摄的旋转的彩色点云的（视频流）窗口，当 Kinect 捕捉到人体时，则绘制红线来展示这个人的上肢姿态。