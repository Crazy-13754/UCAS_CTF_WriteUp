让人不知所措的众多题目之一。
在 Burp Suite 中通过内嵌浏览器访问，发现相应中有 Password。

![alt Brup](BabyHttp1.png "Brup Suite")

取解码后花括号中的内容提交，发现会有提示。

![alt 提示](BabyHttp2.png "提示")

随后，你就明白要干什么并写脚本来变成一个“快人”。

```python
# coding:utf8
import requests
import base64
url = "http://124.16.75.117:51001/" # 目标URL
s = requests.Session() # 获取 Session
response = s.get(url) # 打开链接
head = response.headers # 获取响应头
print(head)
#flag = base64.b64decode(head['Password']).split(':')[1] # 获取相应头中的Flag
flag = base64.b64decode(head['Password']).decode('utf-8')
print(flag)
extracted_flag = flag[5:-1]  # 取中括号中的内容
print(extracted_flag)
postData = {'password': extracted_flag} # 构造 Post 请求体
result = s.post(url=url, data=postData) # 利用 Post 方式发送请求
# (注意要在同一个 Session 中 , 有的时候还需要设置 Cookies , 但是此题不需要)
print(result.text) # 打印响应内容
```
此题服务器通过 Cookie 中的 PHPSESSID 判断是否为同一个浏览器的提交。