[TOC]

## 1 使用 maven 创建一个 project

### 1.1 下载安装 maven

- maven 是一个 java 工具，使用它必须 [下载安装 Java 环境](<https://www.oracle.com/technetwork/java/javase/downloads/index.html>)

- 去 [maven 官网](<https://maven.apache.org/download.cgi>) 下载安装 maven，安装配置参考[官方文档](<https://maven.apache.org/users/index.html>)

- 运行以下命令查看安装的 maven 信息
```cmd
mvn -v
```
### 1.2 创建 project

- 使用以下命令创建一个 project，创建命令的目录下不要有 pom.xml 文件 

```cmd
# 注：不换行
mvn archetype:generate -DgroupId=com.mycompany.app -DartifactId=my-app -DarchetypeArtifactId=maven-archetype-quickstart -DarchetypeVersion=1.4 -DinteractiveMode=false
# 以上命令会在当前目录下创建一个项目，项目结构如下
my-app
|-- pom.xml
`-- src
    |-- main
    |   `-- java
    |       `-- com
    |           `-- mycompany
    |               `-- app
    |                   `-- App.java
    `-- test
        `-- java
            `-- com
                `-- mycompany
                    `-- app
                        `-- AppTest.java
```

### 1.3 构建 project 生成 jar 包

```cmd
#进入项目根目录(cd my-app)，使用以下命令进行构建生成 jar 包
mvn package
# jar 包在 my-app/target/ 目录下，使用以下命令运行 jar 包，会打印出 hello world
java -cp target/my-app-1.0-SNAPSHOT.jar com.mycompany.app.App
```



## 2 Maven 生命周期及常见命令

### 2.1 maven 生命周期

- **validate**: 验证项目是否正确
- **compile**: 编译项目的源代码
- **test**: 使用合适的单元测试框架测试编译的源代码。这些测试不应该要求代码被打包或部署
- **package**: 打包项目，生成 jar 包 或 war 包
- **verify**: 运行所有检查，检测 package 是否有效
- **install**: 将项目打成 maven 放入本地 maven 仓库
- **deploy**: 将项目发布到远程 maven 仓库

- **clean**: 清空 target 目录

- **site**: 生成项目文档，运行 `mvn site` 命令会在 target/site/目录下生成文件



- **注：** maven 命令可顺序执行，如下

```cmd
mvn clean dependency:copy-dependencies package
```

- **注：** test 命令默认执行以下类型类 `**/*Test.java` ` **/Test*.java` ` **/*TestCase.java`。默认不执行以下类型 `**/Abstract*Test.java` `**/Abstract*TestCase.java`

### 2.2 其它命令

- [官方文档命令总结](<https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html>)

```cmd
mvn test-compile #编译测试源码

-D 指定参数，如 -Dmaven.test.skip=true 跳过单元测试；
-P 指定 Profile 配置，用逗号隔开，可以用于区分环境；
-e 显示maven运行出错的信息；
-o 离线执行命令,即不去远程仓库更新包；
-X 显示maven允许的debug信息；
-U 强制去远程更新snapshot的插件或依赖，默认每天只更新一次；
-f 强制指定 pom 文件
-l 指定构建日志路径
-X 输出 debug 日志
-s 指定 settings 文件
-N 不去递归执行子模块
```



## 3 父子项目管理

### 3.1 Super POM

- [Super POM](<https://maven.apache.org/guides/introduction/introduction-to-the-pom.html>) 是 Maven 默认的 POM，如果项目没有显示继承某个 pom，则此项目继承 Super POM

### 3.2 Minimal POM

- 一个 pom 最小需要元素为：`project root` `modelVersion ` `groupId `  `artifactId ` `version `

### 3.3 项目继承的内容

- dependencies
- developers and contributors
- plugin lists (including reports)
- plugin executions with matching ids
- plugin configuration
- resources（资源配置属性）

### 3.4 项目继承几种示例 —— parent

- 项目继承是在子模块中指定父项目
- 当你拥有多个 maven 项目，且他们又共同的配置，就可以将公共配置重构成它们的父项目，并继承这个父项目

#### 3.4.1 示例1

```xml
.
 |-- my-module
 |   `-- pom.xml
 `-- pom.xml

<!--对应的子pom-->
<project>
  <parent>
    <groupId>com.mycompany.app</groupId>
    <artifactId>my-app</artifactId>
    <version>1</version>
  </parent>
  <!--可以继承父pom的version和groupId-->
  <modelVersion>4.0.0</modelVersion>
  <artifactId>my-module</artifactId>
</project>
```

#### 3.4.2 示例2

```xml
.
 |-- my-module
 |   `-- pom.xml
 `-- parent
     `-- pom.xml
<!--对应的子pom-->
<project>
  <parent>
    <groupId>com.mycompany.app</groupId>
    <artifactId>my-app</artifactId>
    <version>1</version>
    <!--使用此元素指定父pom的位置-->
    <relativePath>../parent/pom.xml</relativePath>
  </parent>
  <modelVersion>4.0.0</modelVersion>
  <artifactId>my-module</artifactId>
</project>
```

### 3.5 项目汇总几种示例 —— module

- 项目汇总是在父项目中指定子模块

- 父模块必须是 pom

- 调用父项目的命令时，会触发子项目的相同命令
- 当你拥有的几个项目需要同时构建运行时，你可以创建一个父项目，并在父项目中指定子项目

#### 3.5.1 示例1

```xml
.
 |-- my-module
 |   `-- pom.xml
 `-- pom.xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.mycompany.app</groupId>
  <artifactId>my-app</artifactId>
  <version>1</version>
  <packaging>pom</packaging>
  <modules>
    <module>my-module</module>
  </modules>
</project>
```

#### 3.5.2 示例2

```xml
.
 |-- my-module
 |   `-- pom.xml
 `-- parent
     `-- pom.xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.mycompany.app</groupId>
  <artifactId>my-app</artifactId>
  <version>1</version>
  <packaging>pom</packaging>
 
  <modules>
    <!--通过相对路径指定子模块-->
    <module>../my-module</module>
  </modules>
</project>
```



## 4 Maven 依赖管理

### 4.1 依赖传递机制

- 依赖多版本处理(*Dependency mediation*)：当项目遇到多个版本的依赖时，maven 会选取依赖树中最浅层次的版本。如果版本在同一层次，取先声明的
- 依赖版本管理(*Dependency management*)：通过 dependencyManagement，可同一管理依赖版本
- 依赖作用域(*Dependency scope*)：可为每个依赖设置特定的生命周期(生效时期)
- 依赖排除(*Excluded dependencies*)：通过 exclusion 元素排除依赖
- 依赖传递可选(*Optional dependencies*)：使用 optional 元素可使得 A->(optional)B C->A 的情形下，C 不会继承 B
- **注：** 虽然依赖传递可以使你从依赖的项目中得到其它依赖，但是仍然建议在本项目中显示的依赖其它依赖
- **注：** [excluded 与 optional 官方文档](<https://maven.apache.org/guides/introduction/introduction-to-optional-and-excludes-dependencies.html>)

### 4.2 *Dependency scope*

- 依赖作用域用于限制依赖传递，并且会影响项目的各个 maven 任务的执行
- 共有 6 个 scope
  - **complie**：依赖被传递，(默认)
  - **provided**：提示需要 JDK 或容器(tomcat)在运行时提供此依赖。被此 scope 修饰的依赖，只在编译和测试时有效，依赖不被传递
  - **runtime**：此依赖只在运行或测试时有效，不参与编译，依赖被传递
  - **test**：此依赖只在测试的编译和运行期时有效，依赖不被传递
  - **system**：见 5.4
  - **import**：此 scope 只用于 packaging 为 pom 的 dependencyManagement 节点下，用于伪实现 多父项目继承(maven 只支持当父项目)
- 以下表格，左边列表是当前项目的依赖，上面行是当前依赖所依赖的依赖

|          | compile  | provided | runtime  | test |
| -------- | -------- | -------- | -------- | ---- |
| compile  | compile  | -        | runtime  | -    |
| provided | provided | -        | provided | -    |
| runtime  | runtime  | -        | runtime  | -    |
| test     | test     | -        | test     | -    |

- 有上表可以得出以下结论
  - 被传递的依赖如果是 complie 类型，它会随着本项目的依赖的 scope 而相应变化（取最小作用域的那个 scope）
  - provided、test 不会被传递
  - 被传递的依赖如果是 runtime 类型，会综合本项目依赖的 scope 取最小作用域的那个 scope



### 4.3 *Dependency Management*

- 依赖项管理是一种集中依赖项信息的机制
- 如果有很多项目继承与一个共同的父项目，可以将所有依赖信息放入父 pom，并且在子 pom 中简单的引用父 pom
- dependencyManagement 中的依赖优先于 依赖就近原则

#### 4.3.1 示例1 —— 依赖版本选择

- Project A

```xml
<project>
 <modelVersion>4.0.0</modelVersion>
 <groupId>maven</groupId>
 <artifactId>A</artifactId>
 <packaging>pom</packaging>
 <name>A</name>
 <version>1.0</version>
 <dependencyManagement>
   <dependencies>
     <dependency>
       <groupId>test</groupId>
       <artifactId>a</artifactId>
       <version>1.2</version>
     </dependency>
     <dependency>
       <groupId>test</groupId>
       <artifactId>b</artifactId>
       <version>1.0</version>
       <scope>compile</scope>
     </dependency>
     <dependency>
       <groupId>test</groupId>
       <artifactId>c</artifactId>
       <version>1.0</version>
       <scope>compile</scope>
     </dependency>
     <dependency>
       <groupId>test</groupId>
       <artifactId>d</artifactId>
       <version>1.2</version>
     </dependency>
   </dependencies>
 </dependencyManagement>
</project>
```

- Project B

```xml
<project>
  <parent>
    <artifactId>A</artifactId>
    <groupId>maven</groupId>
    <version>1.0</version>
  </parent>
  <modelVersion>4.0.0</modelVersion>
  <groupId>maven</groupId>
  <artifactId>B</artifactId>
  <packaging>pom</packaging>
  <name>B</name>
  <version>1.0</version>
 
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>test</groupId>
        <artifactId>d</artifactId>
        <version>1.0</version>
      </dependency>
    </dependencies>
  </dependencyManagement>
 
  <dependencies>
    <dependency>
      <groupId>test</groupId>
      <artifactId>a</artifactId>
      <version>1.0</version>
      <scope>runtime</scope>
    </dependency>
    <dependency>
      <groupId>test</groupId>
      <artifactId>c</artifactId>
      <scope>runtime</scope>
    </dependency>
  </dependencies>
</project>
```

- b 为 1.0：b 在其父模块中的 management 中定义了，并且 **management 优先于就近原则**（即使 a、c 中依赖了 b，也不会被使用）
- d 为 1.0：d 在本项目中定义了，并且**本项目定义优先于父项目定义**

- [dependencyManagement 中可用的 element、tag](<https://maven.apache.org/ref/3.6.2/maven-model/maven.html#class_DependencyManagement>)

#### 4.3.2 示例 2 —— BOM

- 根项目为 BOM pom 项目，通过它统一指定需要被创建的子项目的版本，构成了一个统一版本库
- 其它项目使用这个统一版本库时，通过在 dependencyManagement 中 import 此 BOM

##### 4.3.2.1 BOM 项目

- 它定义了一个项目库

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.test</groupId>
  <artifactId>bom</artifactId>
  <version>1.0.0</version>
  <packaging>pom</packaging>
  <properties>
    <project1Version>1.0.0</project1Version>
    <project2Version>1.0.0</project2Version>
  </properties>
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>com.test</groupId>
        <artifactId>project1</artifactId>
        <version>${project1Version}</version>
      </dependency>
      <dependency>
        <groupId>com.test</groupId>
        <artifactId>project2</artifactId>
        <version>${project1Version}</version>
      </dependency>
    </dependencies>
  </dependencyManagement>
  <modules>
    <module>parent</module>
  </modules>
</project>
```

##### 4.3.2.2 正常的父项目

- 此项目为其他项目的父项目
- 此项目继承 BOM 项目

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
     xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>com.test</groupId>
    <version>1.0.0</version>
    <artifactId>bom</artifactId>
  </parent>
  <groupId>com.test</groupId>
  <artifactId>parent</artifactId>
  <version>1.0.0</version>
  <packaging>pom</packaging>
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>log4j</groupId>
        <artifactId>log4j</artifactId>
        <version>1.2.12</version>
      </dependency>
      <dependency>
        <groupId>commons-logging</groupId>
        <artifactId>commons-logging</artifactId>
        <version>1.1.1</version>
      </dependency>
    </dependencies>
  </dependencyManagement>
  <modules>
    <module>project1</module>
    <module>project2</module>
  </modules>
</project>
```

##### 4.3.2.3 其它子项目引入 bom

- 依赖了 bom 中定义的版本号 

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
     xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.test</groupId>
  <artifactId>use</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>com.test</groupId>
        <artifactId>bom</artifactId>
        <version>1.0.0</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.test</groupId>
      <artifactId>project1</artifactId>
    </dependency>
    <dependency>
      <groupId>com.test</groupId>
      <artifactId>project2</artifactId>
    </dependency>
  </dependencies>
</project>
```

### 4.4 system scope

- 在当前系统路径中找依赖(而不是在仓库中)。一般为 JDK 或 VM 提供的依赖

```xml
<project>
  ...
  <dependencies>
    <dependency>
      <groupId>javax.sql</groupId>
      <artifactId>jdbc-stdext</artifactId>
      <version>2.0</version>
      <scope>system</scope>
      <systemPath>${java.home}/lib/rt.jar</systemPath>
    </dependency>
    <dependency>
      <groupId>sun.jdk</groupId>
      <artifactId>tools</artifactId>
      <version>1.5.0</version>
      <scope>system</scope>
      <systemPath>${java.home}/../lib/tools.jar</systemPath>
    </dependency>
  </dependencies>
  ...
</project>
```



## 5 profiles 解析

- Maven 2.0 开始支持 profile 功能
- 通过指定有效的 profile 可以很方便的更换运行配置，而不需要使用 -f 来指定不同的 pom
- [官网地址](<https://maven.apache.org/guides/introduction/introduction-to-profiles.html>)

### 5.1 profile 级别

- 项目级别：定义在本项目中的 `pom.xml` 中
- 用户级别：定义在 `%USER_HOME/.m2/settings.xml`
- 全局级别：定义在 `${maven.home}/conf/settings.xml`

- Profile descriptor：定义在项目根路径下的 `profiles.xml` ，Maven 3.0 不支持

### 5.2 profile 激活方式

- 通过 -P 指定

```cmd
mvn groupId:artifactId:goal -P profile-1,profile-2
```

- 通过 Maven settings 中的 activeProfiles 节点指定

```xml
<settings>
  ...
  <activeProfiles>
    <activeProfile>profile-1</activeProfile>
  </activeProfiles>
  ...
</settings>
```

- 通过 activation 节点 的 jdk

```xml
<profiles>
  <profile>
    <activation>
      <jdk>1.4</jdk><!--jdk 为 1.4 开头-->
      <jdk>[1.3,1.6)</jdk><!--jdk 1.3 到 1.6-->
      <jdk>[1.3,1.6]</jdk><!--其实基本上等于上面，因为 1.6_x 就不在范围了-->
    </activation>
    ...
  </profile>
</profiles>
```

- 通过 activation 节点 的 os

```xml
<profiles>
    <profile>
        <activation>
            <os>
                <name>Windows XP</name>
                <family>Windows</family>
                <arch>x86</arch>
                <version>5.1.2600</version>
            </os>
        </activation>
        ...
    </profile>
</profiles>
```

- 通过 activation 节点 的 property
  - **Note**: Environment variables like `FOO` are available as properties of the form `env.FOO`. Further note that environment variable names are normalized to all upper-case on Windows.

```xml
<profiles>
    <profile>
        mvn groupId:artifactId:goal -Ddebug=false  
        <activation>
            <property>
                <!--如果系统配置中有 debug 对应的的值则激活-->  
                <name>debug</name>
                <!--如果系统配置中没有debug 对应的的值则激活-->  
                <!--<name>debug</name>-->
                <!--<value>!true</value>--><!--同时可以指定value值-->
            </property>
        </activation>
        ...
    </profile>
    <profile>
        mvn groupId:artifactId:goal -Denvironment=test  
        As of Maven 3.0, profiles in the POM can also be activated based on properties from active profiles from the settings.xml.
        <activation>
            <property>
                <name>environment</name>
                <value>test</value>
            </property>
        </activation>
        ...
    </profile>
</profiles>
```

- 通过 file 节点

```xml
<profiles>
  <profile>
    <activation>
      <file>
        <missing>target/generated-sources/axistools/wsdl2java/org/apache/maven</missing>
      </file>
    </activation>
    ...
  </profile>
</profiles>
```

- 定义默认激活 profile
  - 当没有其他 profile 被激活时，才激活此 profile

```xml
<profiles>
  <profile>
    <id>profile-1</id>
    <activation>
      <activeByDefault>true</activeByDefault>
    </activation>
    ...
  </profile>
</profiles>
```

### 5.3 profile 中可以定义的内容

#### 5.3.1 在外部文件中

- 在 settings.xml 或 profiles.xml 中只能定义 `<repositories>` `<pluginRepositories>` `<properties>`

#### 5.3.2 在本项目的 pom 中

- `<repositories>`
- `<pluginRepositories>`
- `<dependencies>`
- `<plugins>`
- `<properties>` (not actually available in the main POM, but used behind the scenes)
- `<modules>`
- `<reporting>`
- `<dependencyManagement>`
- `<distributionManagement>`
- a subset of the `<build>` element, which consists of
  - `<defaultGoal>`
  - `<resources>`
  - `<testResources>`
  - `<finalName>`

### 5.4 显示激活的 profile

```cmd
mvn help:active-profiles
```



## 6 Settings 节点解析

- [官网地址](<https://maven.apache.org/settings.html>)

## 7 pom 节点解析

- [官网地址](<https://maven.apache.org/pom.html>)





