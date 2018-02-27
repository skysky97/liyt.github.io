---
title: WPF MvvmLight安装使用
date: 2018-02-25 10:17:27
tags: [WPF, MVVM, MvvmLight]
categories: WPF
---
# 安装
NuGet搜索**MvvmLight**，会搜到两个包，MvvmLight和MvvmLightLibs。前者会创建额外的解决方案结构文件，我们选择前者安装。备注：在使用过程中发现自动安装的CommonServiceLocator包为v2.0.2版本，此版本疑似不包含Microsoft.Practices.ServiceLocation引用集，切换到v1.3.0版本解决，原因未知。  

# 使用
安装完成后自动创建了一个MainViewModel和ViewModelLocator。ViewModeLocator的作用相当于注册ViewModel，使得View和ViewModel关联起来，替代之前在代码中手动指定DataContext的做法。具体原理还不清楚，待后续学习（关键字：IOC, MEF, ViewModelLocator)。  

## 属性
属性绑定需要实现INotifyPropertyChanged接口，MvvmLight中通过RaisePropertyChanged(PropertyName)方法实现。
```csharp
private string _Title;
public string Title
{
    get { return _Title; }
    set { _Title = value; RaisePropertyChanged("Title"); }
}
```

## 命令
MvvmLight中通过RelayCommand对象表示命令.
```csharp
public MainViewModel()
{
    Title = "Hello World";
    Test = new RelayCommand(OnTest);
}

private string _Title;
public string Title
{
    get { return _Title; }
    set { _Title = value; RaisePropertyChanged("Title"); }
}

public RelayCommand Test { get; set; }

public void OnTest()
{
    Title = "Test OK";
}
```
`OnTest`是实际的命令操作，通过RelayCommand(OnTest)创建一个实例。  

## 上下文
前面在MainViewModel中创建了一个Title属性和Test命令，但ViewModel和View还没有关联。通常我们可以通过代码来建立这种关联：  
ViewModelLocator.cs  
```csharp
public ViewModelLocator()
{
    ServiceLocator.SetLocatorProvider(() => SimpleIoc.Default);

    SimpleIoc.Default.Register<MainViewModel>();
}

public MainViewModel Main
{
    get
    {
        return ServiceLocator.Current.GetInstance<MainViewModel>();
    }
}
```
先注册了MainViewModel，然后返回一个实例Main，Main的生命周期由ServiceLocator管理。  
  
在View中将Main绑定到DataContext。  
MainWindow.xaml  
```xml
<Window.DataContext>
	<Binding Path="Main"  Source="{StaticResource Locator}"/>
</Window.DataContext>
```

## 测试
创建一个TextBox与Title关联，用于输入；创建一个TextBlock与Title关联显示；创建一个Button绑定Test命令。
```xml
<Grid>
	<StackPanel Orientation="Vertical">
		<TextBox Text="{Binding Path=Title, UpdateSourceTrigger=PropertyChanged}" />
		<TextBlock Text="{Binding Title}"/>
		<Button Content="测试" Command="{Binding Test}"/>
	</StackPanel>
</Grid>
```


