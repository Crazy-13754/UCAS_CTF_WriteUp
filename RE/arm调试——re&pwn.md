工具篇：[ubuntu20.04 PWN（含x86、ARM、MIPS）环境搭建](https://blog.csdn.net/qq_41202237/article/details/118188924)

调试篇：[arm pwn 入门](https://blingblingxuanxuan.github.io/2021/01/27/arm-pwn-start/)

调试例：

```sh
qemu-aarch64 -g 9999 ./test
#使用qemu将程序运行并用端口9999连接gdb
```

然后新建终端，开启gdb调试

```sh
gdb-multiarch ./test
pwndbg> set architecture aarch64
pwndbg> set endian little
pwndbg> target remote:9999
#设置调试目标架构、大/小端储存，连接端口
```

然后，即可以开始调试！

编译时要有 `-g` 选项以便程序可以调试
