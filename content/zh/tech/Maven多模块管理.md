+++
title = "Maven多模块管理"
date = "2023-10-06T14:34:43+08:00"
tags = []
slug = "Maven-multi-module-management"
+++

# 拆分模块

##  模块拆分策略：

### 按职责划分

eg:

- --order
  - --service
  - --po
  - --controller
  - --dao
  - --common
  - --util

### 按功能划分

- --service
  - --pay
  - --order
  - --manage



# 依赖冲突

## 查看依赖冲突

1. 通过`mvn -Dverbose dependency:tree`来查看。

如果有`omitted for duplicate`表示有jar包被重复依赖，最后写着`omitted for conflict with xxx`的，说明和别的jar包版本冲突了，而该行的jar包不会被引入。

2. idea可以用maven helper查看

## 解决方式

1. 第一声明优先：即在pom.xml文件自上而下，先声明的jar坐标，就先引用该jar的传递依赖。
2. 路径最短者优先：即直接依赖的级别高于依赖传递。可以在最外层来定义某个依赖的版本来统一版本。
3. 排除依赖：如果导入某个包的时候不想要其中的某个依赖，可以用`<exclusions>`标签。

例如：

```xml
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>5.2.7.RELEASE</version>
            <exclusions>
                <exclusion>
                    <artifactId>spring-core</artifactId>
                    <groupId>org.springframework</groupId>
                </exclusion>
            </exclusions>
        </dependency>
```

这段代码即可排除``org.springframewor`这个依赖。

# 搭建多模块项目

## 父模块

使用idea初始化父项目，什么依赖都不需要加，仅用于管理。父模块可以仅留下pom和.gitignore。

在父工程的pom文件中可以设置项目相关属性。

- 定义子模块
  可以在`<modules>`属性中定义子模块。`<module>`中填写子模块的名称

```xml
<!--父模块-->
<modules>  
    <module>user-service</module>  
    <module>order-service</module>  
    <module>eureka-server</module>  
</modules>
<!--子模块的parent属性-->
<parent>  
    <groupId>com.sipc</groupId>  
    <!--子模块名，和上方的module名一样-->
    <artifactId>cloud-learn</artifactId>  
    <version>0.0.1-SNAPSHOT</version>  
</parent>
```

- 更换打包方式

```xml
<packaging>pom</packaging>
```

pom 是最简单的打包类型。不像jar和war，它生成的构件只有它本身。将 **packaging** 申明为 **pom** 则意味着没有代码需要测试或者编译，也没有资源需要处理。
由于我们使用了聚合，所以打包方式必须为pom，否则无法构建。

- 自定义属性

```xml
<properties>  
    <spring-cloud.version>2022.0.4</spring-cloud.version>  
    <spring-cloud-alibaba.version>2022.0.0.0</spring-cloud-alibaba.version>  
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>  
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>  
    <java.version>17</java.version>  
</properties>
```

可以在这里定义一些变量，以便后续的依赖快速修改版本等。后面的依赖可以直接使用`<version>${spring-cloud-alibaba.version}</version>`这样的方式来定义依赖版本。

- 定义统一依赖管理dependencyManagement
  在Maven中，当父模块定义了`<dependencies>`元素时，子模块可以直接使用这些依赖项。子模块不需要再次指定版本号，而是可以直接引用父模块中定义的依赖项。当需要变更版本号的时候只需要在父类容器里更新，不需要任何一个子项目的修改；如果某个子项目需要另外一个特殊的版本号时，只需要在自己的模块dependencies中声明一个版本号即可。子类就会使用子类声明的版本号，不继承于父类版本号。

```xml
<dependencyManagement>  
    <dependencies>        
    <!--阿里巴巴下载仓库-->  
        <dependency>  
            <groupId>com.alibaba.cloud</groupId>  
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>  
            <version>${spring-cloud-alibaba.version}</version>  
            <type>pom</type>  
            <scope>import</scope>  
        </dependency>        
        <!-- springCloud -->  
        <dependency>  
            <groupId>org.springframework.cloud</groupId>  
            <!--是springcloud为了管理依赖所创造的依赖，里面有springcloud的所有依赖-->
            <artifactId>spring-cloud-dependencies</artifactId>  
            <version>${spring-cloud.version}</version>  
	        <!--pom的意思是仅作为pom引入，不导入实际jar包-->
            <type>pom</type>  
            <scope>import</scope>  
        </dependency>
        <!-- mysql-->    
	    <dependency>
	        <groupId>mysq1</groupId>
	        <artifactId>mysql-connector-java</artifactId>
	        <version>5.1.2</version>
        </dependency>
    </dependencies>
</dependencyManagement>
<!--子项目中只需要声明groupId喝artifactId-->
```

与`<dependencies>`的区别：

1. Dependencies相对于dependencyManagement，所有生命在dependencies里的依赖都会自动引入，并默认被所有的子项目继承。
2. dependencyManagement里只是声明依赖，并不自动实现引入，因此子项目需要显示的声明需要用的依赖。如果不在子项目中声明依赖，是不会从父项目中继承下来的；只有在子项目中写了该依赖项，并且没有指定具体版本，才会从父项目中继承该项，并且version和scope都读取自父pom;另外如果子项目中指定了版本号，那么会使用子项目中指定的jar版本。

## 子模块

在父模块之下建立一个子模块，需要在pom文件修改属性。

- 定义父模块

```xml
<parent>  
    <groupId>com.sipc</groupId>  
    <!--工件Id要和parent一致-->
    <artifactId>cloud-learn</artifactId>  
    <version>0.0.1-SNAPSHOT</version>  
</parent>
```

- 添加该模块需要的依赖
  如果在父模块的`<dependencyManagement>`定义过就不需要写version。

ps. 如果想导入其他子模块，需要在 `dependency` 中引入要依赖的子模块。我们依然应该使用在父模块声明`<dependencyManagement>`来集中管理版本。在其他子模块只需要导入groupId和artifactId，版本由父项目统一管理。

例如：父工程pom：

```xml
            <dependency>
                <groupId>com.sipc</groupId>
                <artifactId>user-service</artifactId>
                <version>${order-service.version}</version>
            </dependency>
```

在user-service的pom引入：

```xml
        <dependency>
            <groupId>com.sipc</groupId>
            <artifactId>order-service</artifactId>
        </dependency>
```

即可在user-service中使用order-service的暴露给外界的接口。



Referrence：

[maven依赖冲突以及解决方法](https://juejin.cn/post/6844904198220283918)

[maven多模块管理](https://juejin.cn/post/6844903970024980488)

