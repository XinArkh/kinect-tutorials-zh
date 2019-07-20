Kinect 骨骼追踪
===================


:目标: 学习如何从 Kinect 获取骨骼 (skeleton) 跟踪数据，尤其是关节点 (joints) 位置。

:源码: `点此查看 <https://github.com/XinArkh/kinect-tutorials-zh/tree/master/src/kinect2/4_SkeletalTracking>`_    :download:`4_SkeletalTracking.zip <../../src/kinect2/4_SkeletalTracking.zip>`


概述
-------

本章教程相当简单，展示了如何获取在 Kinect 视角下的基本人体信息。我们将会展示如何提取身体关节的 3D 位置，这样的信息经过进一步处理，可以实现从简单的绘制骨骼、到复杂的姿势识别在内的各种事情。

为此，我们将从我们在\ :ref:`上一章 <PointCloud-2>`\ 中创建的框架开始，并添加一些内容。


Kinect 代码
--------------


全局变量
+++++++++++

我们将维护一个布尔值作为全局变量，它告诉我们是否看到了一个物体（从而判断是否要绘制手臂）；还要维护一个数组，保存上次看到的物体中所有关节点。

.. code:: cpp

    // Body tracking variables
    BOOLEAN tracked;                            // Whether we see a body
    Joint joints[JointType_Count];              // List of joints in the tracked body

关节点数组的每一个元素都是一个\ ``Joint``\ 结构体，与被跟踪的人体的每个关节点相对应。\ `这个文档 <https://msdn.microsoft.com/en-us/library/microsoft.kinect.kinect.jointtype.aspx>`_\ 列出了被 Kinect 跟踪的\ ``JointType_Count = 25``\ 个关节点。例如，通过查看\ ``joints[JointType_ElbowRight].Position``\ ，我们可以获得用户右肘的位置。


Kinect 初始化
++++++++++++++++

这里唯一的新功能是：当我们打开\ ``MultiSourceFrameReader``\ 时，我们发出人体跟踪数据的请求。

.. code: cpp

    IKinectSensor* sensor;             // Kinect sensor
    IMultiSourceFrameReader* reader;   // Kinect data source
    ICoordinateMapper* mapper;         // Converts between depth, color, and 3d coordinates

    bool initKinect() {
        if (FAILED(GetDefaultKinectSensor(&sensor))) {
            return false;
        }
        if (sensor) {
            sensor->get_CoordinateMapper(&mapper);

            sensor->Open();
            sensor->OpenMultiSourceFrameReader(
                      FrameSourceTypes::FrameSourceTypes_Depth
                    | FrameSourceTypes::FrameSourceTypes_Color
                    | FrameSourceTypes::FrameSourceTypes_Body,
                &reader);
            return reader;
        } else {
            return false;
        }
    }

默认情况下，Kinect 希望人们能够面对传感器站立，以便更好地跟踪。在函数\ ``NuiSkeletonTrackingEnable()``\ 中，通过设置第二个参数，也可以指示设备跟踪对面坐着的人（比如坐在沙发上)。


从 Kinect 中获取关节数据
++++++++++++++++++++++++++++++

在我们的新\ ``getBodyData()``\ 函数中，我们仍然像以前一样从\ ``MultiSourceFrame``\ 中请求帧。

.. code:: cpp

    void getBodyData(IMultiSourceFrame* frame) {
        IBodyFrame* bodyframe;
        IBodyFrameReference* frameref = NULL;
        frame->get_BodyFrameReference(&frameref);
        frameref->AcquireFrame(&bodyframe);
        if (frameref) frameref->Release();

        if (!bodyframe) return;

        // ------ NEW CODE ------
        IBody* body[BODY_COUNT];
        bodyframe->GetAndRefreshBodyData(BODY_COUNT, body);
        for (int i = 0; i < BODY_COUNT; i++) {
            body[i]->get_IsTracked(&tracked);
            if (tracked) {
                body[i]->GetJoints(JointType_Count, joints);
                break;
            }
        }
        // ------ END NEW CODE ------

        if (bodyframe) bodyframe->Release();
    }

Kinect 可以同时追踪最多\ ``BODY_COUNT``\ 个用户（在 SDK 中，\ ``BODY_COUNT == 6``\ ）。我们使用\ ``IBodyFrame::GetAndRefreshBodyData()``\ 函数填充\ ``IBody``\ 指针数组。注意，每个\ ``IBody``\ 不一定指向 Kinect 可以看到的一个实际的人，第一个被跟踪的人体也不一定是数组的第一个元素。因此，我们需要对每一个元素检查它是否是一个被跟踪的人体。

为了简单起见，我们每次只处理这个应用程序中的一个人。因此，我们检查返回的每一个人体是否都指向一个被跟踪的人，并在找到一个后跳出循环。我们使用\ ``IBody::GetJoints()``\ 函数，用人体中所有的关节点位置填充\ ``joints``\ 数组。我们还将跟踪状态保存在全局跟踪变量中。

注意：调用\ ``IBodyFrame::GetAndRefreshBodyData()``\ 时始终以\ ``BODY_COUNT``\ 作为第一个参数；调用\ ``IBody::GetJoint``\时始终以\ ``JointType_Count``\ 作为第一个参数。


OpenGL 显示
---------------

我们用关节点数组中的坐标绘制一些简单的线条，来显示人体的上肢。也就是说，我们要从右肩到右肘画一条线，然后从右肘到右手腕画一条线；左边也是一样的道理。当然，只有在 Kinect 检测到人的情况下才需要画线，所以我们要先检查全局布尔变量\ ``tracked``\ 。

在\ ``Joint``\ 结构体中的\ ``Position``\域里面，关节点坐标以 3D \ ``CameraSpacePoint``\ 的形式表示。

.. code:: cpp

    void drawKinectData() {
        // ...
        if (tracked) {
            // Draw some arms
            const CameraSpacePoint& lh = joints[JointType_WristLeft].Position;
            const CameraSpacePoint& le = joints[JointType_ElbowLeft].Position;;
            const CameraSpacePoint& ls = joints[JointType_ShoulderLeft].Position;;
            const CameraSpacePoint& rh = joints[JointType_WristRight].Position;;
            const CameraSpacePoint& re = joints[JointType_ElbowRight].Position;;
            const CameraSpacePoint& rs = joints[JointType_ShoulderRight].Position;;
            glBegin(GL_LINES);
            glColor3f(1.f, 0.f, 0.f);
            // lower left arm
            glVertex3f(lh.X, lh.Y, lh.Z);
            glVertex3f(le.X, le.Y, le.Z);
            // upper left arm
            glVertex3f(le.X, le.Y, le.Z);
            glVertex3f(ls.X, ls.Y, ls.Z);
            // lower right arm
            glVertex3f(rh.X, rh.Y, rh.Z);
            glVertex3f(re.X, re.Y, re.Z);
            // upper right arm
            glVertex3f(re.X, re.Y, re.Z);
            glVertex3f(rs.X, rs.Y, rs.Z);
            glEnd();
        }
    }

结束！构建并运行，确保你的 Kinect 已经插入。你应该会看到一个包含 Kinect 所拍摄的旋转的彩色点云的（视频流）窗口，当 Kinect 捕捉到人体时，则绘制红线来展示这个人的上肢姿态。