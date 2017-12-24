title: 脚本发布测试环境到正式环境
date: 2015-12-01 16:56:38
categories: linux
tags: [shell,linux]
---

#### 场景
昨天把我们的一个新项目布了一个测试环境，但在测试环境测试通过后，想把测试环境发布到正式环境，就需要额外的一些操作，所以想着用脚本一键发布到正式环境，但这里有个问题是之前spring配置文件是被打包在webapp-service.jar中。所以需要一个替换命令。

<!-- more -->

#### 替换jar包里的文件用的命令：
```
	jar uvf ./webapp-service-1.1.1.jar ./spring-common.xml
```

#### 脚本
```
	#!/bin/bash
	export JAVA_HOME=/nginx/jdk1.7.0_79
	export JRE_HOME=/nginx/jdk1.7.0_79/jre
	export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib:$CLASSPATH
	export PATH=$JAVA_HOME/bin:$PATH

	#stop tomcat
	echo "stopping tomcat"
	cd /baai/webapp-apache-tomcat-7.0.61/bin
	./shutdown.sh

	#delete webapp
	echo "delete webapp"
	cd /baai/webapp_api
	rm -rf webapp


	#copy files
	echo "copy files"
	cp -r /baai/test_webapp_api/webapp /baai/webapp_api

	#replace spring-common.xml
	echo "replace spring-common.xml"
	cp /baai/webapp_api/spring-common.xml /baai/webapp_api/webapp/WEB-INF/lib
	cd /baai/webapp_api/webapp/WEB-INF/lib
	jar uvf ./webapp-service-1.1.1.jar ./spring-common.xml
	rm -f spring-common.xml

	#start tomcat
	echo "starting tomcat"
	cd /baai/webapp-apache-tomcat-7.0.61/bin
	./startup.sh
```
脚本写的不好，正在学习。






















































