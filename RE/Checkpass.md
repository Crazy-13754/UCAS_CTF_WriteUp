

*求解出密码。flag格式为NeSE{...}*

最头痛的一集。

先用IDA打开看看，发现事情有点复杂，不仅函数非常多，而且没有任何调试信息。

从字符串入手，发现一个有趣的字符串

```
.rodata:0000000000039DC8	00000012	C	Invalid password\n
```

进入字符串所在地址，用Xrefs查找调用，发现`sub_5960`非常可疑。

经过亿点点观察研究和瞎猜，把该函数标注如下（节选）：

```c
void __noreturn check()
{
  __int64 input_raw; // rbx
  char *input_raw_end; // rax
  __int128 *input_raw_2; // rbx
  __int64 temp; // rax
  __int128 *v4; // rax MAPDST
  __int64 v5; // rcx
  __int128 vck; // xmm0
  __int128 v7; // xmm0
  __int128 input_msg; // [rsp+0h] [rbp-128h] BYREF
  __int128 v9; // [rsp+10h] [rbp-118h]
  __int64 *v10; // [rsp+20h] [rbp-108h]
  __int64 v11; // [rsp+28h] [rbp-100h]
  char v12; // [rsp+33h] [rbp-F5h]
  ...
  char v32; // [rsp+47h] [rbp-E1h]
  char text[32]; // [rsp+48h] [rbp-E0h] BYREF
  __int128 v35; // [rsp+70h] [rbp-B8h]
  __int64 v36[2]; // [rsp+80h] [rbp-A8h] BYREF
  __int128 v37; // [rsp+90h] [rbp-98h]
  __int64 input_1; // [rsp+A0h] [rbp-88h] BYREF
  __int64 argc; // [rsp+B0h] [rbp-78h]
  __int64 dst[2]; // [rsp+B8h] [rbp-70h] BYREF
  __int128 v41; // [rsp+C8h] [rbp-60h]
  __int64 v42[2]; // [rsp+D8h] [rbp-50h] BYREF
  __int128 v43; // [rsp+E8h] [rbp-40h]

  sub_55D60821F000((__int128 *)v36);
  v9 = v37;
  input_msg = *(_OWORD *)v36;
  get_input((__int64)&input_1, &input_msg);
  if ( argc == 2 )
  {
    if ( *(_QWORD *)(input_1 + 40) == 41LL )
    {
      input_raw = *(_QWORD *)(input_1 + 24);
      if ( (_UNKNOWN *)input_raw == &aPicoCTF || *(_QWORD *)input_raw == 0x3535357B4553654ELL )// NeSE{555
      {
        input_raw_end = (char *)(input_raw + 40);
        if ( (_UNKNOWN *)(input_raw + 40) == &unk_55D608239D94 || *input_raw_end == '}' )
        {
          if ( *(char *)(input_raw + 8) > -65 )
          {
            input_raw_2 = (__int128 *)(input_raw + 8);
            if ( *input_raw_end > -65 )
            {
              temp = malloc_X(0x20uLL, 1uLL);
              if ( !temp )
                EXIT_X(32LL);
              v4 = (__int128 *)temp;
              v35 = Secret;
              resize_X((__int64)&v4, 0LL, 0x20uLL);
              v5 = *((_QWORD *)&v35 + 1);
              vck = *input_raw_2;
              *(__int128 *)((char *)v4 + *((_QWORD *)&v35 + 1) + 16) = input_raw_2[1];
              *(__int128 *)((char *)v4 + v5) = vck;
              *((_QWORD *)&v35 + 1) = v5 + 32;
              if ( !v5 )
              {
                v7 = *v4;
                v9 = v4[1];
                input_msg = v7;
                transform((__int64)dst, (unsigned __int8 *)&input_msg, 0LL);
                v9 = v41;
                input_msg = *(_OWORD *)dst;
                transform((__int64)v42, (unsigned __int8 *)&input_msg, 1LL);
                v9 = v43;
                input_msg = *(_OWORD *)v42;
                transform((__int64)v36, (unsigned __int8 *)&input_msg, 2LL);
                v9 = v37;
                input_msg = *(_OWORD *)v36;
                transform((__int64)text, (unsigned __int8 *)&input_msg, 3LL);
                ...
```


得知flag是`PicoCTF{`或`NeSE{555`开头，`}`结尾，总共有40个字符，故中间需要填入32个字符。

该函数的后半部分是一堆的`if`嵌套，显得十分复杂，但是稍微观察发现其实际作用只是将一段字节（称为`FLAG`）与`text`比对，如果全部相同则输出`success`

通过动态调试，发现`input_msg`就是输入字符串中间的32位字符。

输入的32位字符通过`transform`函数进行不同的4次变换后得到了`text`，接下来研究该函数。

又经过亿点点分析，把该函数标注如下：

```c
char __fastcall transform(__int64 dst, unsigned __int8 *src, __int64 flag)
{
  __int64 base; // rdx
  unsigned __int64 key; // rax
  char result; // al
  char re[32]; // [rsp+8h] [rbp-20h]

  base = flag << 8;
  re[0] = tran_Lib[base + *src];
  re[1] = tran_Lib[base + src[1]];
  ...
  ...
  re[31] = tran_Lib[base + src[31]];
  
  *(_OWORD *)(dst + 16) = 0LL;
  *(_OWORD *)dst = 0LL;
  key = *(_QWORD *)&tran_Lib_2[base + 64];
  if ( key > 0x1F )
    goto error;
  *(_BYTE *)(dst + 8) = re[key];
  key = *(_QWORD *)&tran_Lib_2[base + 200];
  if ( key > 0x1F )
    goto error;
  *(_BYTE *)(dst + 25) = re[key];
  ...
  ...
  key = *(_QWORD *)&tran_Lib_2[base + 56];
  if ( key > 0x1F )
    goto error;
  *(_BYTE *)(dst + 7) = re[key];
  key = *(_QWORD *)&tran_Lib_2[base + 88];
  if ( key > 0x1F
    || (*(_BYTE *)(dst + 11) = re[key], key = *(_QWORD *)&tran_Lib_2[base + 184], key > 0x1F)
    || (*(_BYTE *)(dst + 23) = re[key], key = *(_QWORD *)&tran_Lib_2[base + 40], key > 0x1F)
    || (*(_BYTE *)(dst + 5) = re[key], key = *(_QWORD *)&tran_Lib_2[base + 144], key >= 0x20) )
  {
error:
    exception_length(key, 32LL, (__int64)&off_55D6084481F8);
  }
  result = re[key];
  *(_BYTE *)(dst + 18) = result;
  return result;
}
```

综上，将加密逻辑翻译成简单的C++

```cpp
#include <cstdio>
#include <cstdlib>
#include <cstring>
#define F(n) for(int i=0;i<n;++i)

unsigned char FLAG[32]={...};
unsigned char tran_Lib[1024]={...};
unsigned char tran_Lib_2[128]={...};

void transform(unsigned char* dst,unsigned char* src, int flag)
{
  int base;
  unsigned char charcs[32];
  base = flag << 8;
  F(32) {
    charcs[i] = tran_Lib[base + src[i]];
  }
  base = flag << 5;
  F(32) {
    dst[i] = charcs[tran_Lib_2[base + i]];
  }
}

void encrypt(unsigned char* dst,unsigned char* src){
  unsigned char t1[32],t2[32],t3[32];
  transform(t1,src,0);
  transform(t2,t1,1);
  transform(t3,t2,2);
  transform(dst,t3,3);
}

void check() {
  unsigned char input[100], text[32];
  scanf("%s", input);
  encrypt(text, input);
  F(32) {
    if (text[i] != FLAG[i]) {
      puts("WRONG");
      return;
    }
  }
  puts("CORRET");
}
```

倒立着写出解密逻辑

```cpp
void Invtransform(unsigned char* src, unsigned char* dst, int flag) {
  unsigned char charcs[32];
  int base = flag << 5;
  F(32) {
    charcs[tran_Lib_2[base + i]] = dst[i];
  }
  base = flag << 8;
  F(32) {
    for (int c = 0;c < 256;++c) {
      if (charcs[i] == tran_Lib[base + c]) {
        src[i] = c;
        break;
      }
    }
  }
}

void decrypt(unsigned char* src, unsigned char* dst) {
  unsigned char t1[32], t2[32], t3[32];
  Invtransform(t3, dst, 3);
  Invtransform(t2, t3, 2);
  Invtransform(t1, t2, 1);
  Invtransform(src, t1, 0);
}
```

对`FLAG`进行`decrypt`后即可得到flag的中间32字符，从而得到flag
