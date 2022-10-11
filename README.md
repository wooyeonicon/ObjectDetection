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
