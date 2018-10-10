---
title: 在PySide2中嵌入matplotlib
key: 20181010
tags:
- PySide2 matplotlib plot
---

## 一、matplotlib简介
<!--more-->
&nbsp;&nbsp;&nbsp;&nbsp;[matplotlib](https://matplotlib.org/ "matplotlib")是一个Python 2D绘图库，可以生成各种硬拷贝格式和跨平台交互式环境的出版物质量数据。Matplotlib可用于Python脚本，Python和IPython shell，Jupyter笔记本，Web应用程序服务器和四个图形用户界面工具包。

&nbsp;&nbsp;&nbsp;&nbsp;matplotlib提供了三种不同风格的API供用户使用:

- pyplot API
- object-oriented API
- pylab API(disapproved)

&nbsp;&nbsp;&nbsp;&nbsp;pyplot API是函数式风格的一组API，借鉴了MATLAB的函数风格，使得matplotlib可以像MATLAB一样工作。

&nbsp;&nbsp;&nbsp;&nbsp;object-oriented API是面向对象风格的API，如果需要更多自定义绘图及高级使用，建议直接使用这些对象。OOP风格使用到的对象主要有**Figure类**, **Axes类**, **FigureCanvas类**及**Line2D类**。

&nbsp;&nbsp;&nbsp;&nbsp;在matplotlib中，整个图像为一个Figure对象，我们可以开启多个Figure窗口对象。在Figure对象中可以包含一个或者多个Axes对象，每个Axes对象都是一个拥有自己坐标系统的绘图区域，如下图所示。

![](https://images0.cnblogs.com/blog/413416/201301/28161628-cc13f9c508d44b59af00d482ccaeec95.png)

&nbsp;&nbsp;&nbsp;&nbsp;绘图区域主要由以下几个元素组成，Title为标题。Axis为坐标轴，Label为坐标轴标注。Tick为刻度线，Tick Label为刻度注释。

![](https://images0.cnblogs.com/blog/413416/201301/28161103-3defdfae66314204991f221298223d7a.png)

&nbsp;&nbsp;&nbsp;&nbsp;各个对象之间有下面的对象隶属关系：

![](https://images0.cnblogs.com/blog/413416/201301/29220639-06b363b5f4e14b9585be1be494712f8c.png)

&nbsp;&nbsp;&nbsp;&nbsp;了解了OOP风格主要的类元素后，为了实现matplotlib与PySide2的连接，主要对象就是**FigureCanvas类**, 它代表了具体实现绘图的后端(backend), 之前的分析都是在程序逻辑及OOP层面进行绘图，它必须连接一个后端绘图程序才能真正在屏幕上绘制出图像。可以理解为建房子，我们在设计图纸中设计好了房子的结构，需要施工队来将我们的设计变成现实的建筑。

## 二、matplotlib布局
&nbsp;&nbsp;&nbsp;&nbsp;Figure对象有两种添加Axes的方式，第一种就是使用

	add_axes([left, bottom, width, height]) #range 0 to 1
&nbsp;&nbsp;&nbsp;&nbsp;对于这种方式的好处是我们可以自由决定Axes在Figure上的位置及大小，如下图所示

![](https://raw.githubusercontent.com/whuhan2013/ImageRepertory/master/python/figure_12.png)

&nbsp;&nbsp;&nbsp;&nbsp;第二种添加Axes的方式就是使用matplotlib的布局管理器，可以类比PySide2中的layout。
	
	axes = add_subplot(nrows, ncols, position)
&nbsp;&nbsp;&nbsp;&nbsp;参数一为子图的总行数，参数二为子图的总列数，参数三为子图的位置，返回值为我们添加的Axes对象。

&nbsp;&nbsp;&nbsp;&nbsp;了解了matplotlib的基本工作原理后，后面对Figure及Axes的具体操作就可以查阅[官方文档](https://matplotlib.org/api/api_overview.html "官方文档")来获取了。

## 三、在PySide2中嵌入matplotlib
&nbsp;&nbsp;&nbsp;&nbsp;之前提到FigureCanvas是连接PySide2与matplotlib的关键，那么为什么FigureCanvas可以连接PySide2与matplotlib呢。

&nbsp;&nbsp;&nbsp;&nbsp;通过之前的分析我们知道要实现我们的绘图功能，需要一个“施工队”来帮助我们完成这个事情，在matplotlib中也就是backends,关于backends的具体描述可以查阅[官方文档](https://matplotlib.org/tutorials/introductory/usage.html#sphx-glr-tutorials-introductory-usage-py)，matplotlib有Qt5Agg、TkAgg、WXAgg、GTK3Agg等一系列的backends,这使得我们可以在其他程序中嵌入matplotlib实现我们的绘图功能。
&nbsp;&nbsp;&nbsp;&nbsp;为了便于理解如何通过FigureCanvas连接PySide2与matplotlib,我们从以下两句代码来入手。

	from matplotlib.backends.backend_qt5agg import FigureCanvasQTAgg as FigureCanvas
	
	issubclass(FigureCanvas, QtWidgets.QWidget)

&nbsp;&nbsp;&nbsp;&nbsp;第一句代码代表FigureCanvas是从backend_qt5agg导入的，那么也就是此时实现绘图的backends为Qt5,即PyQt5与PySide2.
&nbsp;&nbsp;&nbsp;&nbsp;第二句代码的值为True，代表FigureCanvas为QWidget的子类，既然是QWidget的子类，那么我们就可以将其当做QWidget来进行处理，作为我们程序中的小部件来使用。具体的示例可以查阅[官方示例Embeddeing in Qt](https://matplotlib.org/gallery/user_interfaces/embedding_in_qt_sgskip.html?highlight=pyqt)

## 四、在框架quina中使用matplotlib
&nbsp;&nbsp;&nbsp;&nbsp;本文最主要的目的是为了在quina中嵌入matplotlib来实现曲线图的绘制功能，quina是在PySide2上封装的一层业务框架，由于涉及公司业务，此处不做详解。一个最简单的嵌入示例程序为
	
	import quina
	import numpy as np
	
	from matplotlib.backends.backend_qt5agg import FigureCanvasQTAgg as FigureCanvas
	from matplotlib.figure import Figure
	
	if __name__ == '__main__':
	    # construct MainWindow
	    w = quina.widgets.MainWindow()
	
	    # prepare data
	    x = np.linspace(1, 10)
	    y = x ** 2
	
	    # construct Figure object
	    fig = Figure()
	    axes = fig.add_subplot(1, 1, 1)
	    axes.plot(x, y, 'r')
	
	    # construct FigureCanvas object
	    canvas = FigureCanvas(fig)
	
	    # set canvas to center widget in MainWindow
	    w.set_central_widget(canvas)
	
	    w.exec()

&nbsp;&nbsp;&nbsp;&nbsp;程序运行结果如下

![](https://img-blog.csdn.net/20181010103435577?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmdXQw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)