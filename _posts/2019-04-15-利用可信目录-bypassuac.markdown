---
layout: post
title:  "通过模拟可信目录绕过UAC的利用分析"
date:   2019-04-15 17:48:13
categories: jekyll update
permalink: /archivers/模拟可信目录绕过UAC
---

## 通过模拟可信目录绕过UAC的利用分析

### 0x00 前言


从3gstudent博客学习到的技巧，自己来复现一遍，顺便熟悉一下markdown语法，通过模拟可信目录来绕过UAC，主要是参考3gstudent的博客内容进行复现。

文章地址：https://3gstudent.github.io/3gstudent.github.io/%E9%80%9A%E8%BF%87%E6%A8%A1%E6%8B%9F%E5%8F%AF%E4%BF%A1%E7%9B%AE%E5%BD%95%E7%BB%95%E8%BF%87UAC%E7%9A%84%E5%88%A9%E7%94%A8%E5%88%86%E6%9E%90/

### 0x01 简介


* 原理介绍
* 实现细节
* 实际测试
* 利用分析

### 0x02 原理介绍

#### 1、Long UNC
使用long UNC可以进行文件名欺骗。
例如：

```
type putty.exe > "\\?\C:\Windows\System32\calc.exe 
```

type 命令的功能是将一个文件的内容 输出为另外一个文件，相当于拷贝。

![](https://raw.githubusercontent.com/xxxxxyyyy/blog_image/master/2019-04/01.jpg)
上面的命令的结果就会在对应位置生成一个putty.exe,名字是"calc.exe ",但是在查看属性时，大小只有27KB，同真实的calc.exe文件属性一样，猜测系统去识别属性时是去查看了真实的calc.exe.

用这样同样的方法也可以建立文件

例如：
```
md "\\?\c:\windows "
```
新创建的文件夹，如果直接访问的话，会导向真实的windows文件夹
如下图：
![](https://raw.githubusercontent.com/xxxxxyyyy/blog_image/master/2019-04/02.jpg)
#### 2、默认能够绕过UAC的文件
需要满足一下三个条件：

* 程序配置为自动提升权限，以管理员权限执行
* 程序包含签名
* 从受信任的目录("c:\windows\system32")去执行

#### 3、普通用户权限能够在磁盘根目录创建文件夹
普通用户权限需要能够在c盘下创建文件夹。

#### 4、dll劫持
exe 程序在启动过程中需要加载dll，加载的这个dll是按照一定顺序去搜索的，默认先搜索exe程序同级目录
综上，满足绕过了UAC的所有条件
实现思路如下：

1. 找到一个默认能够绕过UAC的文件，例如c:\windows\system32\winsat.exe。
2. 使用Long UNC创建一个特殊的文件夹“c:\windows \”，并将winsat.exe复制到该目录。
3. 执行winsat.exe，记录它启动的过程，看它会加载哪些dll，例如winnm.dll。
4. 编写payload.dll,指定导出函数跟c:\windows\system32\winmm.dll相同，并命名为“c:\windows \system32\winmm.dll”。
5. 执行"c:\windows \system32\winsat.exe"，将在动绕过UAC，加载"c:\windows \system32\winmm.dll",来执行我们的payload。

### 0x03 实现细节
* * *
##### 1、寻找可供利用的exe
这些文件的特征之一是manifest中的autoElevate属性为true,可以借助powershell实现自动化搜索，参考工具：
https://github.com/g3rzi/Manifesto
原博客使用的GUI版本，这里我试试ps的：
![](https://raw.githubusercontent.com/xxxxxyyyy/blog_image/master/2019-04/03.jpg)
可以看到能找到 autoElevate 属性为true的文件还是比较多。

##### 2、使用Long UNC 创建一个特殊的文件夹“c:\windows \”
C++的实现代码如下：
```
CreateDirectoryW(L"\\\\?\\C:\\Windows \\", 0);
```
通过命令行实现的命令如下：
```
md "\\?\c:\windows \system32\"
```

##### 3、记录启动过程，寻找启动时加载的dll
这个步骤我脑子犯抽了，应该建立好LONG UNC 目录后，（“c:\windows \system32\”）,再把你上个步骤发现的含有autoElevate属性为true的程序都拷贝到该目录下来，一个个的去执行，然后去监控是否在当前目录("c:\windows \system32\")没有找到的dll，说明这个程序会在当前目录去寻找dll，那我们就可以把我们的dll放到该目录，去达成劫持效果哦。
下面是我执行“fodhelper.exe”的结果，可以看到有多个dll在当前目录无法找到，我们就来利用PROPSYS.dll吧。
![](https://raw.githubusercontent.com/xxxxxyyyy/blog_image/master/2019-04/04.jpg)

#### 4、生成自己的dll并进行替换
使用ExportsToC++ 来自动导出dll函数表并生成C++代码。
注意：ExportsToC++ 启动时时候需要在 Visual Tools Command Prompt 里面去启动，不然会报错。

使用ExportsToC++ 打开(c:\windows\system32\PROPSYS.DLL),然后选择Convert,并填入c:\windows\system32\propsys.dll:

![](https://raw.githubusercontent.com/xxxxxyyyy/blog_image/master/2019-04/05.jpg)

在vs中新建dll项目，将上面生成的C++代码复制进去
注意：如果你要利用的操作系统是64位的，你需要编译64位的dll，我开始没注意这个问题，又卡了好久，多亏 @3gstudent 师傅耐心解答。
![](https://raw.githubusercontent.com/xxxxxyyyy/blog_image/master/2019-04/06.jpg)


然后将该dll生成出来更名为 PROPSYS.dll 放入"c:\windows \system32\"文件夹
![](https://raw.githubusercontent.com/xxxxxyyyy/blog_image/master/2019-04/07.jpg)

输入全路径进行启动：
![](https://raw.githubusercontent.com/xxxxxyyyy/blog_image/master/2019-04/08.jpg)

启动了多个cmd.exe，权限直接是system的。

### 0x03 总结

可以看到该方法还是很好用的，并且存在多个exe可以去利用，虽然只是简单的去复现，但是还是能学到不少东西。特别是中间走了弯路的时候(自己的dll一直不执行)，自己去尝试其它的方法，使用Ghidra去查看该exe会调用dll中的哪些函数，然后自己去简单实现这些函数，这个方法当然也是可以的，动手的过程当中还是需要耐心。
