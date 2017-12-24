title: JVM中的类加载原理
date: 2016-01-21 16:56:38
categories: java
tags: [jvm,java]
---
#### JVM三种预定义类型加载器

**启动（BootStrap）类加载器**：引导类载入器，是用本地代码实现的类载入器。(用C++写的二进制代码实现，是不可实例化的)，它负责装载核心类库<Java_Runtime_Home>/lib中指定的jar包加载到内存中。其实现涉及到JVM本地实现的细节，是无法直接引用的。
**扩展(Extension)类加载器**：由ExtClassLoader实现，负责加载<Java_Runtime_Home>/lib/ext或者由系统变量java.ext.dir指定位置中的类库加载到内存中。可以直接使用
**系统(System)类加载器**:由AppClassLoader实现，负责将系统类路径classpath所指的目录下的类库加载到内存中，开发者也可以直接使用。

<!--more-->

除此以外还有一类特殊的类型：线程上下文类加载器

#### 类加载双亲委派机制

双亲委派机制：就是某个特定的类加载器在接到加载类的请求时，首先将加载任务委托给上一层级（注意不是父类，只是在调用里会有System-Extension-BootStrap这样的一个加载顺序）加载器，依次递归，如果父类加载器可以完成类加载任务，就成功返回，只有父类加载器无法完成此加载任务时，才自己去加载。启动类加载器、标准扩展类加载器和系统类加载器的关系大致如图：

![类加载器](/uploads/img/javaClassloader.png)

从上图我们可以看出系统类加载器的父加载器是标准扩展类加载器，但是我们在试图获取标准扩展类加载器的父类加载器时却得到了null，就是说标准扩展类加载器本身强制设定父类加载器为null。

我们可以从下面一段程序加以验证：
```java
public class LoaderTest {  
	public static void main(String[] args) {  
	    try {  
	        System.out.println(ClassLoader.getSystemClassLoader());  
	        System.out.println(ClassLoader.getSystemClassLoader().getParent());  
	        System.out.println(ClassLoader.getSystemClassLoader().getParent().getParent());  
	    } catch (Exception e) {  
	        e.printStackTrace();  
	    }  
	}  
} 
```

#### 为什么扩展类的上一层级的加载器已经为null，仍然可以将类加载委托给启动类加载器
因为在继承ClassLoader方法时，都没有对LoadClass方法进行重写，而在LoadClass方法中，其默认的委派规则是这样的：
```java
if (parent != null) {  
    //如果存在父类加载器，就委派给父类加载器加载  
    c = parent.loadClass(name, false);  
} else {  
    //如果不存在父类加载器，就检查是否是由启动类加载器加载的类，  
    //通过调用本地方法native findBootstrapClass0(String name)  
    c = findBootstrapClass0(name);  
}  
```
也就是说如果父类加载器为null，则会调用本地方法进行启动类加载尝试。
此外，虚拟机出于安全等因素考虑，不会加载<Java_Runtime_Home>/lib存在的陌生类，开发者通过将要加载的非JDK自身的类放置到此目录下期待启动类加载器加载是不可能的。

#### 类加载器的自定义

先说一下用户自定义类加载器的工作流程:
1、首先检查请求的类型是否已经被这个类装载器装载到命名空间中了，如果已经装载，直接返回；否则转入步骤2；
2、委派类加载请求给父类加载器（更准确的说应该是双亲类加载器，真实虚拟机中各种类加载器最终会呈现树状结构），如果父类加载器能够完成，则返回父类加载器加载的Class实例；否则转入步骤3；
3、调用本类加载器的findClass（…）方法，试图获取对应的字节码，如果获取的到，则调用defineClass（…）导入类型到方法区；如果获取不到对应的字节码或者其他原因失败，返回异常给loadClass（…）， loadClass（…）转而抛异常，终止加载过程

在类的加载过程中，真正完成类的加载工作的类加载器和启动这个加载过程的类加载器，有可能不是同一个。真正完成类的加载工作是通过调用defineClass来实现的；而启动类的加载过程是通过调用loadClass来实现的。前者称为一个类的定义加载器（defining loader），后者称为初始加载器（initiating loader）。在Java虚拟机判断两个类是否相同的时候，使用的是类的定义加载器。也就是说，哪个类加载器启动类的加载过程并不重要，重要的是最终定义这个类的加载器。

方法 loadClass()抛出的是 java.lang.ClassNotFoundException异常；方法 defineClass()抛出的是 java.lang.NoClassDefFoundError异常。
类加载器在成功加载某个类之后，会把得到的 java.lang.Class类的实例缓存起来。下次再请求加载该类的时候，类加载器会直接使用缓存的类的实例，而不会尝试再次加载。也就是说，对于一个类加载器实例来说，相同全名的类只加载一次，即 loadClass方法不会被重复调用。
在绝大多数情况下，系统默认提供的类加载器实现已经可以满足需求。但是在某些情况下，您还是需要为应用开发出自己的类加载器。比如tomcat中的类载入器模块。

#### 一个加载器的实现例子：
类加载器：
```java
package classloader;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.InputStream;
public class FileSystemClassLoader extends ClassLoader{
    private String rootDir;
    
    public FileSystemClassLoader(String rootDir){
        this.rootDir = rootDir;
    }
    
    protected Class<?> findClass(String name) throws ClassNotFoundException{
        byte[] classData = getClassData(name);
        if(classData == null){
            throw new ClassNotFoundException();
        }
        else {
            return defineClass(name, classData, 0,classData.length);
        }
    }
    
    private byte[] getClassData(String className){
        String path = classNameToPath(className);
        try {
            InputStream ins = new FileInputStream(path);
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            int bufferSize = 4096;
            byte[] buffer = new byte[bufferSize];
            int bytesNumRead = 0;
            
            while ((bytesNumRead = ins.read(buffer))!=-1) {
                baos.write(buffer,0,bytesNumRead);
            }
            return baos.toByteArray();
        } catch (Exception e) {
            // TODO: handle exception
            e.printStackTrace();
        }
        return null;
    }
    
    private String classNameToPath(String className) {
        return rootDir+File.separatorChar+className.replace('.', File.separatorChar)+".class";
    }
}
```
实例类：
```java 
package com.example;
public class Sample {
    private Sample instance;
    
    public void setSample(Object instance) {
        System.out.println(instance.toString());
        this.instance = (Sample)instance;
    }
}
```
测试类:
```java
package classloader;
import java.lang.reflect.Method;
public class ClassIdentity {
    public static void main(String args[]) {
        new ClassIdentity().testClassIdentity();    
    }
    
    public void testClassIdentity() {
        String classDataRootPath = "D:/Aisainfo/JAVA/evetntT/bin";
        FileSystemClassLoader fscl1 = new FileSystemClassLoader(classDataRootPath);
        FileSystemClassLoader fscl2 = new FileSystemClassLoader(classDataRootPath);
        String className = "com.example.Sample";
        try {
            Class<?> class1 = fscl1.findClass(className);
            Object obj1 = class1.newInstance();
            Class<?> class2 = fscl2.findClass(className);
            Object obj2 = class2.newInstance();
            
            Method setSampleMethod  = class1.getMethod("setSample", java.lang.Object.class);
            setSampleMethod.invoke(obj1,obj2);
        } catch (Exception e) {
            // TODO: handle exception
            e.printStackTrace();
        }
    }
}
```
#### 关于Method
getMethod方法第一个参数指定一个需要调用的方法名称，第二个参数是需要调用方法的参数类型列表，是参数类型！如无参数可以指定null，该方法返回一个方法对象
setSampleMethod.invoke(?,?)第一个参数为已经实例化的类，第二个参数为要调用的参数


#### Java虚拟机如何判定两个类是否相同？
两个因素：1、类的命名相同 2、看加载此类的类加载器是否相同。只有两个都相同的情况下，才认为两个类是相同的。即使是字节码相同，而加载的类加载器不同也是不行的。这样一说的话对于上面的一个测试类的结果会不会报错?为什么会报错应该就会有所了解了！

#### 为什么Java虚拟机要用采用这种双亲委派的代理模式？
代理模式的设计是为了保证Java核心库的安全。所有的Java类至少需要引用java.lang.object这个类，也就是说在运行时，这个类需要被加载到虚拟机中。如果这个加载过程交给Java应用自己的类加载器去管理的话，很可能就存在多个版本的java.lang.object，并且这些类是不兼容的。同时这样做也是出于安全的考虑。



















