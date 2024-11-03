
## C # 的介绍与Mono的诞生

1. 2000年发布的，微软推出一个专为 Windows 系统设计的开发平台和运行时环境——.NET Framework 。C# 是 .NET 的主要语言，在2005年 .NET Framework 2.0发布，引入了重要了更新如泛型的支持。
2. 2004 年Mono 发布，开源项目 Mono 推出为 .NET 提供了一个跨平台实现，使 C# 可以在 Linux 和 macOS 上运行，填补了当时 .NET 仅支持 Windows 的局限。
3. 2016年微软收购 Mono的母公司 Xamarin
4. 2016 年 .**NET Core 1.0**发布，提供了一个轻量级的跨平台版本，支持 Windows 、Linux 和 macOS。

## C # 本身 与 IL(Intermediate Language)的关系

C# 编译后生成中间语言（IL），IL 是一种平台无关的中间语言，在运行时通过 .NET 的公共语言运行时（CLR）转换成机器码。

## C# 在Unity引擎

Untiy大概在2008年引入了Mono这一 .NET 开源运行时实现 ，支持了C#语言作为脚本开发，自此C#就是Unity的主要脚本语言了。
Mono 在被微软收购前版本的更新较慢，导致 Unity 的 C# 语言特性和 .NET 功能落后主流版本。

# Mono和IL2CPP

IL2CPP是AOT编译，C#代码转换成C++代码后编译成原生机器码；Mono是解释执行，JIT编译（这在IOS上不支持），为什么不支持，[这篇文章有详细的说明](https://www.cnblogs.com/murongxiaopifu/p/4278947.html)。

这里简单介绍下
 **首先关于机器码**：机器码是计算机代码最低级的表示形式，比汇编还要低级的语言，对应于CPU可以执行的指令。**
 
![[Pasted image 20241031110434.png]]

那么其实，我们是可以在运行时将机器码映射到某一块内存，然后根据获取到的函数指针，去调用执行这段内存机器码的。

问题就出自分配内存的权限上：
![[Pasted image 20241031111043.png]]

**IOS封了内存（或者堆）的可执行权限，相当于变相的封锁了JIT这种编译方式。**
# 性能

![[Pasted image 20241031103342.png]]
# 例子

![[Pasted image 20241031101759.png]]
转换成IL2CPP ：
![[Pasted image 20241031101809.png]]
![[Pasted image 20241031102441.png]]

 # C#转换 C++步骤
 
 1. Patching：这里的patching是指预处理，处理不支持AOT的代码
	![[Pasted image 20241031101325.png]]
  2. Stripping:通过静态分析，移除掉没有使用的代码
	  ![[Pasted image 20241031101631.png]]
3. Metadata：添加源信息![[Pasted image 20241031102236.png]]
4. 异常：例如会显式检查是否为null
	![[Pasted image 20241031103041.png]]
