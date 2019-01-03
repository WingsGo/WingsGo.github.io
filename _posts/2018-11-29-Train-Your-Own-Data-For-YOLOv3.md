---
title: YOLOv3-Train Your Own Data Step by Step
key: 20181129
tags:
- YOLOv3 CNN Object Localization
---

## 一、YOLO算法简介
<!--more-->
&nbsp;&nbsp;&nbsp;&nbsp;[YOLO](https://pjreddie.com/darknet/yolo/)是一个基于卷积神经网络的实时目标检测算法，基于darknet框架实现，关于YOLO算法的理论知识可以参考YOLO的官方网站及paper，另外还可以通过吴恩达的深度学习课程[Convolutional Neural Networks](https://www.coursera.org/learn/convolutional-neural-networks/home/week/3)，学习YOLO算法的原理。本文的目的是通过作者提供的代码，实现自己的数据集的模型训练过程。

## 二、数据集准备
&nbsp;&nbsp;&nbsp;&nbsp;首先需要准备自己的数据集，本文以单一类型识别为例，介绍如何进行单一分类的数据训练。作者官网有介绍如何在VOC格式的数据集下进行训练，关于VOC数据集的格式可以参考这篇博客[PASCAL VOC数据集分析](https://blog.csdn.net/zhangjunbob/article/details/52769381),笔者尝试训练了VOC数据集的20种分类数据，在Ubuntu16.04LTS+2G显存GTX960m显卡上训练，迭代45000次大概用时3天4夜，得出了初步的效果，所以不建议初次进行训练的人使用VOC数据集进行训练。笔者本次训练使用的数据集为食物图片数据集，初步选取了300张图片。接下来便是对数据进行标注。

## 三、数据标记
&nbsp;&nbsp;&nbsp;&nbsp;YOLOv3目标检测需要对要识别的数据以最小包围矩阵进行标记，需要中心点及四个顶点的像素坐标作为图片的标签。此处推荐一个标注工具[LabelImg](https://github.com/tzutalin/labelImg)
具体的安装使用方法在作者的github上已经写得很清楚，此处不再赘述。该工具标注后的标签是标准的VOC数据集的XML格式标签，文件名和图片名一致，这样方便后续我们的转换，我们要记住标签文件的保存路径，笔者以自己的为例，图片文件存放于*/home/wingc/SourceCode/food_detection/darknet/data/obj*。标签文件存放于 */home/wingc/SourceCode/food_detection/darknet/data/obj/annotations*。标签文件的格式如下所示

	<annotation>
	    <folder>Image</folder>
	    <filename>0000</filename>
	    <source>
	        <database>Pedestrian_ultrared</database>
	    </source>
	    <size>
	        <width>384</width>
	        <height>288</height>
	    </size>
	    <object>
	        <name>n00000001</name>
	        <bndbox>
	            <xmin>232</xmin>
	            <xmax>248</xmax>
	            <ymin>161</ymin>
	            <ymax>203</ymax>
	        </bndbox>
	    </object>
	</annotation>

## 四、标签格式转换
&nbsp;&nbsp;&nbsp;&nbsp;第四步就是将标记好的xml格式的标签转化为txt格式，这里提供一个脚本进行格式的转化，具体的文件路径根据个人的不同配置进行修改。

    import xml.etree.ElementTree as ET
	import glob
	import os
	
	sets = [('2012', 'train'), ('2012', 'val'), ('2007', 'train'), ('2007', 'val'), ('2007', 'test')]
	
	classes = ["Shredded cabbage"]
	
	
	def convert(size, box):
	    dw = 1. / (size[0])
	    dh = 1. / (size[1])
	    x = (box[0] + box[1]) / 2.0 - 1
	    y = (box[2] + box[3]) / 2.0 - 1
	    w = box[1] - box[0]
	    h = box[3] - box[2]
	    x = x * dw
	    w = w * dw
	    y = y * dh
	    h = h * dh
	    return x, y, w, h
	
	
	# 图片所在路径及图片名
	def convert_annotation(base_path, img_id):
	    in_file = open(os.path.join(base_path, 'annotations/{}.xml'.format(img_id)), 'r')
	    out_file = open(os.path.join(base_path, 'labels/{}.txt'.format(img_id)), 'w')
	    tree = ET.parse(in_file)
	    root = tree.getroot()
	    size = root.find('size')
	    w = int(size.find('width').text)
	    h = int(size.find('height').text)
	
	    for obj in root.iter('object'):
	        difficult = obj.find('difficult').text
	        cls = obj.find('name').text
	        if cls not in classes or int(difficult) == 1:
	            continue
	        cls_id = classes.index(cls)
	        xmlbox = obj.find('bndbox')
	        b = (float(xmlbox.find('xmin').text), float(xmlbox.find('xmax').text), float(xmlbox.find('ymin').text),
	             float(xmlbox.find('ymax').text))
	        bb = convert((w, h), b)
	        out_file.write(str(cls_id) + " " + " ".join([str(a) for a in bb]) + '\n')
	
	
	def generate_train_test_txt(work_path, xml_path, train_path, test_path, test_percentage):
	    train_file = open(train_path, 'w')
	    test_file = open(test_path, 'w')
	    counter = 1
	    test_index = round(100 / test_percentage)
	    for xml_file_path in glob.iglob(os.path.join(xml_path, "*.xml")):
	        title, ext = os.path.splitext(os.path.basename(xml_file_path))
	        if counter == test_index:
	            counter = 1
	            test_file.write(work_path + '/' + title + '.jpg' + '\n')
	        else:
	            train_file.write(work_path + '/' + title + '.jpg' + '\n')
	            counter += 1
	    train_file.close()
	    test_file.close()
	
	
	if __name__ == '__main__':
	    wd = input("请输入图片所在文件夹\n")
	    xml_wd = os.path.join(wd, 'annotations')
	    label_wd = os.path.join(wd, 'labels')
	    train_txt = os.path.join(wd, 'train.txt')
	    test_txt = os.path.join(wd, 'test.txt')
	
	    # 划分训练集与测试集并写入文件
	    generate_train_test_txt(wd, xml_wd, train_txt, test_txt, 10)
	
	    # 将标签数据放入labels文件夹
	    if not os.path.exists(os.path.join(wd, 'labels')):
	        os.makedirs(os.path.join(wd, 'labels'))
	
	    train_img_paths = open(train_txt, 'r').read().strip().split()
	    test_img_paths = open(test_txt, 'r').read().strip().split()
	    img_paths = train_img_paths + test_img_paths
	    for img_path in img_paths:
	        convert_annotation(wd, os.path.splitext(os.path.basename(img_path))[0])

需要注意的是如果数据集格式不是按照PASCAL VOC数据集来组织的，那么txt格式的标签需要保存到和图片文件同一目录下,原因在于代码中会根据图片路径读取标签路径，代码在data.c的fill_truth_detection函数中，具体的代码分析可参考博客[YOLOv2代码分析_读取labels{by zhangzexuan}](https://blog.csdn.net/xuan_xuan_/article/details/78269123)

## 五、数据集的划分
&nbsp;&nbsp;&nbsp;&nbsp;接下来我们就要将数据集划分为训练集和测试集，训练集的图片路径存放在train.txt中，一行为一张图片的路径,测试集的图片路径存放在test.txt中，数据集划分的脚本在第四部分已经给出，训练集与测试集的比例在代码中可进行调整，同时由于笔者只进行了部分图片标签的标注，所以是根据已标记的XML格式标签的文件名进行划分。

## 六、配置文件的准备
&nbsp;&nbsp;&nbsp;&nbsp;当以上的准备工作完成后，最后一步就是进行配置文件的配置，这也是最重要的一步，直接决定你训练的成败与否。我们需要创建三个配置文件

- cfg/obj.data
- cfg/obj.names
- cfg/yolo-obj.cfg

&nbsp;&nbsp;&nbsp;&nbsp;obj.data文件指定了划分的种类，训练集与测试集的路径，要识别的物体的名字，及训练过程中权重文件的保存路径。

	classes= 1  
	train  = train.txt  
	valid  = test.txt  
	names = obj.names  
	backup = backup/ 

&nbsp;&nbsp;&nbsp;&nbsp;obj.names中存放要标记的物体的名称，每行对应一种类别,这里注意到只指定了图片的路径而没有准备标签的路径,原因在第四节已经解释,标签路径会根据图片路径来寻找,例如:

图片路径为:/home/project/images/test.jpg或/home/project/JEPGimages/test.jpg或/home/project/任意名称/test.jpg

标签路径为:/home/project/labels/test.txt或/home/project/labels/test.txt或/home/project/任意名称/test.txt

	Shredded Cabbage

&nbsp;&nbsp;&nbsp;&nbsp;yolo-obj.cfg最为重要，改配置文件定义了一些超参数，以下说几个重点的地方：

1. batch与subdivisions要注意测试的时候和训练的时候要分别注释掉不同的部分，batch指的是一次输入多少张图片进行一次迭代，batch如果设置的太大，可能导致内存爆掉，提示核心已转存储，这时候你就需要调整一下batch的大小，最好为2的倍数。笔者2G的GTX960m的显卡上设置为16。训练完进行测试的时候需要将这里再改回test的batch与subdivisions，即batch=1,subdivisions=1。
2. 设置classes=1
3. 设置filters=(classes + 5) * 3，filters的设置依据可以看[这里](https://github.com/AlexeyAB/darknet)

下面提供一个我自己的配置文件，需要修改的地方已经用星号标出，如果对参数修改还有疑问，可以参考这篇博客
[YOLOv3: 训练自己的数据](https://blog.csdn.net/lilai619/article/details/79695109)
	
	[net]
	# Testing            ### 测试模式                                          ★★★
	# batch=1
	# subdivisions=1
	# Training           ### 训练模式，每次前向的图片数目 = batch/subdivisions ★★★
	batch=32
	subdivisions=8
	width=416            ### 网络的输入宽、高、通道数                          ★★
	height=416
	channels=3
	momentum=0.9         ### 动量                                              ★
	decay=0.0005         ### 权重衰减                                          ★
	angle=0
	saturation = 1.5     ### 饱和度                                            ★
	exposure = 1.5       ### 曝光度                                            ★
	hue=.1               ### 色调                                              ★
	
	learning_rate=0.001  ### 学习率                                            ★★★
	burn_in=1000         ### 学习率控制的参数
	max_batches = 50200  ### 迭代次数                                          ★★★
	policy=steps         ### 学习率策略                                        ★★
	steps=40000,45000    ### 学习率变动步长                                    ★★
	scales=.1,.1         ### 学习率变动因子                                    ★★
	
	
	
	[convolutional]
	batch_normalize=1    ### BN
	filters=32           ### 卷积核数目
	size=3               ### 卷积核尺寸
	stride=1             ### 卷积核步长
	pad=1                ### pad
	activation=leaky     ### 激活函数
	
	# Downsample
	
	[convolutional]
	batch_normalize=1
	filters=64
	size=3
	stride=2
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=32
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=64
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	# Downsample
	
	[convolutional]
	batch_normalize=1
	filters=128
	size=3
	stride=2
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=64
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=128
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3              ### 连接层
	activation=linear
	
	[convolutional]
	batch_normalize=1
	filters=64
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=128
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	# Downsample
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=3
	stride=2
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=128
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	[convolutional]
	batch_normalize=1
	filters=128
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	[convolutional]
	batch_normalize=1
	filters=128
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	[convolutional]
	batch_normalize=1
	filters=128
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	
	[convolutional]
	batch_normalize=1
	filters=128
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	[convolutional]
	batch_normalize=1
	filters=128
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	[convolutional]
	batch_normalize=1
	filters=128
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	[convolutional]
	batch_normalize=1
	filters=128
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	# Downsample
	
	[convolutional]
	batch_normalize=1
	filters=512
	size=3
	stride=2
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=512
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=512
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=512
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=512
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=512
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=512
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=512
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=512
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	# Downsample
	
	[convolutional]
	batch_normalize=1
	filters=1024
	size=3
	stride=2
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=512
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=1024
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	[convolutional]
	batch_normalize=1
	filters=512
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=1024
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	[convolutional]
	batch_normalize=1
	filters=512
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=1024
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	[convolutional]
	batch_normalize=1
	filters=512
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=1024
	size=3
	stride=1
	pad=1
	activation=leaky
	
	[shortcut]
	from=-3
	activation=linear
	
	######################
	
	[convolutional]
	batch_normalize=1
	filters=512
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	size=3
	stride=1
	pad=1
	filters=1024
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=512
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	size=3
	stride=1
	pad=1
	filters=1024
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=512
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	size=3
	stride=1
	pad=1
	filters=1024
	activation=leaky
	
	[convolutional]
	size=1
	stride=1
	pad=1
	filters=18                                             ★★★
	activation=linear
	
	[yolo]
	mask = 6,7,8
	anchors = 10,13,  16,30,  33,23,  30,61,  62,45,  59,119,  116,90,  156,198,  373,326
	classes=1                                              ★★★
	num=9
	jitter=.3
	ignore_thresh = .5
	truth_thresh = 1
	random=0                                               ★★★
	
	[route]
	layers = -4
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[upsample]
	stride=2
	
	[route]
	layers = -1, 61
	
	
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	size=3
	stride=1
	pad=1
	filters=512
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	size=3
	stride=1
	pad=1
	filters=512
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=256
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	size=3
	stride=1
	pad=1
	filters=512
	activation=leaky
	
	[convolutional]
	size=1
	stride=1
	pad=1
	filters=18                                                  ★★★
	activation=linear
	
	[yolo]
	mask = 3,4,5
	anchors = 10,13,  16,30,  33,23,  30,61,  62,45,  59,119,  116,90,  156,198,  373,326
	classes=1                                                   ★★★
	num=9
	jitter=.3
	ignore_thresh = .5
	truth_thresh = 1
	random=0                                                    ★★★
	
	[route]
	layers = -4
	
	[convolutional]
	batch_normalize=1
	filters=128
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[upsample]
	stride=2
	
	[route]
	layers = -1, 36
	
	
	
	[convolutional]
	batch_normalize=1
	filters=128
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	size=3
	stride=1
	pad=1
	filters=256
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=128
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	size=3
	stride=1
	pad=1
	filters=256
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	filters=128
	size=1
	stride=1
	pad=1
	activation=leaky
	
	[convolutional]
	batch_normalize=1
	size=3
	stride=1
	pad=1
	filters=256
	activation=leaky
	
	[convolutional]
	size=1
	stride=1
	pad=1
	filters=18              ### 3x(classes + 4coor + 1prob) = 3x(20+4+1) = 75              ★★★
	activation=linear
	
	[yolo]
	mask = 0,1,2            ### mask序号                        ★
	anchors = 10,13,  16,30,  33,23,  30,61,  62,45,  59,119,  116,90,  156,198,  373,326  ★★★
	classes=20              ### 类比数目                        ★★★
	num=9
	jitter=.3               ### 数据扩充的抖动操作              ★
	ignore_thresh = .5      ### 文章中的阈值1                   ★
	truth_thresh = 1        ### 文章中的阈值2                   ★
	random=0                ### 多尺度训练开关                  ★★★

## 七、训练过程中参数的保存
&nbsp;&nbsp;&nbsp;&nbsp;在linux下可通过数据重定向，将训练过程中输出的数据保存到文件中，并通过python脚本解析可视化，具体的步骤可参考博客[YOLOv3使用笔记——曲线可视化](https://blog.csdn.net/cgt19910923/article/details/80783614)文中已经写得很详细，并提供了可视化的脚本。

## 八、批量测试图片
&nbsp;&nbsp;&nbsp;&nbsp;[YOLOv3批量测试图片并保存在自定义文件夹下](https://blog.csdn.net/mieleizhi0522/article/details/79989754)

## 总结
&nbsp;&nbsp;&nbsp;&nbsp;到这里就完成了整个YOLOv3从数据集准备到得到训练后权重文件的过程，并且还提供了训练过程的可视化及批量图片测试，应该能满足大部分场景下的需求了，在这里还是要感谢以上博主的无私分享，最后建议大家有时间可以看一下源码，如果读懂了源码，以上这些就都不是问题了。	

## 参考文献
1. [YOLO: Real-Time Object Detection](https://pjreddie.com/darknet/yolo/)
2. [darknet](https://github.com/AlexeyAB/darknet)
3. [How to train YOLOv2 to detect custom objects](https://timebutt.github.io/static/how-to-train-yolov2-to-detect-custom-objects/)
4. [YOLOv3: 训练自己的数据](https://blog.csdn.net/lilai619/article/details/79695109)
5. [PASCAL VOC数据集分析](https://blog.csdn.net/zhangjunbob/article/details/52769381)
6. [YOLOv2代码分析_读取labels{by zhangzexuan}](https://blog.csdn.net/xuan_xuan_/article/details/78269123)
7. [YOLOv3使用笔记——曲线可视化](https://blog.csdn.net/cgt19910923/article/details/80783614)
8. [YOLOv3批量测试图片并保存在自定义文件夹下](https://blog.csdn.net/mieleizhi0522/article/details/79989754)
