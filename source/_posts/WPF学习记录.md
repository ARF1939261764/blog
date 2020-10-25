---
title: WPF学习记录
typora-copy-images-to: ./WPF学习记录
typora-root-url: ./WPF学习记录
categories:
  - - C#
    - WPF
tags:
  - C#
  - WPF
abbrlink: 16649
date: 2020-10-18 10:35:15
---



&emsp;&emsp;以前花了不少时间看C#、WPF的书，但是写代码一直没有感觉，总是以写单片机那种裸机程序的思路去写C#代码，导致代码结构、思路总是乱糟糟的，不过最近用WPF做了一个小程序，这种情况似乎有所改善，这个小程序虽然小，但是这次有了一点写面向对象程序的感觉，同时发现以前学的好多C#语法都忘记了，所以趁现在复习一下，记录一下，方便下次回(fuzhi)顾(zhantie)。

![64559312_p0](64559312_p0.jpg)

<!-- more -->

&emsp;&emsp;

# ListBox与页面绑定

![image-20201018112048039](/image-20201018112048039.png)

&emsp;&emsp;现在有一个需求，如上图，上图左边是一个ListBox，右边是一个Page，要求选择ListBox中的选项时，右边的页面也切换到对应的页面。

代码如下：

**xaml:**

```xaml
<Window x:Class="WpfApp1.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        xmlns:local="clr-namespace:WpfApp1"
        mc:Ignorable="d"
        Title="MainWindow" Height="450" Width="800">
    <Grid>
        <Grid.ColumnDefinitions>
            <ColumnDefinition/>
            <ColumnDefinition/>
        </Grid.ColumnDefinitions>
        <ListBox Name="ListBox_PageSel" Margin="10" Grid.Column="0" SelectedIndex="0"/>
        <Frame x:Name="Frame_Page" BorderBrush="Gray" BorderThickness="0.5" Margin="10" Content="{Binding ElementName=ListBox_PageSel,Path=SelectedItem.Page}" Grid.Column="1" NavigationUIVisibility="Hidden"/>
    </Grid>
</Window>
```

**c#:**

``` c#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Data;
using System.Windows.Documents;
using System.Windows.Input;
using System.Windows.Media;
using System.Windows.Media.Imaging;
using System.Windows.Navigation;
using System.Windows.Shapes;

namespace WpfApp1
{
    class NamePageMap
    {
        public string Name {get;set;}
        public Page Page { get; set; }
    }
    /// <summary>
    /// MainWindow.xaml 的交互逻辑
    /// </summary>
    public partial class MainWindow : Window
    {
        List<NamePageMap> NamePageMapTab;
        public MainWindow()
        {
            InitializeComponent();
            NamePageMapTab = new List<NamePageMap>() { 
                new NamePageMap(){Name = "页面0" , Page = new Page0()},
                new NamePageMap(){Name = "页面1" , Page = new Page1()},
                new NamePageMap(){Name = "页面2" , Page = new Page2()}
            };
            ListBox_PageSel.ItemsSource = NamePageMapTab;
            ListBox_PageSel.DisplayMemberPath = "Page";
        }
    }
}
```

上面还有三个Page的代码没有列出来。

思路如下：

**声明一个类`NamePageMap`**

```C#
class NamePageMap
{
    public string Name {get;set;}
    public Page Page { get; set; }
}
```
&emsp;&emsp;里面有两个成员，一个`Name`，一个`Page`，表示了`Name`与`Page`的对应关系

**在主窗口类中定义一个集合`NamePageMapTab`作为`名称-Page映射表`**

```C#
List<NamePageMap> NamePageMapTab;
```

&emsp;&emsp;这个集合表示了ListBox中所有可选项。

**在构造函数中初始化该集合**

```C#
NamePageMapTab = new List<NamePageMap>() { 
                new NamePageMap(){Name = "页面0" , Page = new Page0()},
                new NamePageMap(){Name = "页面1" , Page = new Page1()},
                new NamePageMap(){Name = "页面2" , Page = new Page2()}
            };
```

&emsp;&emsp;上面的代码使用了集合初始化器来初始化集合。

**将ListBox的ItemsSource指向到NamePageMapTab**

```c#
ListBox_PageSel.ItemsSource = NamePageMapTab;
ListBox_PageSel.DisplayMemberPath = "Page";
```

&emsp;&emsp;第一行代码执行完，ListBox便与`名称-Page映射表`绑定了，但是NamePageMapTab中的每个元素类型是`NamePageMap`，ListBox怎么知道要显示哪一个呢，所以还需要第二行代码，第二行代码则指定了ListBox在每个Item中需要显示的是那个成员。

&emsp;&emsp;至此，ListBox便可以将选项显示在List中了，但是还需要完成页面与选项之间的对应，这部分不需要C#代码，只需要在xaml中设置绑定就好了。

**将Page与ListBox的选项绑定**

```xaml
<Frame x:Name="Frame_Page" BorderBrush="Gray" BorderThickness="0.5" Margin="10" Content="{Binding ElementName=ListBox_PageSel,Path=SelectedItem.Page}" Grid.Column="1" NavigationUIVisibility="Hidden"/>
```

&emsp;&emsp;上面的标签首先定义了一个`Frame`，这个`Frame`将作为`Page`的容器，而容器的内容则是`Frame.Content`属性，需要在`Frame`中显示`Page`时，只需要将对应的`Page`赋值给`Frame.Content`属性即可，这里没有直接赋值，而是用来绑定的方法来实现。

&emsp;&emsp;`Content="{Binding ElementName=ListBox_PageSel,Path=SelectedItem.Page}"`中，`ElementName`指定了绑定的控件，`Path`则指定了绑定控件的哪一个属性。

**总结**

&emsp;&emsp;这里利用集合、绑定，使代码结构变得清晰，除了上述的用法，还可以用于`ComboBox`与各种值的绑定。



