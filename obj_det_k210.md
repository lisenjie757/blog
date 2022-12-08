# K210目标检测

## 项目目标

- 本项目期望实现对楼道内出现杂物的进行检测，防止楼道内造成拥堵
- 在K210上实现相关代码

## 开发环境

- K210开发板
- [MaixPy](https://wiki.sipeed.com/soft/maixpy/zh/)（Micropython）

## 1 传统目标检测

通过传统图像处理的方法，使前景和背景的分离，以此来实现检测杂物的目标  
下面是实验测试中用到的几幅图片，分辨率均为224*224，最后一张是背景图：

[![b490.jpg](https://i.postimg.cc/vHDMDyTp/b490.jpg)](https://postimg.cc/xcrBFWKR) [![a1.jpg](https://i.postimg.cc/d1Dk0rKC/a118.jpg)](https://postimg.cc/hzWPyJv4) [![c1.jpg](https://i.postimg.cc/PxMkmyZw/c1.jpg)](https://postimg.cc/R6q8mcgS)  
[![b2.jpg](https://i.postimg.cc/NFD1nTmP/bback.jpg)](https://postimg.cc/y3JggDfF) [![c2.jpg](https://i.postimg.cc/SxbDhhHm/c2.jpg)](https://postimg.cc/4mwz6rJ0)

## 1.1 边缘检测

### 1.1.1 Canny算子（失败）

内置函数无法检出边缘，原因不明

> **image.find_edges(edge_type[, threshold])**
> - threshold：是一个包含一个低阈值和一个高阈值的二值元组，可以通过调整该值来控制边缘质量，默认(100,200)

```py
import sensor, image, time, lcd
lcd.init()

while(True):
    img = image.Image("/sd/test/edge/bback"+str()+".jpg",copy_to_fb=True)
    img = img.to_grayscale()
    img.find_edges(image.EDGE_CANNY)
    lcd.display(img)
```

[![c1.jpg](https://i.postimg.cc/PxMkmyZw/c1.jpg)](https://postimg.cc/R6q8mcgS) [![c1-canny.jpg](https://i.postimg.cc/1zrWdDsx/c1-canny.jpg)](https://postimg.cc/sB19QQMK)  
[![c2.jpg](https://i.postimg.cc/SxbDhhHm/c2.jpg)](https://postimg.cc/4mwz6rJ0) [![c1-canny.jpg](https://i.postimg.cc/1zrWdDsx/c1-canny.jpg)](https://postimg.cc/sB19QQMK)

### 1.1.2 Morph滤波

使用高通滤波器对图像进行边缘滤波  
需要定义一个卷积核

#### Morph滤波 + 自适应二值化

> **image.morph(size, kernel[, mul=Auto, add=0, threshold=False, offset=0, invert=False, mask)**
> - mul：与卷积像素结果相乘的数字，可进行全局对比度调整
> - add：与每个像素卷积结果相加的数值，可进行全局亮度调整
> - threshold：传递threshold=True参数来启动图像的自适应阈值处理，根据环境像素的亮度（核函数周围的像素的亮度有关），将像素设置为1或者0
> - offset：负数offset值将更多像素设置为1，而正值仅将最强对比度设置为1

```py
import sensor, image, time, lcd
lcd.init()
kernel_size = 1 # kernel width = (size*2)+1, kernel height = (size*2)+1
kernel = [-1, -1, -1,\
          -1, +8, -1,\
          -1, -1, -1]

while(True):
    img = image.Image("/sd/test/edge/bback"+str()+".jpg",copy_to_fb=True)
    img = img.to_grayscale()
    img.morph(kernel_size, kernel)
    img.binary([(img.get_histogram().get_threshold()[0], 256)])
    lcd.display(img)
```

[![c1.jpg](https://i.postimg.cc/PxMkmyZw/c1.jpg)](https://postimg.cc/R6q8mcgS) [![c1-morph-only.jpg](https://i.postimg.cc/0Qb8nYJn/c1-morph-only.jpg)](https://postimg.cc/1n1h5NBV) [![c1-morph-binary.jpg](https://i.postimg.cc/pTKTfXHL/c1-morph-binary.jpg)](https://postimg.cc/sQDsyzxk)  
[![c2.jpg](https://i.postimg.cc/SxbDhhHm/c2.jpg)](https://postimg.cc/4mwz6rJ0) [![c2-morph-only.jpg](https://i.postimg.cc/PfVDdy4X/c2-morph-only.jpg)](https://postimg.cc/JtJ0qbvv) [![c2-morph-binary.jpg](https://i.postimg.cc/PxKGsBbM/c2-morph-binary.jpg)](https://postimg.cc/r0dZ5ZPd)

#### Morph滤波 + 函数内二值化

使用`morph()`函数内置的自适应阈值处理，滤波效果与使用`binary()`有所不同，比较稀疏

```py
import sensor, image, time, lcd
lcd.init()
kernel_size = 1 # kernel width = (size*2)+1, kernel height = (size*2)+1
kernel = [-1, -1, -1,\
          -1, +8, -1,\
          -1, -1, -1]

while(True):
    img = image.Image("/sd/test/edge/bback"+str()+".jpg",copy_to_fb=True)
    img = img.to_grayscale()
    img.morph(kernel_size, kernel, threshold=True, invert=True)
    lcd.display(img)
```

[![c1.jpg](https://i.postimg.cc/PxMkmyZw/c1.jpg)](https://postimg.cc/R6q8mcgS) [![c1-morph.jpg](https://i.postimg.cc/3xbC69DD/c1-morph.jpg)](https://postimg.cc/gx8ZLVsY)  
[![c2.jpg](https://i.postimg.cc/SxbDhhHm/c2.jpg)](https://postimg.cc/4mwz6rJ0) [![c2-morph.jpg](https://i.postimg.cc/XJNjCKQQ/c2-morph.jpg)](https://postimg.cc/Lq7FdPd1)

#### Morph滤波 + 函数内二值化 + 腐蚀/膨胀

腐蚀和膨胀是形态学运算，用于去除噪声

- 腐蚀：从分割区域的边缘删除像素
- 膨胀：将像素添加到分割区域的边缘中
- 开运算：先腐蚀再膨胀，消除小物体，断开纤细连接
- 闭运算：先膨胀再腐蚀，消除小型空洞，连接间断的沟壑

> **image.erode(size[, threshold[, mask=None]])**
> **image.dilate(size[, threshold[, mask=None]])**
> **image.open(size[, threshold[, mask=None]])**
> **image.close(size[, threshold[, mask=None]])**
> - size：卷积核尺寸，这些方法通过二维卷积运算来实现
> - threshold：用来设置去除相邻点的个数，threshold数值越大，相应操作的程度就越大

```py
import sensor, image, time, lcd
lcd.init()
kernel_size = 1 # kernel width = (size*2)+1, kernel height = (size*2)+1
kernel = [-1, -1, -1,\
          -1, +8, -1,\
          -1, -1, -1]

while(True):
    img = image.Image("/sd/test/edge/c2"+str()+".jpg",copy_to_fb=True)
    img = img.to_grayscale()
    img.morph(kernel_size, kernel, threshold=True, invert=True,add=70)
    img.erode(1, threshold = 2)
    lcd.display(img)
```

[![c1.jpg](https://i.postimg.cc/PxMkmyZw/c1.jpg)](https://postimg.cc/R6q8mcgS) [![c1-morph-add.jpg](https://i.postimg.cc/P55rNnr0/c1-morph-add.jpg)](https://postimg.cc/pp3t17jB) [![c1-erode.jpg](https://i.postimg.cc/MH9NFN2j/c1-erode.jpg)](https://postimg.cc/Zvyw9fqT)  
[![c2.jpg](https://i.postimg.cc/SxbDhhHm/c2.jpg)](https://postimg.cc/4mwz6rJ0) [![c2-morph-add.jpg](https://i.postimg.cc/Qd1SBw4k/c2-morph-add.jpg)](https://postimg.cc/5X98T7M6) [![c2-erode.jpg](https://i.postimg.cc/dV2mQd35/c2-erode.jpg)](https://postimg.cc/sM2ZwMpW)

## 1.2 轮廓提取（无内置函数）

在边缘检测之后，通过提取边缘的外接轮廓，实现对杂物的提取  
在opencv中可以使用`cv2.findContours`函数实现此功能，而在MaixPy中无相应函数

## 1.3 帧差分

通过当前图像与背景图像做差来判断当前图像中有无杂物

### 1.3.1 背景差分

背景差分是先采样一张无杂物的纯背景图像作为固定背景图像，与当前图像作差分  
优点是简单易于实现，缺点是难以易受光照等外在因素的干扰

### 1.3.2 帧间差分

帧间差分即相邻几帧间的差分，背景图像是动态变化的  
优点是可以排除光照等背景变化的影响，缺点是时间一久前景（杂物）也会被建模进背景而无法提取，不符合本项目的要求

## 1.4 图像相似性

## 2 深度学习目标检测

## 2.1 物体分类

## 2.2 物体检测

my modirty tests  
sjh insnfi  
atestd
sasdsd
3jsd
sadsd
