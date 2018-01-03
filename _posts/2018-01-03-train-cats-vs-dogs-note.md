---
layout: post
title: 训练cats_vs_dogs心得
data: 2018-01-03
author: Nicholas Huang
header-img: 
categories: CNN
tags:
    - CNN
    - Deep Learning
--- 

# 网络框架演化
最初的代码来源于[github](https://github.com/kevin28520/My-TensorFlow-tutorials/tree/master/01%20cats%20vs%20dogs)，我利用他的代码把cnn网络改成了AlexNet，进行训练。结果是识别为猫的个数是6104，识别为狗的个数是6396。这个结果很不行，我准备加ResNet来提高效果。
# 测试效果对比
修改为AlexNet的测试结果
accuracy如图：
![alex_net_accuracy](https://github.com/h312903294/MarkdownPicRep/raw/master/alex_net_accuracy.png)
loss如图：
![alex_net_loss](https://github.com/h312903294/MarkdownPicRep/raw/master/alex_net_loss.png)
# 踩坑记录
test代码中不要调用tf.global_variables_initializer，它会导致测试结果的随机化。



