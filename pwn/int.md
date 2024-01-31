有两个文件，给了libc，这是程序里没有对应函数字符串要到libc里面找

日常 `chaecksec --file=int` 
```sh
┌──(kali㉿helloeveryone)-[~/ctf/pwn/q28_int]
└─$ checksec --file=int
[*] '/home/kali/ctf/pwn/q28_int/int'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```
有金丝雀、堆栈不可执行，没开地址随机化，got表动态加载可写

看看ida
```c
//main
int __cdecl main()
{
  setvbuf(stdout, 0, 2, 0);
  setvbuf(stdin, 0, 2, 0);
  puts("Welcome to array manager");
  sub_80485CD();
  return 0;
}

//sub_80485CD()
unsigned int sub_80485CD()
{
  int v1; // [esp+1Ch] [ebp-2Ch] BYREF
  int v2; // [esp+20h] [ebp-28h] BYREF
  int i; // [esp+24h] [ebp-24h]
  int v4; // [esp+28h] [ebp-20h]
  int s[4]; // [esp+2Ch] [ebp-1Ch] BYREF
  unsigned int v6; // [esp+3Ch] [ebp-Ch]

  v6 = __readgsdword(20u);//v6是金丝雀
  v4 = 0;
  v1 = 0;
  v2 = 0;
  memset(s, 0, sizeof(s));
  for ( i = 0; i <= 3; ++i )
  {
    puts("enter index:");
    __isoc99_scanf("%d", &v1);
    if ( v1 > 3 )//漏洞所在，只对索引进行单边检测，存在数组越界漏洞
    {
      puts("index is too big");
      exit(0);
    }
    puts("enter value:");
    __isoc99_scanf("%d", &v2);
    s[v1] = v2;
  }
  while ( v4 <= 3 )                             
    printf(byte_804880E, s[v4++]);//"0x%08x"
  return __readgsdword(0x14u) ^ v6;//检查v6是否被篡改
}
```
检查代码，发现数组越界漏洞。但索引只能向负方向增长(即栈的低地址方向)，怎么才能修改返回地址？

可以自行gdb一下，发现因为是32位程序，栈只有32位，即 `0x00000000 ~ 0xffffffff` ，漏洞数组单位是 `int` ，一个元素4字节，所以索引模一个比值 `0x100000000/4=0x40000000=1073741824` 后对应的是同一个地址！考虑到索引数据类型是 `int` ，范围是 `-2147483648 ~ 2147483647` ，这个思路完全可行！(把整个栈越过去，这真是一个疯狂的操作！)

`sub_80485CD()`栈的结构：
```
-00000048                 db ? ; undefined
-00000047                 db ? ; undefined
-00000046                 db ? ; undefined
-00000045                 db ? ; undefined
-00000044                 db ? ; undefined
-00000043                 db ? ; undefined
-00000042                 db ? ; undefined
-00000041                 db ? ; undefined
-00000040                 db ? ; undefined
-0000003F                 db ? ; undefined
-0000003E                 db ? ; undefined
-0000003D                 db ? ; undefined
-0000003C                 db ? ; undefined
-0000003B                 db ? ; undefined
-0000003A                 db ? ; undefined
-00000039                 db ? ; undefined
-00000038                 db ? ; undefined
-00000037                 db ? ; undefined
-00000036                 db ? ; undefined
-00000035                 db ? ; undefined
-00000034                 db ? ; undefined
-00000033                 db ? ; undefined
-00000032                 db ? ; undefined
-00000031                 db ? ; undefined
-00000030                 db ? ; undefined
-0000002F                 db ? ; undefined
-0000002E                 db ? ; undefined
-0000002D                 db ? ; undefined
-0000002C var_2C          dd ?
-00000028 var_28          dd ?
-00000024 var_24          dd ?
-00000020 var_20          dd ?
-0000001C s               dd 4 dup(?)
-0000000C var_C           dd ?
-00000008                 db ? ; undefined
-00000007                 db ? ; undefined
-00000006                 db ? ; undefined
-00000005                 db ? ; undefined
-00000004                 db ? ; undefined
-00000003                 db ? ; undefined
-00000002                 db ? ; undefined
-00000001                 db ? ; undefined
+00000000  s              db 4 dup(?)
+00000004  r              db 4 dup(?)
+00000008
+00000008 ; end of stack variables
```
可见数组首地址 `-0000001C` 到返回地址 `+00000004` 有 `0x20=32` 个字节，那么索引就是 `8` ，返回地址用数组索引就可以表示成 `8 - 0x40000000 = 8 - 1073741824 = -1073741816` 

事实上，现在我们就可以对栈上任意写，可以通过修改函数返回地址劫持程序流程

题目中没有给利用函数，没有/bin/sh字符串，所以需要用到题目给的libc中的system函数和/bin/sh字符串

又由linux的延迟绑定机制，只有在先前程序中运行过的库函数才会在got表中储存实际地址，所以这里可以选择泄露 `puts` 地址

可构造栈如下

```
返回地址  puts_plt
         ret_address
         puts_got
```
这样 `put` 的真正地址（即got表中的值）就作为put的参数被打印出来


