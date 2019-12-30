+++
date="2019-12-30" 
title="ITK example - digitally reconstructed radiograph"
categories=["ITK"] 
tags=["ITK example"]

+++

# ITK example - digitally reconstructed radiograph

## 简介

该示例根据 https://github.com/InsightSoftwareConsortium/ITK/blob/master/Examples/Filtering/DigitallyReconstructedRadiograph1.cxx 实现对应python版本。

## 重点类介绍

### ResampleImageFilter

参见：https://itk.org/Doxygen/html/classitk_1_1ResampleImageFilter.html

通过坐标变换对图像进行重采样。ResampleImageFilter 通过一些坐标变换对存在的图像进行重采样，通过一些image function进行插值。

注意，插值函数的选择是很重要的。这个函数通过[SetInterpolator()](https://itk.org/Doxygen/html/classitk_1_1ResampleImageFilter.html#ac2d2efedb7d195170b2d8d424b2690c4)进行设定。默认的是`LinearInterpolateImageFunction<InputImageType, TInterpolatorPrecisionType>`，对于普通的医学影像是够用了的。但是，一些合成图像的像素是从有限的指定集合中提取的。一个例子是一个Mask，它将大脑标记成少量的组织类型。对于这样的图像，并不想要对这些像素值之间的值进行插值，这个时候 `NearestNeighborInterpolateImageFunction< InputImageType, TCoordRep >`会是一个更好的选择。

如果样本来自图像区域的外部，默认情况下使用默认的像素值。如果需要有不同的表现，可以通过`SetExtrapolator().`函数进行设置。

输出图像的信息（spacing，size，direction）需要进行设置。这些信息具有单位间距、零原点、标识方向的默认值。可选地，可以从参考图像获得输出信息。如果提供了参考图像并且用户参考图像处于打开状态，则将使用参考图像的间距、原点和方向。


### CenteredEuler3DTransform

参见：https://itk.org/Doxygen/html/classitk_1_1CenteredEuler3DTransform.html

这个变换围绕特定的坐标系或者中心进行旋转，然后进行平移。

- SetComputeZYX： 旋转操作依赖于旋转顺序，设定为ZXY，或ZYX

### RayCastInterpolateImageFunction

将光线投射到三维图像中，并使用双线性插值对遍历的每个体素平面进行积分。

- SetFocalPoint：设置光线源的位置
- SetThreshold：设置阈值之上的voxels才会被累加
- SetTransform：设置Transform，用来计算新的光源的位置



```python
import math
import numpy as np
import itk
import itkwidgets
from itkwidgets import view
from ipywidgets import interactive
import ipywidgets as widgets
```


```python
# 加载原始体数据并显示
input_name = 'data/CT_3D_lung_fixed.mha'
volume_lung = itk.imread(input_name, itk.ctype('float')) # 读取影像文件，并将数据格式转换为float
print(volume_lung.GetLargestPossibleRegion().GetSize())
print(volume_lung.GetBufferedRegion().GetSize())
print(volume_lung.GetSpacing())
print(volume_lung.GetOrigin())
view(volume_lung, gradient_opacity=0.5, cmp=itkwidgets.cm.bone)
```

    itkSize3 ([115, 157, 129])
    itkSize3 ([115, 157, 129])
    itkVectorD3 ([1.366, 1.366, 2.5])
    itkPointD3 ([-153.827, -150.352, -1434.5])



    Viewer(geometries=[], gradient_opacity=0.5, point_sets=[], rendered_image=<itkImagePython.itkImageF3; proxy of…



```python
output_image_pixel_spacing = [1.366, 1.366, 1]
output_image_size = [501, 501,1] #[501, 501, 1]
    
   
InputImageType = type(volume_lung)
FilterType = itk.ResampleImageFilter[InputImageType, InputImageType]
filter = FilterType.New()
filter.SetInput(volume_lung)
filter.SetDefaultPixelValue(0)
filter.SetSize(output_image_size)
filter.SetOutputSpacing(output_image_pixel_spacing)
    
TransformType = itk.CenteredEuler3DTransform[itk.D]
transform = TransformType.New()
transform.SetComputeZYX(True)

InterpolatorType = itk.RayCastInterpolateImageFunction[InputImageType, itk.D]
interpolator = InterpolatorType.New()

viewer = None

def DigitallyReconstructedRadiograph(
    ray_source_distance=100, 
    camera_tx=0., 
    camera_ty=0., 
    camera_tz=0.,
    rotation_x=0.,
    rotation_y=0.,
    rotation_z=0.,
    projection_normal_p_x=0.,
    projection_normal_p_y=0.,
    rotation_center_rt_volume_center_x=0.,
    rotation_center_rt_volume_center_y=0.,
    rotation_center_rt_volume_center_z=0.,
    threshold=0.,
):
    """
    Parameters description:
    
    ray_source_distance = 400                              # <-sid float>            Distance of ray source (focal point) focal point 400mm
    camera_translation_parameter = [0., 0., 0.]            # <-t float float float>  Translation parameter of the camera
    rotation_around_xyz = [0., 0., 0.]                     # <-rx float>             Rotation around x,y,z axis in degrees
    projection_normal_position = [0, 0]                    # <-normal float float>   The 2D projection normal position [default: 0x0mm]
    rotation_center_relative_to_volume_center = [0, 0, 0]  # <-cor float float float> The centre of rotation relative to centre of volume
    threshold = 10                                          # <-threshold float>      Threshold [default: 0]
    """


    dgree_to_radius_coef = 1./180.*math.pi
    camera_translation_parameter = [camera_tx, camera_ty, camera_tz]
    rotation_around_xyz = [rotation_x*dgree_to_radius_coef, rotation_y*dgree_to_radius_coef, rotation_z*dgree_to_radius_coef]
    projection_normal_position = [projection_normal_p_x, projection_normal_p_y]
    rotation_center_relative_to_volume_center = [
        rotation_center_rt_volume_center_x, 
        rotation_center_rt_volume_center_y, 
        rotation_center_rt_volume_center_z
    ]
    
    imageOrigin = volume_lung.GetOrigin()
    imageSpacing = volume_lung.GetSpacing()
    imageRegion = volume_lung.GetBufferedRegion()
    imageSize = imageRegion.GetSize()
    imageCenter = [imageOrigin[i] + imageSpacing[i]*imageSize[i]/2.0 for i in range(3)]
    
    transform.SetTranslation(camera_translation_parameter)
    transform.SetRotation(rotation_around_xyz[0], rotation_around_xyz[1], rotation_around_xyz[2])

    center = [c + imageCenter[i] for i, c in enumerate(rotation_center_relative_to_volume_center)]
    transform.SetCenter(center)
    
    interpolator.SetTransform(transform)
    interpolator.SetThreshold(threshold)
    focalPoint = [imageCenter[0], imageCenter[1], imageCenter[2] - ray_source_distance/2.0]
    interpolator.SetFocalPoint(focalPoint)
    
    filter.SetInterpolator(interpolator)
    filter.SetTransform(transform)

    origin = [
        imageCenter[0] + projection_normal_position[0] - output_image_pixel_spacing[0]*(output_image_size[0] - 1)/2.,
        imageCenter[1] + projection_normal_position[1] - output_image_pixel_spacing[1]*(output_image_size[1] - 1)/2.,
        imageCenter[2] + imageSpacing[2]*imageSize[2]
    ]

    filter.SetOutputOrigin(origin)
    filter.Update()

    global viewer
    if viewer is None:
        viewer = view(filter.GetOutput(), mode='z')
    else:
        print("Update viewer image")
        viewer.image = filter.GetOutput()
    
    # print informations
    print("Volume image informations:")
    print("\tvolume image origin : ", imageOrigin)
    print("\tvolume image size   : ", imageSize)
    print("\tvolume image spacing: ", imageSpacing)
    print("\tvolume image center : ", imageCenter)
    print("Transform informations:")
    print("\ttranslation         : ", camera_translation_parameter)
    print("\trotation            : ", rotation_around_xyz)
    print("\tcenter               : ", center)
    print("Interpolator informations: ")
    print("\tthreshold           : ", threshold)
    print("\tfocalPoint          : ", focalPoint)
    print("Filter informations:")
    print("\toutput origin        : ", origin)
  

DigitallyReconstructedRadiograph()

slider = interactive(
    DigitallyReconstructedRadiograph, 
    ray_source_distance=(0, 800, 50),
    camera_tx=(-400, 400,10), 
    camera_ty=(-400, 400,10), 
    camera_tz=(-400, 400,10),
    rotation_x=(-45,45,1),
    rotation_y=(-45,45,1),
    rotation_z=(-45,45,1),
    projection_normal_p_x=(-100,100,1),
    projection_normal_p_y=(-100,100,1),
    rotation_center_rt_volume_center_x=(-100,100,1),
    rotation_center_rt_volume_center_y=(-100,100,1),
    rotation_center_rt_volume_center_z=(-100,100,1),
)

widgets.VBox([viewer, slider])
```

    Volume image informations:
    	volume image origin :  itkPointD3 ([-153.827, -150.352, -1434.5])
    	volume image size   :  itkSize3 ([115, 157, 129])
    	volume image spacing:  itkVectorD3 ([1.366, 1.366, 2.5])
    	volume image center :  [-75.282, -43.120999999999995, -1273.25]
    Transform informations:
    	translation         :  [0.0, 0.0, 0.0]
    	rotation            :  [0.0, 0.0, 0.0]
    	center               :  [-75.282, -43.120999999999995, -1273.25]
    Interpolator informations: 
    	threshold           :  0.0
    	focalPoint          :  [-75.282, -43.120999999999995, -1323.25]
    Filter informations:
    	output origin        :  [-416.782, -384.621, -950.75]



    VBox(children=(Viewer(geometries=[], gradient_opacity=0.22, mode='z', point_sets=[], rendered_image=<itkImageP…

结果示例如下：

![](/image/itk_example_drr.png)