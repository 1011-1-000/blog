title: linux下自动化接口测试
date: 2015-11-26 16:56:38
categories: java
tags: [ant,linux]
---
#### 先开个头

整套系统，也不是个系统，只是对于公司来说目前够用而已。目前实现git上传测试接口，服务器读取需要测试的接口文件，分十个线程并发执行测试任务，将测试日志打包，邮件到我的个人邮箱。目前只支持返回格式为json.主要是用ant来实现这一整套流程。虽然很简单，但是这个过程学到了很多。总结一下。

#### ant环境配置
没什么好说的
修改/etc/profile添加：
```
export ANT_HOME=/application/apache-ant-1.9.6
export PATH=$PATH:$ANT_HOME/bin

source /etc/profile立即生效
```
<!--more-->

#### ant配置文件

##### 下载文件
```
<target name="wget" depends="init">
		<echo>下载最新的接口测试文件</echo>
		<get dest="${junit.testfile.dir}" verbose="true">
		  <url url="${junit.testfile.url}"/> 
		</get>
</target>
```
##### 解压文件
```
	<target name="unzip" depends="wget">
		<echo>解压测试文件</echo>
		<unzip src="${junit.testfile.dir}/master.zip" dest="${junit.testfile.dir}"/>
    </target>
```
##### 编译文件（本来是想每个接口写junit的，发现这样比较麻烦，但程序还是跑在junit里了）
```
	<target name="junit-javac" depends="unzip" description="junit test case compile">
		<echo>编译junit</echo>
		<javac destdir="${junit.build.dir}" srcdir="${junit.src.dir}" fork="false" includeantruntime="false">
			<classpath>
				<fileset dir="${junit.lib.dir}">
					<include name="*.jar"/>
				</fileset>
			</classpath>
		</javac>
	
		<echo>复制配置文件到编译目录下</echo>
		<copy todir="${junit.build.dir}">
			<fileset dir="${junit.src.dir}">
				<include name="**/*.properties"/>    
				<include name="**/*.xml"/>
			</fileset>
		</copy>
    </target>
```
##### 运行编译后的代码
```
	<target name="junit-run" depends="junit-javac" description="junit test case execute">
		<echo>运行单元测试</echo>
		<junit printsummary="yes" haltonfailure="false">
			<classpath>
				<pathelement location="${junit.build.dir}"/>
				<fileset dir="${junit.lib.dir}">
					<include name="*.jar"/>
				</fileset>
			</classpath>
			<formatter type="xml"/>
			<batchtest todir="${junit.report.dir}">
				<fileset dir="${junit.src.dir}">
					<include name="**/*Test.java"/>
				</fileset>
			</batchtest>
		</junit>
    </target>
```
##### junit报告生成
```
	<target name="junit-report" depends="junit-run" description="junit test case report">
		<echo>生成单元测试报告</echo>
	
		<junitreport todir="${junit.report.dir}">
			<fileset dir="${junit.report.dir}">
				<include name="**/*.xml"/>
			</fileset>
			<report format="frames" todir="${junit.report.dir}/html"/>
		</junitreport>        
  	</target>
```
##### 打包文件日志
```
  	<target name="junit-package" depends="junit-report" description="package">
		<echo>使用zip打包日志</echo>
		<tstamp>
			<format property="DSTAMP" pattern="yyyyMMdd"/>
		</tstamp>
		<zip destfile="${junit.logs.dir}/logs-${DSTAMP}.zip" basedir="${junit.logs.dir}"></zip>
	
		<echo>保存日志文件</echo>
		<copy todir="${junit.logs.zip.dir}">
			<fileset dir="${junit.logs.dir}">
				<include name="*.zip"/>    
			</fileset>
		</copy>
  	</target>
```
##### 发送到我的邮箱
```
  	<target name="junit-mail" depends="junit-package" description="mail to me">
		<echo>发送邮件到我的邮箱</echo>
	
		<mail mailhost="${mailhost}" subject="${mailsubject}" user="${username}" password="${password}"> 
			<from address="${mailfrom}" /> 
			<to address="${mailto}" /> 
			<message mimetype="text/html" src="${junit.report.dir}\html\all-tests.html"/>
			<fileset dir="${junit.report.dir}\html"> 
				<include name="Testsuite-report-${var}.html" /> 
				<include name="Testcase-reports-${var}.zip" /> 
			</fileset>
			<attachments>
				<fileset dir="${junit.logs.dir}">
				  <include name="logs-*.zip"/>
				</fileset>
		    </attachments>
		</mail>        
  	</target>
```
#### Java部分

接下来就是java代码，利用httpclient发送和接收请求，利用fastjson解析json对象。得到返回结果，将返回结果与期望值做对比。
这里主要用到的是

- executor线程池，用于多个线程并发执行接口的测试

- antomicInteger实现原子性的自增和自减，用于计数，在所有线程执行结束后，获取线程执行成功及失败的数量。

#### 碰到的问题
1、读取properties配置文件时，总是找不到路径。将

	Object.class.getResourceAsStream(name）
改为

	ResourceBundle bundle = ResourceBundle.getBundle("config");
可以避免路径问题

2、用阿里云的服务器时,直接执行ant junit-mail里是可以发送邮件的，但是放到脚本里用定时任务时，却总发不出邮件

	sendmail: fatal: parameter inet_interfaces: no local interface found for ::1

解决办法
```
	vim /etc/postfix/main.cf
    修改inet_protocols = all 为inet_protocols = ipv4
	然后执行service postfix restart
```
3、定时任务的问题

shell脚本可以正常执行，定时任务时却不可以，需要在脚本中导入环境变量
```
	#!/bin/bash
	cd /application/junitTest

	export ANT_HOME=/application/apache-ant-1.9.6
	export PATH=$PATH:$ANT_HOME/bin

	JAVA_HOME=/usr/java/jdk1.7.0_79
	PATH=$JAVA_HOME/bin:$PATH
	CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
	export JAVA_HOME
	export PATH
	export CLASSPATH
	
	ant junit-mail
```
shell脚本是我的弱项。需要加强！！！




