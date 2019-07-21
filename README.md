# kinect-tutorials-zh

本 Kinect 中文教程是对[这套英文教程](https://homes.cs.washington.edu/~edzhang/tutorials/index.html)的**非官方**中文翻译，并对一些内容做了修缮和补充。

本教程分为 Kinect v1 部分和 Kinect v2 部分，可根据 Kinect 硬件型号自行选择阅读。

教程面向 Kinect 的初学者，使用**C++**语言构建了 Kinect 的一些入门案例，并使用 OpenGL (GLUT 或 SDL) 来实现可视化。

## 本地构建

教程已托管至Read the Docs，可以随时[在线阅读](https://kinect-tutorials-zh.readthedocs.io/zh_CN/latest/index.html)。如果需要在本地构建离线文档，可按照以下方式操作：

1. 将该仓库 clone 至本地，然后进入项目根目录。

2. 按照`requirements.txt`中所列举的环境要求自行安装所需要的 Python 依赖库。

3. 通过`make`命令构建文档。

   **Linux 下的 make 命令：**

```shell
make html
```

**Windows 下的 make 命令：**

```shell
./make.bat html
```

构建好的 html 文档位于`root/build/html`。

