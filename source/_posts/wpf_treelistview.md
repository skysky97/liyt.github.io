---
title: WPF中TreeListView控件的实现
date: 2018-02-02 17:08:32
tags: [WPF, TreeListView]
categories: WPF
---
# 说明
WPF中没有原生的TreeListView控件，本文根据MSN上的[示例代码](http://files.cnblogs.com/browser-yy/TreeListView.zip)来分析其实现方法。最终实现的效果图如下：  
![overview][]

# 实现
整个TreeListView主要包括以下几个部分：
- 折叠按钮，根据折叠展开状态变换显示效果，没有展开内容时隐藏。
- 每个单元格。
- 整个控件。

## 折叠按钮
折叠按钮使用ToggleButton控件来实现，这里通过ControlTemplate设置ToggleButton的模板。
首先创建外层的Border确定折叠按钮的边界，然后创建内层Border绘制折叠展开符号，示例中折叠状态下显示为方框中间一个＋号。
将Button的`IsChecked`属性当作触发器，来改变内层Border的背景，实现折叠展开翻转效果。
```xml
<Style x:Key="ExpandCollapseToggleStyle"
       TargetType="{x:Type ToggleButton}">
  <Setter Property="Focusable"
          Value="False"/>
  <Setter Property="Width"
          Value="19"/>
  <Setter Property="Height"
          Value="13"/>
  <Setter Property="Template">
    <Setter.Value>
      <ControlTemplate TargetType="{x:Type ToggleButton}"> <Border Width="19"

                Height="13"
                Background="Transparent">
          <Border Width="9"
                  Height="9"
                  BorderThickness="1"
                  BorderBrush="#FF7898B5"
                  CornerRadius="1"
                  SnapsToDevicePixels="true">
            <Border.Background>
              <LinearGradientBrush StartPoint="0,0"
                                   EndPoint="1,1">
                <LinearGradientBrush.GradientStops>
                  <GradientStop Color="White"
                                Offset=".2"/>
                  <GradientStop Color="#FFC0B7A6"
                                Offset="1"/>
                </LinearGradientBrush.GradientStops>
              </LinearGradientBrush>
            </Border.Background>
            <Path x:Name="ExpandPath"
                  Margin="1,1,1,1"
                  Fill="Black"
                  Data="M 0 2 L 0 3 L 2 3 L 2 5 L 3 5 L 3 3 
                        L 5 3 L 5 2 L 3 2 L 3 0 L 2 0 L 2 2 Z"/>
          </Border>
        </Border>
        <ControlTemplate.Triggers>
          <Trigger Property="IsChecked"
                   Value="True">
            <Setter Property="Data"
                    TargetName="ExpandPath"
                    Value="M 0 2 L 0 3 L 5 3 L 5 2 Z"/>
          </Trigger>
        </ControlTemplate.Triggers>
      </ControlTemplate>
    </Setter.Value>
  </Setter>
</Style>
```

[overview]: http://liyunting-blog.oss-cn-hangzhou.aliyuncs.com/TreeListView/overview.png?Expires=1517821154&OSSAccessKeyId=TMP.AQF68TJC4Ide1tJPA3AcCyX2OB4Wl804BxRGOMmj7ncAkWG3yTOzjh1rlXCKAAAwLAIUWEKetKGPUQK53jKSnv-LazIbzq0CFCCiK9ojTAVaqu-TgRf5ZPnpk3RY&Signature=84DTld5suSKIPvw1fCxWIcBj6S4%3D
