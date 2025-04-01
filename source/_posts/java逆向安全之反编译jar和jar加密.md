---
layout: 'title:'
title: java逆向安全之反编译jar和jar加密
date: 2022-06-09 15:04:31
categories:
- java
tags:
- blog
---

反编译class

```
@echo off
echo %1
set clasPath=%1
for /f "tokens=4* delims=\" %%a in ("%clasPath%") do (
    ::输出第一个分段(令牌)
    echo %%a
	java -jar .\tools\cfr-0.152.jar %1
	java -jar .\tools\cfr-0.152.jar %1 > %%a.java
)


::if "%1"=="" ( echo Please enter [class file name] or [jar package path and outputdir path] ) ^
::else if "%2"=="" (java -jar .\tools\cfr-0.152.jar %1) ^
::else ( java -jar .\tools\cfr-0.152.jar %1 --outputdir %2 ) 


pause
```

反编译Jar

```
@echo off
java -jar .\tools\cfr-0.152.jar %1 --outputdir ./source_code
pause
```

加密jar

```
@echo off
set /p password=输入密码：
java -cp .\tools\xjarDemo-1.0-SNAPSHOT.jar XjarDemo %password% %1 .\encrypted.jar
pause
```

运行加密jar-加载密码配置文件

```
@echo off
java -jar %1 --xjar.keyfile=.\config\xjar.key      
::> nohup.out 2>&1 &     
pause
```

生成密码文件

```
@echo off
rem 将 tmp.txt 文件内容存入 value 变量
rem set /p value=<./password.key
rem echo %value% > xjar.key
copy /y password.key xjar.key
```