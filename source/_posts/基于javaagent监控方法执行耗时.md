---
title: 基于javaagent监控方法执行耗时
date: 2019-05-26 13:30:18
tags:
- javaagent
categories: JVM实战
---
![动态代理实现原理.png](https://upload-images.jianshu.io/upload_images/17387004-964993ae18068f70.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**背景描述**
javaagent是在JDK5之后提供的新特性，也可以叫java代理。开发者通过这种机制(Instrumentation)可以在加载class文件之前修改方法的字节码(此时字节码尚未加入JVM)，动态更改类方法实现AOP，提供监控服务如；方法调用时长、可用率、内存等。
<!-- more -->
**开发简述**
通过实现ClassFileTransformer接口方法，动态更改方法的字节码。在方法前后加上时间戳，最后执行完成输出执行时长。

**环境准备**
1、IntelliJ IDEA Community Edition 2018.3.1 x64
2、jdk1.8 64位

**配置信息**（路径相关修改为自己的）
1、java调试时配置
2.1、配置位置：Run/Debug Configurations ->VM options
2.2、配置内容(编译后的jar放到根目录下)：-javaagent:E:\itstack-demo-javaagent-1.0-SNAPSHOT.jar=agentargs  

**代码示例**
```
itstack-demo-javaagent
├── pom.xml
└── src
    └── main
        ├── java
        │   └── org.itstack.demo.agent
		│        ├── MyAgent.java
		│        └── MyTransformer.java
        ├── resources
        │    ├── META-INF
        │    │   └── MANIFEST.MF
        │    └── log4j2.xml
		└── test
		     └── java
			     └── org.itstack.demo.test
				     └── AgentTest.java
			  
```
pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.jd.jr.test</groupId>
    <artifactId>baitiao-test-javaagent</artifactId>
    <version>1.0-SNAPSHOT</version>

    <name>test-javaagent</name>
    <description>test-javaagent</description>

    <properties>
        <!-- Build args -->
        <argline>-Xms512m -Xmx512m</argline>
        <skip_maven_deploy>false</skip_maven_deploy>
        <updateReleaseInfo>true</updateReleaseInfo>
        <project.build.sourceEncoding>utf-8</project.build.sourceEncoding>
        <maven.test.skip>true</maven.test.skip>
        <maven.configuration.manifestFile>src/main/resources/META-INF/MANIFEST.MF</maven.configuration.manifestFile>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.javassist</groupId>
            <artifactId>javassist</artifactId>
            <version>3.22.0-GA</version>
        </dependency>
    </dependencies>

    <build>
        <sourceDirectory>src/main/java</sourceDirectory>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>
                <includes>
                    <include>**/**</include>
                </includes>
            </resource>
        </resources>
        <testSourceDirectory>src/test/java</testSourceDirectory>
        <testResources>
            <testResource>
                <directory>src/test/resources</directory>
                <filtering>true</filtering>
                <includes>
                    <include>**/**</include>
                </includes>
            </testResource>
        </testResources>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>2.4</version>
                <configuration>
                    <archive>
                        <manifestFile>${maven.configuration.manifestFile}</manifestFile>
                    </archive>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-resources-plugin</artifactId>
                <version>2.5</version>
                <configuration>
                    <encoding>${project.build.sourceEncoding}</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>2.3.2</version>
                <configuration>
                    <source>1.6</source>
                    <target>1.6</target>
                    <encoding>${project.build.sourceEncoding}</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-source-plugin</artifactId>
                <version>2.1.2</version>
                <executions>
                    <execution>
                        <id>attach-sources</id>
                        <goals>
                            <goal>jar</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

    <profiles>
        <profile>
            <id>production</id>
            <activation>
                <activeByDefault>true</activeByDefault>
                <jdk>1.6</jdk>
            </activation>
            <properties>
                <maven.compiler.source>1.6</maven.compiler.source>
                <maven.compiler.target>1.6</maven.compiler.target>
                <maven.compiler.compilerVersion>1.6</maven.compiler.compilerVersion>
            </properties>
        </profile>
    </profiles>
</project>
```

MyAgent.java
```java
package org.itstack.demo.agent;

import java.lang.instrument.Instrumentation;

public class MyAgent {

    //JVM 首先尝试在代理类上调用以下方法
    public static void premain(String agentArgs, Instrumentation inst) {
        inst.addTransformer(new MyTransformer());
    }

    //如果代理类没有实现上面的方法，那么 JVM 将尝试调用该方法
    public static void premain(String agentArgs) {
    }

}
```

MyTransformer.java
```java
package org.itstack.demo.agent;

import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;
import javassist.CtNewMethod;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class MyTransformer implements ClassFileTransformer {

    private final static String prefix = "\nlong startTime = System.currentTimeMillis();\n";
    private final static String postfix = "\nlong endTime = System.currentTimeMillis();\n";

    // 被处理的方法列表
    private final static Map<String, List<String>> methodMap = new HashMap<String, List<String>>();

    public MyTransformer() {
        //对指定方法监控
        add("org.itstack.demo.test.AgentTest.queryUserAge");
        add("org.itstack.demo.test.AgentTest.queryUserName");
    }

    private void add(String methodString) {
        String className = methodString.substring(0, methodString.lastIndexOf("."));
        String methodName = methodString.substring(methodString.lastIndexOf(".") + 1);
        List<String> list = methodMap.get(className);
        if (list == null) {
            list = new ArrayList<String>();
            methodMap.put(className, list);
        }
        list.add(methodName);
    }

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined,
                            ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        className = className.replace("/", ".");
        // 判断加载的class的包路径是不是需要监控的类
        if (methodMap.containsKey(className)) {
            CtClass ctclass = null;
            try {
                ctclass = ClassPool.getDefault().get(className);// 使用全称,用于取得字节码类<使用javassist>
                for (String methodName : methodMap.get(className)) {
                    String outputStr = "\nSystem.out.println(\"监控信息(执行耗时)：" + className + "." + methodName + " => \" +(endTime - startTime) +\"毫秒\");";
                    CtMethod ctmethod = ctclass.getDeclaredMethod(methodName);// 得到这方法实例
                    String newMethodName = methodName + "$new";// 新定义一个方法叫做比如queryUserAge$new
                    ctmethod.setName(newMethodName);// 将原来的方法名字修改
                    // 创建新的方法，复制原来的方法，名字为原来的名字
                    CtMethod newMethod = CtNewMethod.copy(ctmethod, methodName, ctclass, null);
                    // 构建新的方法体
                    StringBuilder methodBodyStr = new StringBuilder();
                    methodBodyStr.append("{");
                    methodBodyStr.append(prefix);
                    methodBodyStr.append(newMethodName + "($$);\n");// 调用原有代码，类似于method();($$)表示所有的参数
                    methodBodyStr.append(postfix);
                    methodBodyStr.append(outputStr);
                    methodBodyStr.append("}");
                    newMethod.setBody(methodBodyStr.toString());// 替换新方法
                    ctclass.addMethod(newMethod);         // 增加新方法
                }
                return ctclass.toBytecode();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return null;
    }
}
```

MANIFEST.MF
```
Manifest-Version: 1.0
Premain-Class: org.itstack.demo.agent.MyAgent
Can-Redefine-Classes: true

```

AgentTest.java

```java
package org.itstack.demo.test;

import java.util.logging.Logger;

/**
 * vm options = -javaagent:E:\itstack-demo-javaagent-1.0-SNAPSHOT.jar=agentargs
 */
public class AgentTest {

    private static Logger logger = Logger.getLogger("AgentTest");

    public static void main(String[] args) {
        String userId = "100001";
        queryUserAge(userId);
        queryUserName(userId);
    }

    private static void queryUserAge(String userId) {
        try {
            Thread.sleep(300);
            logger.info("hello userId：" + userId +" age 18");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    private static void queryUserName(String userId) {
        try {
            Thread.sleep(100);
            logger.info("hello userId：" + userId +" name agent");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

}
```
**测试结果**
```
2019-4-17 11:34:33 org.itstack.demo.test.AgentTest queryUserAge$new
信息: hello userId：100001 age 18
监控信息(执行耗时)：org.itstack.demo.test.AgentTest.queryUserAge => 316毫秒
2019-4-17 11:34:33 org.itstack.demo.test.AgentTest queryUserName$new
信息: hello userId：100001 name agent
监控信息(执行耗时)：org.itstack.demo.test.AgentTest.queryUserName => 100毫秒
```