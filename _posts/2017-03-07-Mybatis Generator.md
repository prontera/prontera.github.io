---
layout:     post
title:      "MyBatis Generator速查手册"
subtitle:   "遍历MBG官网后的归纳与总结"
author:     "Chris"
header-img: "img/post-bg-7.jpg"
tags:
    - MyBatis
---

## 前言

从Eclipse到idea都一直都在用Mybatis Generator, 也完整翻阅过官方文档, 可是看完就没有那回事了. 这次决定要记录下来, 以备不时之需. 以下根据mybatis-generator-maven-plugin 1.3.5为基础而写的随笔.

## 快速指南

根据实际情况, 项目里都是用MBG的Maven插件, 这里着重以Maven的形式讲解, 并且禁用了MyBatis的Example.

### XML配置文件

以下元素就是MBG的最小配置

1. `<jdbcConnection>`元素指定如何连接数据库
2. `<javaModelGenerator>`元素指定生成Model的目标package与目标project
3. `<sqlMapGenerator>`元素指定生成Mapping XML文件的目标package与目标project
4. (Optionally)`<javaClientGenerator>`元素指定生成Mapper(即DAO)文件的目标package与目标project, 如果不指定这个元素就不会生成Mapper文件
5. 至少一个`<table>`元素

下面是一个较为完整的示例, 可以保存下来按需修改

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>

    <!--The full path name of a JAR/ZIP file to add to the classpath, or a directory to add to the classpath.-->
    <!-- 数据库的驱动, JAR/ZIP文件的全路径-->
    <classPathEntry location="/Program Files/IBM/SQLLIB/java/db2java.zip"/>

    <!--targetRuntime用MyBatis3, 也就是默认的, 其他不用管-->
    <context id="DB2Tables" targetRuntime="MyBatis3">

        <commentGenerator>
            <!-- 去除自动生成的注释 -->
            <property name="suppressAllComments" value="true"/>
        </commentGenerator>

        <!--基础的数据库连接-->
        <jdbcConnection driverClass="COM.ibm.db2.jdbc.app.DB2Driver"
                        connectionURL="jdbc:db2:TEST"
                        userId="db2admin"
                        password="db2admin">
        </jdbcConnection>

        <!--Java类型解析器, 目前也就只有forceBigDecimals可以给你玩-->
        <javaTypeResolver>
            <!--当数据类型为DECIMAL或者NUMERIC的时候, 如果是true的话则总是使用java.math.BigDecimal-->
            <!--以下是false, 即默认值的情况-->
            <!--如果有小数或者decimal长度大于18, Java类型为BigDecimal-->
            <!--如果没有小数, 以及decimal长度为10至18, Java类型为Long-->
            <!--如果没有小数, 以及decimal长度为5至9, Java类型为Integer-->
            <!--如果没有小数, 以及decimal长度少于5, Java类型为Short-->
            <property name="forceBigDecimals" value="false"/>
        </javaTypeResolver>

        <!--Domain生成器-->
        <javaModelGenerator targetPackage="test.model" targetProject="\MBGTestProject\src">
            <!--据说可以自动添加schema名, 可是我没用到过-->
            <property name="enableSubPackages" value="true"/>

            <!--生成全属性构造器, 没什么用, 如果有指定immutable元素的话这个会被忽略-->
            <property name="constructorBased" value="true"/>

            <!--生成不可变的domain, 这个我也很少用-->
            <property name="immutable" value="true"/>

            <!--每个Domain都继承这个bean-->
            <property name="rootClass" value="com.github.prontera.domain.base.BasicEntity"/>

            <!--当遇到String的时候setter是否会先trim()-->
            <property name="trimStrings" value="true"/>
        </javaModelGenerator>

        <!--Mapping生成器-->
        <sqlMapGenerator targetPackage="test.xml" targetProject="\MBGTestProject\src">
            <property name="enableSubPackages" value="true"/>
        </sqlMapGenerator>

        <!--Mapper生成器, 当type为ANNOTATEDMAPPER时是带有@annotation的Mapper, MIXEDMAPPER是XML文件-->
        <javaClientGenerator type="XMLMAPPER" targetPackage="test.dao" targetProject="\MBGTestProject\src">
            <property name="enableSubPackages" value="true"/>

            <!--每个Mapper所继承的接口-->
            <property name="rootInterface" value="com.github.prontera.Mapper"/>
        </javaClientGenerator>

        <!--字段命名策略过程: <columnRenamingRule> >> property name="useActualColumnNames"-->
        <!--alias属性是个神器, 会为所有SQL都添加, 做关联的时候就非常方便了-->
        <!--至于什么Example, 全关了就是-->
        <table alias="ha" tableName="ALLTYPES" domainObjectName="Customer"
               enableCountByExample="false" enableUpdateByExample="false"
               enableDeleteByExample="false" enableSelectByExample="false"
               selectByExampleQueryId="false">

            <!--指定是否用数据库中真实的字段名, 而不是采用MBG转换后的驼峰-->
            <property name="useActualColumnNames" value="true"/>

            <!--columnOverride是table的子元素, column必填属性为数据表的字段名-->
            <!--property属性指定生成的Java字段名, 这个设定会覆盖table中的property元素中的useActualColumnNames属性-->
            <!--typeHandler属性就是指定Mapping typeHandler的, 如果要修改Model中的请使用javaType-->
            <!--isGeneratedAlways属性为true的时候则Mapping中不会出现此字段的insert或update操作-->
            <columnOverride column="DATE_FIELD" property="startDate"/>

            <!--自动集成改类-->
            <property name="rootClass" value="com.github.prontera.domain.base.HelloBasicClass"/>

            <!--Mapper自动继承的接口-->
            <property name="rootInterface" value="com.github.prontera.Mapper"/>

            <!--当遇到String的时候setter是否会先trim()-->
            <property name="trimStrings" value="true"/>

            <!--先进行columnRenamingRule, 再进行useActualColumnNames. 如果有columnOverride则忽略该配置-->
            <!--关于columnRenamingRule的具体例子 http://www.mybatis.org/generator/configreference/columnRenamingRule.html-->
            <columnRenamingRule searchString="^CUST_" replaceString=""/>

            <!--顾名思义, 忽略某些列-->
            <ignoreColumn column="CREATE_TIME"/>

            <!--也是忽略数据列, 但是可以通过正则表达式, except子元素是可选的, 代表忽略除UPDATE_TIME外的列-->
            <ignoreColumnsByRegex pattern=".*_TIME$">
                <except column="UPDATE_TIME"/>
            </ignoreColumnsByRegex>
        </table>

    </context>
</generatorConfiguration>

```

### 命令行运行

十分简单, 从Maven仓库里下载一个MBG的jar包即可运行

```shell
java -jar mybatis-generator-core-x.x.x.jar -configfile generatorConfig.xml
```

### Maven插件运行

#### 使用Maven带来的便利

在石器时代我们都会在`generatorConfig.xml`中的targetProject属性写上一大串绝对路径, 结合Maven之后所有pom中的`<property>`元素都直接可以在`generatorConfig.xml`中引用, 请看以下例子

```xml
<project ...>
  ...
  <properties>
    <dao.target.dir>src/main/java</dao.target.dir>
  </properties>
  ...
  <build>
    ...
    <plugins>
      ...
      <plugin>
        <groupId>org.mybatis.generator</groupId>
        <artifactId>mybatis-generator-maven-plugin</artifactId>
        <version>1.3.5</version>
        <dependencies>
          <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.40</version>
          </dependency>
        </dependencies>
      </plugin>
      ...
    </plugins>
    ...
  </build>
  ...
</project>
```

`generatorConfig.xml`中就可以直接引用`${dao.target.dir}`, 让配置更加通用

```xml
<sqlMapGenerator targetPackage="com.github.prontera.domain"
                 targetProject="${dao.target.dir}">
  <property name="enableSubPackages" value="true"/>
</sqlMapGenerator>
```

> NOTE: 实战经验告诉我, 不要用MySQL 6.0以上的客户端, 因为会丢失update与delete等操作

#### 运行

在idea中的Maven面板找到`Plugins`中的`mybatis-generator`中运行即可

## 参考

[MyBatis Generator](http://www.mybatis.org/generator/index.html)