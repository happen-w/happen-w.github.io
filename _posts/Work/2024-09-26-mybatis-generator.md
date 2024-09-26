---
title: "mybatis-generator-maven-plugin代码生成问题和思考"
categories: Java
---

### mybatis代码生成
mybatis-generator-maven-plugin是一个maven插件，通过这个插件可以生成通过数据库反向生成代码,
提高开发的效率。生成的代码和自己的代码是放在一起的。如果对生成的代码进行改动，下次改动表的时候，就很麻烦。
解决方案是如果有自定义的一些sql需要写，重新写一个mapper方法。。


### 对于java代码生成的思考
java提供了一种技术, annotationProcessor,可以和java编译器进行交互，在编译期间进行交互，
典型如lombok, 在编译期间修改原来class,然后通过 IDE插件（避免报错,其实可以正确运行），实现代码的简化。
另一个典型的是QueryDsl,在编译器生成QClass文件，参与编译。
每次从新生成,避免修改生成的文件。我觉得这种是比较好的。对于生成的代码也应该是开闭原则，对修改是封闭的，对扩展是开放的。
生成的代码应该是不能修改，要修改就修改生成规则或者扩展生成的类

### 对于mybatis生成代码的优化
很简单，把生成的代码，放到编译目录，不参与源码的管理。这里就需要一些maven插件来帮助完成。当然，前提你应该明白一些
maven的生命周期, phase和goals的这2个重要概念
``` xml
<plugin>
    <artifactId>maven-antrun-plugin</artifactId>
    <version>1.8</version>
    <executions>
        <execution>
            <phase>generate-sources</phase>     // 在maven的generate-sources
            <configuration>
                <target>
                    <echo message="create dir"/>
                    <mkdir dir="${project.build.directory}/generated-sources/mybatis" /> 
                     // 生成存放生成代码目录（主要问题是因为mybatis生成指定目录如果不存在，不会自己创建才需要这个插件）
                </target>
            </configuration>
            <goals>
                <goal>run</goal>
            </goals>
        </execution>
    </executions>
</plugin>
<!-- mybatis源码生成 -->
<plugin>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-maven-plugin</artifactId>
    ......  // 这里就和常规配置一样
    <executions>
        <execution>
            <id>Generate MyBatis Artifacts</id>
            <phase>generate-sources</phase>   // 绑定到在maven的generate-sources
            <goals>
                <goal>generate</goal>       // 执行代码生成
            </goals>
        </execution>
    </executions>
</plugin>
<!-- 将生成代码加入到源码目录 -->
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>build-helper-maven-plugin</artifactId>
    <version>1.8</version>
    <executions>
        <execution>
            <id>add-source</id>
            <phase>generate-sources</phase>  // 绑定到在maven的generate-sources
            <goals>
                <goal>add-source</goal>
                <goal>add-resource</goal>
            </goals>
            <configuration>
                <sources>
                    <source>target/generated-sources/mybatis</source>          // 这里不需要directory
                </sources>
                <resources>
                    <resource>
                        <directory>target/generated-sources/mybatis</directory>  // 这里需要directory  =。= 不看文档凭直觉就被坑了
                    </resource>
                </resources>
            </configuration>
        </execution>
    </executions>
</plugin>
```
mybatis-generator.xml里面生成的位置改为target/generated-sources/mybatis
到这里，生成的代码就在target下面，同时参与编译，但源码层面不被git管理，也就不会被人修改

### 其他语言
所谓的代码生成，和编程语言上的一个概念，宏是很像的。都是通过和编译器交互，从而扩展语言的特性。
宏可以简化代码的开发，但也会加大编译调试的难度。 


### 总结
- java很垃圾也很强大
- 人是不可靠的，尽可能通过代码\工程的手段避免犯错，什么约定，风格，如果没有工具加与约束就是狗屁
 （为什么这么说，比如说mybatis生成代码的问题，我问到所如果有定制sql的时候，
 需要修改mapper的时候，然后后面又有人修改了sql,从新生成mapper不就有冲突，给我的回答是，生成的文件不要去修改，但实际情况就是你约束不了别人）
- 最后，我喜欢静态强类型的语言，checkStyle, JPA.....  =.=


