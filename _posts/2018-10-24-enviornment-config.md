---
title: 一步一步搭建OpenCV+MinGW32+Qt+CLion环境
key: 20181025
tags:
- OpenCV Qt MinGW CLion
---

## 一、前言
<!--more-->
&nbsp;&nbsp;&nbsp;&nbsp;OpenCV是做图像方面研究的必备工具，C++下常用的IDE有Visual Studio、CLion、Qt Creator在图形界面开发方面，Qt Creator无疑是最好的选择。CLion是JetBrains旗下的一款C++IDE，目前只提供30天的试用，不过如果你是学生的话可以去申请学生包，这样就可以免费使用它们家旗下的大多数产品了，包括我觉得最好用的PyCharm。Visual Studio是微软旗下的IDE，但由于他体积庞大，我个人是不怎么喜欢用，除非工作需要开发Windows下的应用程序，需要用到VS的编译器，除此之外我还是喜欢Qt+MinGW或者CLion+MinGW作为工作环境。

&nbsp;&nbsp;&nbsp;&nbsp;言归正传，笔者总结了网上一些博客，自己手动编译OpenCV源码，并在Qt与CLion中配置好了OpenCV库，在这里进行分享，有价值的参考博客我在参考文献中列出，在此十分感谢以下博主的无私分享。

## 二、软件环境
- Windows-10-64bit
- [Qt-5.9.3](https://download.qt.io/official_releases/qt/5.9/5.9.3/)
- [MinGW-5.3.0-32bit](http://www.mingw.org/)（非必须）
- [CMake-3.9.2](https://cmake.org/)
- [OpenCV-3.3.1 / 3.4.1(适用)](https://sourceforge.net/projects/opencvlibrary/files/opencv-win/)

## 三、软件安装
### Qt安装
&nbsp;&nbsp;&nbsp;&nbsp;在安装Qt的时候我们注意在Qt与Tool两个选项下面选择，保证我们使用的Qt是MinGW编译器的，其实这个时候不需要使用CLion或者你知道如何在CLion中配置MinGW编译器，那么就不需要下载MinGW了。

- Qt-Qt5.9-MingGW 5.3.0 32 bit
- Qt-Tools-MinGW 5.3.0

## 四、配置环境变量
&nbsp;&nbsp;&nbsp;&nbsp;Windows下环境变量的设置可以自行百度，此处为系统变量**Path**添加MinGW路径，以Qt安装在D盘为例，添加
**D:\Qt\Qt5.9.3\Tools\mingw530_32\bin**

## 五、配置CMake，生成MakeFile
首先修改**D:\OpenCV_3.3.1\opencv\sources\modules\videoio\src\cap_dshow.cpp**文件，在**#include "DShow.h"**这行的上面加以下内容：

	#ifdef _DEBUG
		#define STRSAFE_NO_DEPRECATE
	#else
		#define NO_DSHOW_STRSAFE
	#endif

打开 cmake-gui，设置源码和生成路径：

    Where is the source code: D:/OpenCV_3.3.1/opencv/sources
    Where to build the binaries: D:/OpenCV_3.3.1/opencv-build

点击 Configure，设置编译器

    Specify the generator for this project: MinGW Makefiles
    Specify native compilers
    Next
    Compilers C: D:\Qt\Qt5.9.3\Tools\mingw530_32\bin\gcc.exe
    Compilers C++: D:\Qt\Qt5.9.3\Tools\mingw530_32\bin\g++.exe
    Finish

编译配置：

    勾选 WITH_QT
    勾选 WITH_OPENGL
	勾选 ENABLE_CXX11
	不勾选 ENABLE_PRECOMPILED_HEADERS

点击 Configure，再次配置：

    不勾选 WITH_IPP # 如果有
    设置 QT_MAKE_EXECUTABLE 为 E:\Qt\Qt5.9.3\5.9.3\mingw53_32\bin\qmake.exe # 如果有
    设置 Qt5Concurrent_DIR 为 E:\Qt\Qt5.9.3\5.9.3\mingw53_32\lib\cmake\Qt5Concurrent
    设置 Qt5Core_DIR 为 E:\Qt\Qt5.9.3\5.9.3\mingw53_32\lib\cmake\Qt5Core
    设置 Qt5Gui_DIR 为 E:\Qt\Qt5.9.3\5.9.3\mingw53_32\lib\cmake\Qt5Gui
    设置 Qt5Test_DIR 为 E:\Qt\Qt5.9.3\5.9.3\mingw53_32\lib\cmake\Qt5Test
    设置 Qt5Widgets_DIR 为 E:\Qt\Qt5.9.3\5.9.3\mingw53_32\lib\cmake\Qt5Widgets
    设置 Qt5OpenGL_DIR 为 E:\Qt\Qt5.9.3\5.9.3\mingw53_32\lib\cmake\Qt5OpenGL
    设置 CMAKE_BUILD_TYPE 为 Release 或者 RelWithDebInfo #如果只编译debug或release版本，则只能在debug或release模式下使用，若想同时使用两个版本，可以分别以debug和release编译后，将两次完成后的install路径**D:\opencv-3.3.1\mingw-build\install\x86\mingw**下的bin和lib文件夹下的内容合并

点击 Generate 生成 Makefile

## 六、编译OpenCV
&nbsp;&nbsp;&nbsp;&nbsp;打开终端进行编译：（-j 是使用 8 个线程进行编译，请根据你的计算机配置合理设置线程数）

	D:
	cd D:\OpenCV_3.3.1\mingw-build
	mingw32-make -j 8
	mingw32-make install

如果 mingw32-make -j 8 遇到错误，请看参考文献中的 编译 OpenCV 常见错误，否则执行 mingw32-make install，完成安装。基本文中的错误都在第五节配置时解决了

## 七、Qt中添加OpenCV库
&nbsp;&nbsp;&nbsp;&nbsp;下面贴一下我自己的。pro文件，作为参考

	#-------------------------------------------------
	#
	# Project created by QtCreator 2018-10-25T13:29:18
	#
	#-------------------------------------------------
	
	QT       += core gui
	
	greaterThan(QT_MAJOR_VERSION, 4): QT += widgets
	
	TARGET = untitled
	TEMPLATE = app
	
	# The following define makes your compiler emit warnings if you use
	# any feature of Qt which has been marked as deprecated (the exact warnings
	# depend on your compiler). Please consult the documentation of the
	# deprecated API in order to know how to port your code away from it.
	DEFINES += QT_DEPRECATED_WARNINGS
	
	# You can also make your code fail to compile if you use deprecated APIs.
	# In order to do so, uncomment the following line.
	# You can also select to disable deprecated APIs only up to a certain version of Qt.
	#DEFINES += QT_DISABLE_DEPRECATED_BEFORE=0x060000    # disables all the APIs deprecated before Qt 6.0.0
	
	# 添加头文件包含目录
	INCLUDEPATH += D:\opencv-3.3.1\mingw-build\install\include
	# 为debug与release版本分别添加动态链接库
	win32:CONFIG(release, debug|release): LIBS += -LD:\opencv-3.3.1\mingw-build\install\x86\mingw\bin \
	                                            -llibopencv_calib3d331 \
	                                            -llibopencv_core331 \
	                                            -llibopencv_dnn331 \
	                                            -llibopencv_features2d331 \
	                                            -llibopencv_flann331 \
	                                            -llibopencv_highgui331 \
	                                            -llibopencv_imgcodecs331 \
	                                            -llibopencv_imgproc331 \
	                                            -llibopencv_ml331 \
	                                            -llibopencv_objdetect331 \
	                                            -llibopencv_photo331 \
	                                            -llibopencv_shape331 \
	                                            -llibopencv_stitching331 \
	                                            -llibopencv_superres331 \
	                                            -llibopencv_video331 \
	                                            -llibopencv_videoio331 \
	                                            -llibopencv_videostab331 \
	                                            -lopencv_ffmpeg331 \
	else:win32:CONFIG(debug, debug|release): LIBS += -LD:\opencv-3.3.1\mingw-build\install\x86\mingw\bin \
	                                            -llibopencv_calib3d331d \
	                                            -llibopencv_core331d \
	                                            -llibopencv_dnn331d \
	                                            -llibopencv_features2d331d \
	                                            -llibopencv_flann331d \
	                                            -llibopencv_highgui331d \
	                                            -llibopencv_imgcodecs331d \
	                                            -llibopencv_imgproc331d \
	                                            -llibopencv_ml331d \
	                                            -llibopencv_objdetect331d \
	                                            -llibopencv_photo331d \
	                                            -llibopencv_shape331d \
	                                            -llibopencv_stitching331d \
	                                            -llibopencv_superres331d \
	                                            -llibopencv_video331d \
	                                            -llibopencv_videoio331d \
	                                            -llibopencv_videostab331d \
	SOURCES += \
	        main.cpp \
	        mainwindow.cpp
	
	HEADERS += \
	        mainwindow.h
	
	FORMS += \
	        mainwindow.ui


## 八、CLion中添加OpenCV库

	cmake_minimum_required(VERSION 3.12)
	project(untitled1)
	
	set(CMAKE_CXX_STANDARD 11)
	
	set(OpenCV_DIR D:/opencv-3.3.1/mingw-build)
	
	add_executable(untitled1 main.cpp)
	
	find_package(OpenCV REQUIRED)
	if (OpenCV_FOUND)
	    include_directories(${OpenCV_INCLUDE_DIRS})
	    target_link_libraries(untitled1 ${OpenCV_LIBS})
	else (OpenCV_FOUND)
	    message(FATAL_ERROR "OpenCV library not found")
	endif (OpenCV_FOUND)




## 参考文献
1. [OpenCV使用CMake和MinGW-w64的编译安装](https://blog.csdn.net/huihut/article/details/81317102)
2. [Opencv + opencv_contrib + Tesseract 之Qt开发环境搭建](https://www.cnblogs.com/hupeng1234/p/8593287.html)
3. [CMake之find_package](https://www.jianshu.com/p/46e9b8a6cb6a)