# -Yolo-
yolo系列的学习笔记

# YoLov1

1. ​

2. ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200722170142957.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dqaW5qaWU=,size_16,color_FFFFFF,t_70#pic_center)

3. **网络结构** 

   网络输入：448x448x3的彩色图片（Yolov1输入大小是固定的，因为全连接层需要前面的大小固定）

   中间层：由若干卷积层和最大池化层组成，用于提取图片的抽象特征。

   全连接层：由两个全连接层组成，用来预测目标的位置和类别概率值。

   网络输出：7x7x30的预测结果。（每个格子都会预测两个框，两个框中各有x,y,w,h,confidence，这些值为归一化后的结果0.几，而不是图像中的真实值。）所以30代表：5+5+20(种类)

   使用合适的损失函数，使损失最小就可以得到想要的结果。

4. **目标损失函数**：

   1. **位置损失函数** ：先对x,y,w,h进行选择（对每个格子中的两个框进行选择，选择IoU最大的框，代表这个框最好补救，或者最好微调，对w,h加根号，为了对小目标调整时更敏感，但是效果不是很好。）
   2. **置信度损失函数** ：
      1. 含Object的：选候选框中confidence最大的与真实框比较，IoU大于阈值，
      2. 不含Object：IoU小于阈值。含有权重，由于不重要，所以权重设置小一点。
   3. **分类损失函数**：20分类损失函数，使用交叉熵。计算概率。

5. **NMS非极大值抑制**：在测试的时候，可能会出现很多预选框，选择IoU最大的，其它的不要。

6. **优点**： 快速、简单。

7. **缺点**: 

   1. 重合在一起东西很难检测
   2. 小目标无法检测，预选框是固定的，只能检测较大的物体。
   3. 多标签物体无法检测，比如是狗 还是斑点狗，具有多个标签。
# YoLov2

1. **Batch Normalization**: v2版本舍弃了Dropout,卷积后全部加入Batch Normalization (v2不存在全连接层)
   1. 网络的每一层的输入都做归一化，收敛更容易。
   2. 现在网络必备。（增加后，网络提升2%mAP）
2. **V2使用更大的分辨率** ：
   1. v1训练时用224x224,测试时使用448x448
   2. v2刚开始训练使用224x224，最后额外加10次448x448的微调。（mAP提升约4%）
3. **网络结构** :
   1. ![img](https://img-blog.csdnimg.cn/d6cc9345cd22439f9344579120ae6e49.png#pic_center)
   2. DarkNet19，实际输入**416x416**  (因为除以32能整除，商希望为奇数，这样才有实际的中心点)（借鉴VGG,GoogLeNet）
   3. 没有FC层，5次下采样。（全连接层容易过拟合，参数多，训练慢），最终输出特征图大小为 w/32 , h/32。实际输出为**13x13** 。（特征图越大，特征越多）
   4. 使用1x1卷积，减少参数。卷积核全为3x3(借鉴VGG),1x1(借鉴GoogLeNet)
4. **v2-聚类提取先验框** ：
   1. yolov1使用两个先验框。faster-rcnn使用9种不同的先验框。
   2. yolov2中使用k-means聚类提取先验框。k=5。对训练数据集中的具体目标大小先进行聚类，得出5类大小不同的先验框，再根据这5类中的中心点，作为代表先验框。这样就使得先验框更合理一些。
   3. k-means中的距离：不是使用欧式距离，因为有的先验框大，误差就大。d = 1 - IoU
   4. K越大，先验框越精确。
# YoLov3

1. 特点：
   1. 最大的改进是网络结构，使其更适合小目标检测。
   2. 特征做的更细致，融入多持续特征图信息来预测不同规格物体。
   3. 先验框更丰富，3种scale，每种3个规格，一共9种。（V1:2种，v2：5种，v3:9种）
   4. softmax改进，预测多标签任务。
2. 多scale:为了能检测到不同大小的物体，设计了3个scale
   1. 一共三种scale:13x13预测大的，26x26预测中等的，52x52预测小的物体。
   2. 每个scale种，产生3个不同大小的候选框。
   3. scale变换经典方法
      1. 图像金字塔：对特征图做不同resize，排成图像金字塔，每层做预测。（速度慢）
      2. **特征金字塔** ：13x13特征图不仅做预测，并且，对13x13特征图做上采样变为26x26，与原来26x26特征图进行融合，得出26x26特征图的预测。（**特征图越小，越靠后，感受野越大，看到的东西越多，可以上采样后，与较大的特征图进行融合，相当于给浅层的特征图一些建议** ）
3. 残差连接：更好的特征（借鉴ResNet的思想）
   1. 没有池化和全连接层，全是卷积
   2. 下采样通过stride为2实现
   3. 3中scale，更多的先验框。（其中一个scale:13x13x3x85）
   4. 网格大小13x13,3指一个scale中预测三个box，80为样本类类别，5代表x,y,w,h,confidence
4. softmax层替代：（**logistic激活函数**）
   1. 物体检测任务中可能一个物体有多个标签。
   2. logistic激活函数来完成，这样就能预测每一个类别是/不是。最终按照confidence与阈值的比较，大于阈值的取出即可。就可以得到多标签分类。

# YoLov4

1. 改进：
   1. 单GPU就能训练的非常好。
   2. BOF
2. Bag of freebies(BOF)：扩充数据量，使数据有更强的泛化能力。
   1. 增加训练成本，提高精度，但不影响推理速度。
   2. 数据增强：调整亮度，对比度，色调，随机缩放，剪切，翻转，旋转。
   3. 网络正则化方法：Dropout、Dropblock等:(随机吃掉一个区域)
   4. 类别不平衡，损失函数设计。
3. Label Smoothing：神经网络太容易过拟合，很自信，所以不让太自信。使原本(0,1)变为[0.05,0.95]，是网络永远达不到最完美。
4. Bag of special (BOS)：网络层面的多样性增加
   1. 增加推断代价，但是提高了精度。
   2. SPPNet（Spatial Pyramid Pooling）：用最大池化来满足最终输入特征一致。
   3. CSPNet(Cross Stage partial Network):没一个block按照特征图的channel维度拆分成两部分，一份正常走网络，一份直接concat到这个block的输出。
   4. CBAM:加入注意力机制。
      1. 注意力机制：分为两部分，（1.有些特征图重要，有些不重要，就分大小比例。2.空间上，有些地方重要，有些地方不重要，划分比例。）
5. PAN(Path Aggregation Network)：特征金字塔中双向传递。
6. Mish激活函数：不一棒子打死，给<0，靠近0的一个机会。

# Yolov5

1. 不同点：

   1. 输入端：Mosaic数据增强、自适应锚框计算
   2. BackBone：Focus结构，CSP结构
   3. Neck：FPN+PAN结构
   4. Prediction：GIOU_Loss

2. Mosaic数据增强

3. 自适应锚框计算：

4. 自适应图片缩放：图片长宽不同时，常用方式为将原始图片统一缩放到一个标准尺寸。就需要对图片缩放或填充。填充后两端的黑边大小都不同，而**填充过多，则存在信息冗余，影响推理速度。**

   但是Yolov5在datasets.py中的letterbox函数中进行修改，对原始图像自适应的添加最少的黑边。

5. Focus结构：yolov4中没有，关键的是切片操作。

6. CSP结构：Yolov4中只有主干网络使用了CSP结构，而Yolov5中设计了两种CSP结构

7. Neck：Yolov4的Neck中，采用的都是普通的卷积操作。Yolov5中Neck结构采用借鉴CSPNet设计的CSP2结构，加强网络特征融合的能力。

8. **Yolov5**中采用其中的**CIOU_Loss**做**Bounding box**的**损失函数**。

9. **Yolov4**在**DIOU_Loss**的基础上采用**DIOU_nms**的方式，而**Yolov5**中仍然采用**加权nms**的方式。（可以进行修改：对于一些**遮挡重叠**的目标，确实会有一些**改进**。）
