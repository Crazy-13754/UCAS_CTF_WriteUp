题目直接给了源代码，非常方便快捷。本题是给出了输出，应该是需要反推输入，进行解密。

首先看看输入格式要求：
***"Wrong format 5"*** 要求输入行数不超过5行，且总字符数为9245。
***"Wrong format 1/0/"*** 要求输入的字符只能是‘0’、‘1’、空格、换行符、回车符。
***"Wrong Format 6"*** 要求每六个非空格字符使用空格隔开。

按照以上要求，可以写出一个能够产生合法输入的程序
```cpp
#include <cstdio>
using namespace std;
int main(){
    int tot=264;
    int ln=5;
    FILE* f=fopen("./input.txt","w");
    for(int l=0;l<ln;++l){
        for(int i=0;i<tot;++i){
            if (i > 0)
                fprintf(f, " ");
            for (int j = 0;j < 6;++j) {
                fprintf(f, "%d",j&1);
            }
        }
        fprintf(f,"\n");
    }
    fclose(f);
    return 0;
}
```

一个合法的输入类似这样，其中每行有264个6位的01串：
```
000000 000000 000000 ...
000000 000000 000000 ...
000000 000000 000000 ...
000000 000000 000000 ...
000000 000000 000000 ...
```


接下来研究加密逻辑，将主要部分的powershell代码手动翻译成C++代码。
```cpp
#include <cstdio>
#include <cstring>
#define F(n) for(int i=0;i<n;++i)
using namespace std;
#define N 264
typedef long long ll;
struct Block{
    char vals[5];//每个char存储6位的01串
};//Block存储一列的5个6位01串

#define NL 30
ll Scramble(const Block& block, int seed){//类似哈希表的插入
    char bm[NL];
    F(NL)bm[i]=2;//blank
    int raw=0;
    F(5){
        raw<<=6;
        raw|=block.vals[i]&63;
    }
    F(NL){
        int y=(i*seed)%NL;//0,14,28,12,26
        char n=bm[y];
        while (n!=2)
        {
            y=(y+1)%NL;
            n=bm[y];
        }
        if(raw&(1<<(NL-1-i))){
            n=3;
        }
        else{
            n=0;
        }
        bm[y]=n;
    }
    ll re=0;
    F(NL){
        re<<=2;
        re|=bm[i];
    }
    return re;
}
int main(){
    int seeds[N], randoms[N];
    Block blocks[N];
    memset(blocks,21,sizeof(blocks));//这里可以手动设置初始值
    F(N){
        seeds[i]=((i+1)*127)%500;
    }
    F(N){
        randoms[i] = ((((i+1) * 327) % 681) + 344) % 313;
    }
    ll result=0;
    F(N){
        ll fun=Scramble(blocks[i],seeds[i]);
        result^=fun^randoms[i];
        printf("%lld\n",result);
    }
    return 0;
```

可以发现，`seeds`、`randoms`是固定的，而`Scramble`函数类似哈希表的插入，实现类似洗牌的效果，可以写出解密函数
```cpp
Block InvScramble(ll fun, int seed){
    char bm[NL],used[NL];
    F(NL){
        bm[NL-1-i]=fun&3;
        fun>>=2;
    }
    F(NL)used[i]=0;
    int raw=0;
    F(NL){
        int y=(i*seed)%NL;
        while(used[y]){
            y=(y+1)%NL;
        }
        used[y]=1;
        raw<<=1;
        if(bm[y])raw|=1;
    }
    Block re;
    F(5){
        re.vals[4-i]=raw&63;
        raw>>=6;
    }
    return re;
}
```

这个函数利用当时的`seed`和`Scramble`返回的明文`fun`计算出密文`block`。

解密的主程序如下
```cpp
void decrypt(){
    FILE* f1 = fopen("output0.txt","r");
    FILE* f2 = fopen("input0.txt", "w");
    ll tmp=0;
    F(N){
        ll x;
        fscanf(f1,"%lld",&x);
        ll fun=x^tmp^randoms[i];
        blocks[i] = InvScramble(fun, seeds[i]);
        tmp^=randoms[i]^fun;
        }
    for(int y=0;y<5;++y){
        for(int x=0;x<N;++x){
            for(int i=0;i<6;++i){
                fprintf(f2,"%d", (blocks[x].vals[y]&(1<<(5-i)))?1:0);
            }
            printf("%c", blocks[x].vals[y]);
            fprintf(f2, " ");
        }
        fprintf(f2,"\n");
        printf("\n");
    }
    fclose(f1);
    fclose(f2);
}
```

得到原始输入
```
100101 101010 100101 100101 101010 101010 ...
100110 101100 100110 100110 101100 101100 ...
100111 101110 100111 100111 101110 101110 ...
101000 110000 101000 101000 110000 110000 ...
101001 110010 101001 101001 110010 110010 ...
```

如果把6位的01串看做一个字符，可以进行ascii解码得到如下字符串
```
%*%%***%%**%%*%*%*%*%%**%*%%%*%*%****%**%%**%% ...
&,&&,,,&&,,&&,&,&,&,&&,,&,&&&,&,&,,,,&,,&&,,&& ...
'.''...''..''.'.'.'.''..'.'''.'.'....'..''..'' ...
(0((000((00((0(0(0(0((00(0(((0(0(0000(00((00(( ...
)2))222))22))2)2)2)2))22)2)))2)2)2222)22))22)) ...
```

可以发现这是5行相同规律的字符串，每行均只由两种字符组成。

稍微探索发现，取其中一行，替换字符得到
```
01001110011001010101001101000101011110110011000 ...
```

再利用(CyberChef)解码成字符串即得到flag。