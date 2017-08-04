# 行为识别

## **目标**

给视频片段进行一定的分类，类别通常是各类人的动作

## **特点**

一个视频片段中包含一段明确的动作，所以也可以看作是输入为视频，输出为动作标签的多分类问题。

## **难点**

特征：即如何在视频中提取出能更好的描述视频判断的特征。特征提取的越好，则分类的效果越好。

特征的编码（encode）/融合（fusion）：这一部分包括两个方面：

空间方面的，在使用多种特征的时候如何编码/融合这些特征以获得更好的效果；

时序方面的，视频很重要的一个特性就是其时序信息，一些动作看单帧的图像是无法判断的，只有通过时序上的变化判断，所以需要将时序上的特征进行编码或者融合，获得整个视频的特征描述。

算法速度：在训练的时候，算法的速度不是特别重要。但是产品上线需要快速的算法。

## **研究过往**

### 非深度学习方法：Action recognition with improved trajectories

IDT（improved dense trajectories）算法稳定性不错，速度比较慢。基本思路，利用光流场来获得视频中的轨迹。沿着轨迹提取，HOF,HOG,MBH,trajectory等特征。

HOF基于灰度,其他基于dense optical flow\(密集光流\)，最后用Fisher Vector特征编码，基于结果训练SVM分类器。iDT改进的是利用前后两帧的光流以及SURF关键点进行匹配，可以用来消除/减弱相机运动带来的影响。

![](/assets/import.png)

### **深度学习方法**

#### Two Stream Network系列

[Two-Stream Convolutional Networks for Action Recognition in Videos （2014NIPS）](https://arxiv.org/pdf/1406.2199.pdf)

主要分为时间和空间两个部分：

* 空间中携带场景和目标。
* 时间中，包含运动信息，目标物体和相机的运动信息。

每个流，用一个深度卷积网络实现，每个流softmax分数在最后进行融合。两种融合方法：

* 求平均值
* 一个叠放的L2正则规划的softmax得分上训练SVM线性分类器

![](/assets/figure1.png)

模型结构如上图所示。![](/assets/figure2.png)光流的效果图.

一些概念：

光流卷积网络：我们模型的输入是几个相邻帧的堆叠的光流位移。这些输入能够描述出视频帧的运动信息。

光流堆叠：一个密集的光流能够被看做是一系列连续帧的位移向量。水平和垂直两部分分开。为了表示一个序列帧的运动信息，我们堆叠L个连续帧的流通道来形成一个总数为2L个输入通道。

轨迹堆叠：另一个可供选择的运动表示，受到基于轨迹描述子的启发，取代光流，在连续几帧相同的位置上采样，根据光流，得到迹的运动信息。

双向光流：

减去平均光流：

时间域卷积网络结构与先前的表示的关系：在本文中，运动信息通过用光流位移来准确的表示。

多任务学习：因为视频训练的数据集相对较小，容易产生过拟合，为了避免这种情况的发生，我们合并多个数据集。

实现细节：卷积网络的配置，所有的隐含层用ReLU激活函数；max pooling的大小为3\*3，步长为2；时间网络和空间网络位移的不同就是，我们删除了时间网络第二层的正则化来减少内存消耗。

对于时间，光流的CNN,思路：从相邻的L帧图片中，提取光流信息作为输入，然后以此为信息输入，然后表示时间。首先用，OpenCV直接获取对应的光流信息，所谓所谓光流信息分几种，传统的是定义了一个displacement——dt\(u,v\)，这个表示在t时刻对应帧上的一个点\(u,v\)，要把它移动到t+1时刻相应地方的方向向量。至于这个点具体要怎么得出，可以用OpenCV直接处理视频的帧然后得到。此外还有其他几种表示光流的方法。

有了对应的光流信息之后，对于某一帧，首先空间信息就是把原图（经过处理）输入到网络，然后空间信息就是从该帧开始接下来连续的L帧，每相邻两帧之间提取所有点的光流信息作为输入，分为X轴和Y轴方向，因此设每帧长宽为w\*h个像素点，那么每帧对应的时间网络输入信息的维数就是w\*h\*2L

最后，空间和时间网络分别给出动作的分类结果，然后把结果融合，使用取平均值或者SVM的方法（实验中显示SVM准确率更高），得到最终结果。

代码的repo:[git code repo](https://github.com/wadhwasahil/Video-Classification-2-Stream-CNN)

[3D Convolutional Neural Networks for Human Action Recognition](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.169.4046&rep=rep1&type=pdf)

之前一般是2D的卷积方法，是对二维的卷积层上对特征进行采样，从而得到下一个层的特征图，形式如下：

![](/assets/2d.png)

3D的增加一个新的时间维度，对一定的数量的帧，用同一个3D的卷积层去采样，得到下个维度的特征，形式：

![](/assets/3dconv.png)

网络的结构如下：![](/assets/3dconvstructure.png)

首先通过硬连接\(hardwired\)的一些kernal，抽取多通道的信息，  
主要有gray, gradient-x, gradient-y, optflow-x,and optflow-y.这些kernal

[Convolutional Two-Stream Network Fusion for Video Action Recognition-\(2016CVPR）](https://arxiv.org/pdf/1604.06573.pdf)



这篇文章采用学习大量融合Convent的既有tower是spatially空间的 还有patio-temporal 时间的.使用了如下方式：

与softmax层融合相比,一个时空网络层可以融合但是不失性能.最后一层融合比较好.池化抽象卷积在相邻时空的特征会提升性能。









