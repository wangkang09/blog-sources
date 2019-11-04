[TOC]

## 1 UML 类图关系种类及强弱关系
- 继承 = 实现 > 组合 > 聚合 > 关联 > 依赖
- 组合：部分不能离开整体
- 聚合：部分可以离开整体
- 关联：属于拥有关系(个人理解)，并且不是部分拥有整体的那种，是拥有别的。拥有什么东西，并不一定会依赖这件东西
- 依赖：属于需要关系，同理需要别的，而不是整体
- 其中组合、聚合的代码形式：**一对多的成员变量关系**
- 关联的代码形式：**一对一**的成员变量关系
- 依赖的代码形式：**局部变量、方法参数、对静态方法的调用**

## 2 示例图1
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190603221415510.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2thbmdzYTk5OA==,size_16,color_FFFFFF,t_70)
## 3 示例图2
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190603223642362.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2thbmdzYTk5OA==,size_16,color_FFFFFF,t_70)
## 参考
[UML各种图总结-精华](https://www.cnblogs.com/jiangds/p/6596595.html)
[Java 后端技术 ——UML 图](https://mp.weixin.qq.com/s/N92K9WwvHdpbej2btC-ofA)
大话设计模式
