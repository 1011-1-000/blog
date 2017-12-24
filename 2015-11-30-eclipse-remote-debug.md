title: eclipse的远程调试
date: 2015-11-30 16:56:38
categories: java
tags: [eclipse,debug]
---

#### 为什么远程调试？

线上的测试环境出了问题，本想在本地测试，让同事代理到我这台机器上，但每次这样做比较麻烦，调试完成后，又要重新改回去。所以今天想着用eclipse做了一下远程debug.

<!-- more -->
#### 设置tomcat
修改tomcat的配置文件，catanlina.sh,在其中添加一行
```
    CATALINA_OPTS="-Xdebug  -Xrunjdwp:transport=dt_socket,address=6666,server=y,suspend=n"
```
中间不要换行，然后启动tomcat，发现catlina.out中会有
```
	Listening for transport dt_socket at address:1234
```
表明tomcat启动调试端口成功

#### 设置eclipse
打开eclipse的Debug configurations,选择Remote Java Application

- 填写项目名称
- 调试服务器IP
- 填写调试端口，也就是上面设置的端口号

Apply-Debug OK

#### 停止远程调试
DEBUG模式下找到Disconnect,点击即可

