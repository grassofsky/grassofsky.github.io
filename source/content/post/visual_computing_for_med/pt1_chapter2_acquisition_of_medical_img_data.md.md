+++
date="2020-01-02" 
title="visual computing for med reading - pt1 chapter 2 获取医学影像数据"
categories=["visual computing for med reading"] 

+++

#  visual computing for med - pt1 chapter 2 获取医学影像数据

## 2.1 简介

如何选择成像模态？

- 相关的解剖结构必须描述完整，
- 数据的分辨率足够以回答特定的诊断和治疗问题，
- 与对比度、信噪比和伪影有关的图像质量必须足以解释与诊断和治疗问题有关的数据，
- 应尽量减少对病人和医生的负担，
- cost需要做一定的限制

对于医学影像而言，最佳空间分辨率和最佳图像质量都不是临床实践中的相关目标。

## 2.2 医学影像数据

医学影像数据的形式，很简单，不用上图了，均匀分布的2D网格或3D网格。对应的像素（或体素）位于网格点上。**这里可能需要注意的是，网格的顶点对应像素值或体素，而不是网格的中心**。

然后给出了线性插值，双线性插值，三线性插值的公式。引出了triubic spline functions。

**TODO：add python code**

最后介绍了领域，6-neighborhood，以及26-neighborhood。



