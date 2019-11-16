[TOC]

### 1 报错发生环境

- Intellij 2017.1.3（2017.3 之后可以正常运行）
- Springboot 2.2.x

### 2 报错场景

- 通过 intellij 新建一个最简单项目
- 创建一个类写一个测试方法
- 运行测试方法即报错

### 3 报错原因分析

#### 3.1 报错日志

- [pom 配置](#4.1 最初阶段)

- 报的是 NoSuchMethodError 错误，可知类是找到了，但是没有方法，所以肯定是 Jar 版本不对
- 知道版本不对但是没什么作用，这里用的是 intellij 相关的

```java
Exception in thread "main" java.lang.NoSuchMethodError: org.junit.platform.commons.util.ReflectionUtils.getDefaultClassLoader()Ljava/lang/ClassLoader;
	at org.junit.platform.launcher.core.ServiceLoaderTestEngineRegistry.loadTestEngines(ServiceLoaderTestEngineRegistry.java:30)
	at org.junit.platform.launcher.core.LauncherFactory.create(LauncherFactory.java:53)
	at com.intellij.junit5.JUnit5IdeaTestRunner.createListeners(JUnit5IdeaTestRunner.java:39)
	at com.intellij.rt.execution.junit.IdeaTestRunner$Repeater.startRunnerWithArgs(IdeaTestRunner.java:49)
	at com.intellij.rt.execution.junit.JUnitStarter.prepareStreamsAndStart(JUnitStarter.java:242)
	at com.intellij.rt.execution.junit.JUnitStarter.main(JUnitStarter.java:70)
```

#### 3.2 报错问题追踪(3.1)  —— 第1次伪解决

- [问题地址](<https://stackoverflow.com/questions/45004453/cannot-run-tests-intellij-spring-project-error-java-lang-nosuchmethoderror>)

- 问题总结
  - junit-jupiter-api 是为 JUnit 5 设计的
  - 而项目又依赖的 JUnit 4
  - 且，测试时使用的 @Test 是 JUnit 5 的注解
- 查看 pom 配置，发现 test 依赖移除了 junit-vintage-engine，[Springboot2.2.x 官方文档](<https://docs.spring.io/spring-boot/docs/2.2.1.RELEASE/reference/html/spring-boot-features.html#boot-features-testing>)中说明，此依赖是适配 junit 4的，如果已经迁移到 junit 5，可移除此依赖（默认是移除的）
- **修改：**将移除的依赖由 junit-vintage-engine 改为 junit-jupiter-api。并重新注入 Junit 4 的 @Test，即可成功
- 虽然可以成功运行测试，但是这**并不是我们正真的目的**，继续往下走！
- [pom 配置](#4.2 使用 Junit 4 成功用例)

#### 3.3 JUnit 5 官方文档

- [文档地址](<https://junit.org/junit5/docs/current/user-guide/#running-tests-ide-intellij-idea>)

- 官方文档(4.1.1章)指出：2017.3 之前的 Intellij 绑定了特定的 JUnit 版本
- 将官方文档说明的依赖添加到项目中，发现还是报错，报错不一样了
- [pom 配置](#4.3 参照 JUnit 5 官方文档)

```java
java.lang.NoSuchMethodError: org.junit.platform.launcher.Launcher.execute
```

#### 3.4 报错问题追踪(3.3) —— 第2次伪解决

- [问题地址]([java.lang.NoSuchMethodError: org.junit.platform.launcher.Launcher.execute](https://stackoverflow.com/questions/50323335/java-lang-nosuchmethoderror-org-junit-platform-launcher-launcher-execute))
- **关键：** pom.xml 中依赖的 Junit 相关的版本，需要和 IDEA_INSTALLATION_HOME/plugins/junit/lib 中对应

- 替换 pom.xml 文件后，发现还是出错了，现在的错和 3.1 一样
- [pom 配置](#4.4 使用和 Intellij 相同的 JUnit 版本)
- 删除所有的关于 Springboot 的配置，成功运行，但是这**并不是我们正真的目的**，继续往下走！
- [第2次伪解决pom配置](4.5 第2次伪解决 pom)

#### 3.5 最终解决

- 通过 Maven Helper 查看了依赖，发现了正真测试的时候使用的是 springboot 中的 jupiter

- 查看了 springboot parent 中的依赖，发现以下配置

```xml
<!--parent 中的配置-->
<junit-jupiter.version>5.5.2</junit-jupiter.version>
<!--测试项目 中的配置-->
<junit.jupiter.version>5.0.0-M4</junit.jupiter.version>
<!--将测试项目中的配置修改为，即可成功运行-->
<junit-jupiter.version>5.0.0-M4</junit.jupiter.version>
```

- [pom 配置](#4.6 最终成功 pom)

### 4 各阶段 pom.xml 配置

#### 4.1 最初阶段

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.2.1.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.example</groupId>
	<artifactId>demoda</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>demoda</name>
	<description>Demo project for Spring Boot</description>
	<properties>
		<java.version>1.8</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
			<exclusions>
				<exclusion>
					<groupId>org.junit.vintage</groupId>
					<artifactId>junit-vintage-engine</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
	</dependencies>
</project>
```

#### 4.2 使用 Junit 4 成功用例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
   <modelVersion>4.0.0</modelVersion>
   <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-parent</artifactId>
      <version>2.2.1.RELEASE</version>
      <relativePath/> <!-- lookup parent from repository -->
   </parent>
   <groupId>com.example</groupId>
   <artifactId>demoda</artifactId>
   <version>0.0.1-SNAPSHOT</version>
   <name>demoda</name>
   <description>Demo project for Spring Boot</description>
   <properties>
      <java.version>1.8</java.version>
   </properties>
   <dependencies>
      <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter</artifactId>
      </dependency>
      <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter-test</artifactId>
         <scope>test</scope>
         <exclusions>
            <exclusion>
               <groupId>org.junit.jupiter</groupId>
               <artifactId>junit-jupiter-api</artifactId>
            </exclusion>
         </exclusions>
      </dependency>
   </dependencies>
</project>
```

#### 4.3 参照 JUnit 5 官方文档

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
   <modelVersion>4.0.0</modelVersion>
   <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-parent</artifactId>
      <version>2.2.1.RELEASE</version>
      <relativePath/> <!-- lookup parent from repository -->
   </parent>
   <groupId>com.example</groupId>
   <artifactId>demoda</artifactId>
   <version>0.0.1-SNAPSHOT</version>
   <name>demoda</name>
   <description>Demo project for Spring Boot</description>
   <properties>
      <java.version>1.8</java.version>
   </properties>
   <dependencies>
      <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter</artifactId>
      </dependency>
      <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter-test</artifactId>
         <scope>test</scope>
         <exclusions>
            <exclusion>
               <groupId>org.junit.vintage</groupId>
               <artifactId>junit-vintage-engine</artifactId>
            </exclusion>
         </exclusions>
      </dependency>
      <!-- Only needed to run tests in a version of IntelliJ IDEA that bundles older versions -->
      <dependency>
         <groupId>org.junit.platform</groupId>
         <artifactId>junit-platform-launcher</artifactId>
         <version>1.5.2</version>
         <scope>test</scope>
      </dependency>
      <dependency>
         <groupId>org.junit.jupiter</groupId>
         <artifactId>junit-jupiter-engine</artifactId>
         <version>5.5.2</version>
         <scope>test</scope>
      </dependency>
      <dependency>
         <groupId>org.junit.vintage</groupId>
         <artifactId>junit-vintage-engine</artifactId>
         <version>5.5.2</version>
         <scope>test</scope>
      </dependency>
   </dependencies>
</project>
```

#### 4.4 使用和 Intellij 相同的 JUnit 版本

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
   <modelVersion>4.0.0</modelVersion>
   <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-parent</artifactId>
      <version>2.2.1.RELEASE</version>
      <relativePath/> <!-- lookup parent from repository -->
   </parent>
   <groupId>com.example</groupId>
   <artifactId>demoda</artifactId>
   <version>0.0.1-SNAPSHOT</version>
   <name>demoda</name>
   <description>Demo project for Spring Boot</description>
   <properties>
      <java.version>1.8</java.version>
      <junit.jupiter.version>5.0.0-M4</junit.jupiter.version>
      <junit.platform.version>1.0.0-M4</junit.platform.version>
      <junit.vintage.version>4.12.0-M4</junit.vintage.version>
   </properties>
   <dependencies>
      <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter</artifactId>
      </dependency>
      <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter-test</artifactId>
         <scope>test</scope>
         <exclusions>
            <exclusion>
               <groupId>org.junit.vintage</groupId>
               <artifactId>junit-vintage-engine</artifactId>
            </exclusion>
         </exclusions>
      </dependency>
      <!-- Only needed to run tests in a version of IntelliJ IDEA that bundles older versions -->
      <dependency>
         <groupId>org.junit.platform</groupId>
         <artifactId>junit-platform-launcher</artifactId>
         <version>${junit.platform.version}</version>
         <scope>test</scope>
      </dependency>
      <dependency>
         <groupId>org.junit.jupiter</groupId>
         <artifactId>junit-jupiter-engine</artifactId>
         <version>${junit.jupiter.version}</version>
         <scope>test</scope>
      </dependency>
      <dependency>
         <groupId>org.junit.vintage</groupId>
         <artifactId>junit-vintage-engine</artifactId>
         <version>${junit.vintage.version}</version>
         <scope>test</scope>
      </dependency>
   </dependencies>
</project>
```

#### 4.5 第2次伪解决 pom

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
   <modelVersion>4.0.0</modelVersion>
   <groupId>com.example</groupId>
   <artifactId>demoda</artifactId>
   <version>0.0.1-SNAPSHOT</version>
   <name>demoda</name>
   <description>Demo project for Spring Boot</description>
   <properties>
      <java.version>1.8</java.version>
      <junit.jupiter.version>5.0.0-M4</junit.jupiter.version>
      <junit.platform.version>1.0.0-M4</junit.platform.version>
      <junit.vintage.version>4.12.0-M4</junit.vintage.version>
   </properties>
   <dependencies>
      <!-- Only needed to run tests in a version of IntelliJ IDEA that bundles older versions -->
      <dependency>
         <groupId>org.junit.platform</groupId>
         <artifactId>junit-platform-launcher</artifactId>
         <version>${junit.platform.version}</version>
         <scope>test</scope>
      </dependency>
      <dependency>
         <groupId>org.junit.jupiter</groupId>
         <artifactId>junit-jupiter-engine</artifactId>
         <version>${junit.jupiter.version}</version>
         <scope>test</scope>
      </dependency>
      <dependency>
         <groupId>org.junit.vintage</groupId>
         <artifactId>junit-vintage-engine</artifactId>
         <version>${junit.vintage.version}</version>
         <scope>test</scope>
      </dependency>
   </dependencies>
</project>
```

#### 4.6 最终成功 pom

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
   <modelVersion>4.0.0</modelVersion>
   <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-parent</artifactId>
      <version>2.2.1.RELEASE</version>
      <relativePath/> <!-- lookup parent from repository -->
   </parent>
   <groupId>com.example</groupId>
   <artifactId>demoda</artifactId>
   <version>0.0.1-SNAPSHOT</version>
   <name>demoda</name>
   <description>Demo project for Spring Boot</description>
   <properties>
      <java.version>1.8</java.version>
      <junit-jupiter.version>5.0.0-M4</junit-jupiter.version>
      <junit.platform.version>1.0.0-M4</junit.platform.version>
      <junit.vintage.version>4.12.0-M4</junit.vintage.version>
   </properties>
   <dependencies>
      <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter</artifactId>
      </dependency>
      <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter-test</artifactId>
         <scope>test</scope>
         <exclusions>
            <exclusion>
               <groupId>org.junit.vintage</groupId>
               <artifactId>junit-vintage-engine</artifactId>
            </exclusion>
            <exclusion>
               <artifactId>junit-jupiter-api</artifactId>
               <groupId>org.junit.jupiter</groupId>
            </exclusion>
            <exclusion>
               <artifactId>junit-jupiter-engine</artifactId>
               <groupId>org.junit.jupiter</groupId>
            </exclusion>
         </exclusions>
      </dependency>
      <!-- Only needed to run tests in a version of IntelliJ IDEA that bundles older versions -->
      <dependency>
         <groupId>org.junit.platform</groupId>
         <artifactId>junit-platform-launcher</artifactId>
         <version>${junit.platform.version}</version>
         <scope>test</scope>
      </dependency>
      <dependency>
         <groupId>org.junit.jupiter</groupId>
         <artifactId>junit-jupiter-engine</artifactId>
         <version>${junit-jupiter.version}</version>
         <scope>test</scope>
      </dependency>
      <dependency>
         <groupId>org.junit.vintage</groupId>
         <artifactId>junit-vintage-engine</artifactId>
         <version>${junit.vintage.version}</version>
         <scope>test</scope>
      </dependency>
   </dependencies>
</project>
```