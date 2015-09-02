# 嵌入式Linux的内核镜像生成过程
最近读了《embedded linux primer》，里面讲到了Linux内核镜像的生成过程。
感觉在这方面算是讲的比较好的。在这里翻译一下。
## 顶层目录的vmlinux
配置好交叉编译环境后，就可以以开始准备编译内核了。

首先需要编译生成内核头文件，然后开始编译内核。内核编译完成后，会在
顶层目录生成vmlinux ELF文件。

这个vmlinux文件包含整个内核代码，包括注释，调试符号信息等。
## piggy.o
piggy.o是包含经过压缩的内核代码的object文件。piggy.o主要是为了方便与其他object文件
链接生成最终Linux内核镜像文件。

生成piggy.o文件主要包含：

- 利用objcopy去掉vmlinux的一些辅助信息，生成镜像文件Image。

- 利用gzip将Image压缩成piggy.gz。

- 利用asm编译piggy.gzip.S生成piggy.o。

到此内核二进制镜像制作完成。
## Bootstrap Loader
许多CPU架构都设计成通过两个阶段来加载Linux内核镜像。第一阶段为BootLoader,
第二阶段为Bootstrap Loader。每个阶段都有各自的设计目的。

Bootstrap Loader主要提供检查内核镜像完整性，解压内核镜像，内核镜像重定位功能。
Bootstrap Loader主要包括：

+ misc.o 内核解压，重定位相关代码。
+ head.o CPU启动相关底层代码。配置cache，建立C运行环境等。
+ 其他.o  

## 另一个vmlinux
将Bootstrap Loader和piggy.o链接成vmlinux。这个vmlinux在/arch/arm/boot/compressed/
目录下。
## zImage
使用objcpoy将刚刚提到的vmlinux打包成Boot Loader需要的zImage文件。
## 结束
把之前的过程总结如下图：

![](http://img.blog.csdn.net/20150902144723646)


