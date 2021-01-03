---
title: gradle 上手
date: 2021-01-02 15:22:10
tags:
  - java
  - gradle 
---

## 零
1. 大多数 java 开发者应该都使用过 maven，但是 gradle 就比较少接触了。
2. maven 上手是比较简单的。
3. gradle 上手成本是比较高的，但一旦熟悉 gradle 之后那么对于灵活构建项目是非常方便。对于构建项目领域，
gradle 比 maven 先进太多了。
4. 本文章假设你了解 groovy，如果不熟悉 groovy 的话可以先了解 groovy。对于有 java 基础的同学来说上手 groovy 基本是零成本的。

<!-- more -->

## maven
maven 是有固定项目构建生命周期。一般只需要配置好依赖和插件即可。想要灵活构建项目就会很麻烦。  
这里不深究 maven 相关的东西，直接来上手 gradle。

## gradle
gradle 是没有固定的项目构建生命周期，而非常巧妙地使用 task 依赖关系来模拟出 "项目构建生命周期"。  
接下来就按照以下顺序来讲解 gradle 。 
1. 示例代码
2. gradle wrapper
3. gradle 声明 jar 依赖
4. gradle task
5. 模块化
6. gradle 常用 api 
7. gradle plugin
8. gradle 生命周期

### 1. 示例代码
```groovy
buildscript {
    repositories {
        mavenLocal()
        maven {
            url "https://maven.aliyun.com/repository/public"
        }
        mavenCentral()
    }
    dependencies {
        classpath "org.springframework.boot:spring-boot-gradle-plugin:2.3.7.RELEASE"
    }
}

plugins {
    id "java"
    id "idea"
}

apply plugin: "org.springframework.boot"

repositories {
    mavenLocal()
    maven {
        url "https://maven.aliyun.com/repository/public"
    }
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-gateway:2.2.6.RELEASE'
}
```
这是 spring-cloud-gateway 网关项目跑起来的 build.gradle 。

---
### 2. gradle wrapper
gradle wrapper 为了适配不同版本的 gradle，有了 wrapper 就可以快速切换不同的版本。从而适配项目的变化。  
每个 gradle 项目都有会如下结构
```
.
│
├── ... 
│
├── grale  --> wrapper 目录
│    ├── gradle-wrapper.jar 
│    └── gradle-wrapper.properties
│
├── gradlew --> linux / mac 执行 gradle 入口
└── gradlew --> windows 执行 gradle 入口
```
1. 执行 gradle 命名： ./gradle task-name 即可。  
2. 这里最主要的是 radle-wrapper.properties， 指定 gradle 版本，或者修改 gradle daemon jvm 内存大小等。
```
# 指定 gradle 版本为 6.7 如果项目使用 5.6 的版本，只需要把 6.7 改成 5.6 即可。
# all 是包括源码和文档的 gradle。 bin 是只有二进制包。这里强烈推荐使用 all
distributionUrl=https\://services.gradle.org/distributions/gradle-6.7-all.zip 
```
---
### 3. gradle 声明 jar 依赖
如上面的例子，在 dependencies 中声明依赖。除了可以使用 implementation 声明依赖。还可以使用 compileOnly,RuntimeOnly 声明仅仅编译期依赖，仅仅运行期依赖
![gradle java plugin](https://docs.gradle.org/current/userguide/img/java-main-configurations.png)

1. 最终执行编译工作的 task compileJava 间接依赖了 compileOnly，implementation。
2. 也就是说，implementation 是编译期和运行期都依赖，相当于 maven scope = compile
3. 其中 compile 声明在高版本已经不推荐使用了。

原理：
1. 之所以能用这些声明，是因为 Java plugin 定义了这些声明依赖的方法。
2. Java plugin 通过 project#ConfigurationContainer 定义了 Configuration。例如 implementation 是一个 Configuration。
3. 在 dependencies 使用声明 implementation 依赖，最终通过 groovy methodMissing 原理传给了 ConfigurationContainer 的 implementation Configuration。
4. 最终结果相当于
```
dependencies {
    add("implementation", "org.springframework.cloud:spring-cloud-starter-gateway:2.2.6.RELEASE")
}
```
---
### 4. gradle task
task 是 gradle 最核心的组件，gradle 所有的构建工作都是指定执行某个 task。  
例如：
```
./gradlw build //执行 build 这个 task
./gradlew compileJava 执行编译 java 这个 task
```
task 通过 dependOn 方法声明 task 之间的依赖。 哪怕 task 不是由自己自定义的。  
task 通过声明依赖关系，最终 task 会组成一个 DAG（有向无环图），如果存在循环依赖 gradle 会报错。  
例如： 
1. 项目中通过 version 插件生成一个 git version 信息并打进 fat jar 中，这样就很明确知道当前 jar 是什么 git commit。
2. 那么只需要把 version 文件生成到 build/resources 目录中， 而在执行 processResources task 之前声明执行生成 version task 即可。
```
// versionFile 是在 net.nemerosa.versioning plugin 声明的 task
project.tasks.getByName("versionFile") {
   file = new File(project.buildDir, 'resources/main/version.properties')
}        
Task processResources = project.tasks.findByName("processResources")
processResources.dependsOn("versionFile")
```
这样执行 processResources task 之前，就会先执行 versionFile。

#### task 声明
```groovy
task printName {
    println(it.name)
}

tasks.create("printName") {
    println(it.name)
}

```
以上两种声明方式是一样的效果。

#### task 配置阶段和执行阶段
上例两个 task 的 println 方法是在配置阶段执行的，也就是说 {} 代码最终包装到 task#configure 方法中。 相当于
```
tasks.create("printName").configure {
    println(it.name)
}
```
而 configure 方法中做对 task 任何配置。
```
tasks.create("printName").configure {
    dependsOn("compileJava")  // 声明当前 task 依赖 compileJava
    group "print"             // 声明 task 所属的 group
    onlyIf { true }           // 声明 task 在什么条件下执行，这里直接返回 true，即任何条件下
    inputs.dir(new File(project.buildDir, "classes")) //声明 task 输入
    outputs.dir(new File(project.buildDir, "resources")) //声明 task 输出
    
    doFirst {
        println "doFirst"
    }
    doLast {
        println "doLast"
    }
}
```
doFirst, doLast 方法就是声明 task 执行期的操作。也就是说 ./gradlew printName, 会先后输出 doFirst，doLast。  
每个 task 可以通过 doFirst，doLast 方法声明多个 action。

#### UP-TO-DATE
1. task 是存在缓存的，也就是说 task 是最新的话，那么会跳过执行 task 去执行下一个 task。  
2. 而 判断 task 是否最新通过 inputs，outputs 来判断的。 gradle 会自动判断 inputs，outputs 声明的目录或文件是否有更新。
3. 如何关闭缓存。
   1. 不声明 inputs，outputs。
   2. 配置 outputs。
   3. 通过 干扰 inputs，outputs 对应的目录或文件。例如 outputs 目录生成的文件删除掉，inputs 目录中内容的更新一下。

```
# 对 outputs 配置下面任何一个即可关闭缓存。
outputs.cacheIf {
   false
}

outputs.upToDateWhen {
   false
}
```   
---
### 5. 模块化
在 项目根目录下 settings.gradle 中声明项目的模块，gradle 通过这种方式非常方便的支持模块化项目。

```groovy
// settings.gradle
rootProject.name = 'platform-starter'
include 'proto'
include 'server'
```
声明当前项目包括 proto，server 两个模块。  
在 gradle 定义里一个模块就是一个 project，project 之间有父子关系。例如 platform-starter 有两个 sub project：proto，server。在 project api 会体现。  
当然模块中还可以再继续定义模块。
```groovy
// settings.gradle
include 'proto'
include 'proto:abc'
include 'proto:xyz'
```
---
### 6. gradle 常用 api   
首先 gradle 项目，最先关注和使用的是 build.gradle，而 build.gradle 通过 groovy dsl 的方式定义一个 project 对象。
先梳理 project 常用方法
1. buildscript
2. repositories 和 dependencies
2. plugins
4. subprojects 和 allprojects
5. task
6. file 和 dir 相关 api

#### 1. buildscript
1. buildscript 主要是用来声明 gradle 本身的依赖。例如 依赖第三方库的插件，写 build.gradle 本身需要依赖第三方 jar。
```
buildscript {
    repositories {
        mavenLocal()
        maven {
            url "https://maven.aliyun.com/repository/public"
        }
        mavenCentral()
    }
    dependencies {
        classpath "org.springframework.boot:spring-boot-gradle-plugin:2.3.7.RELEASE"
        classpath "org.apache.commons:commons-lang3:3.11"
    }
}
```
repositories 声明依赖的 maven 仓库源，对于第三方库插件本身也是普通的 jar。  
dependencies 声明依赖，这里只能使用 classpath 声明。 例如依赖 spring boot 插件，依赖 commons-lang3 包在接下来 build.gradle 对字符串进行处理。


#### 2. repositories 和 dependencies
项目本身的 maven 仓库源和依赖声明。
buildscript 和 project 的 repositories，dependencies，可能是最容易让初学者困惑的。  
这里最主要的区别是 一个gradle 构建脚本自身的依赖，一个项目本身的依赖。  


#### 3. plugin
引入 plugin 通常有两种方式。
```groovy
//1. 这种方式引入的 plugin，plugin 默认会从 gradle 官方 plugin 仓库中找。而 java，idea 已经集成到代码中了。
plugins {
    id "java"
    id "idea"
    id "org.springframework.boot" version "2.3.7.RELEASE"
}

//2. 
buildscript {
    repositories {
        mavenLocal()
        maven {
            url "https://maven.aliyun.com/repository/public"
        }
        mavenCentral()
    }
    dependencies {
        classpath "org.springframework.boot:spring-boot-gradle-plugin:2.3.7.RELEASE"
    }
}

apply plugin: "java"
apply plugin: "idea"
apply plugin: "org.springframework.boot"
```
1. 第一种方式会引入插件并自动 apply plugin 了，当然可以关闭自动 apply
```
plugins {
    id "org.springframework.boot" version "2.3.7.RELEASE" apply false
}
```
2. 第二种需要在 buildscript # dependencies 声明插件的 maven 插件，然后在手动 apply 依赖，这种方式显然没有第一种方式简洁。


#### 4. subprojects 和 allprojects
前面说到，每个模块都是一个 project。这些模块很大部分配置都是一样，如果每个项目都配置一遍，就是重复配置和冗余的代码。  
而 subprojects 和 allprojects 可以解决上述的痛点。
1. subprojects: 当前项目的所有子项目。         exclude current project
2. allprojects: 当前项目和当前项目的所有子项目。 include current project
3. 例如给所有的子项目配置 lang3 的依赖：
```groovy
//在根目录下的 build.gradle 
subprojects {
    dependencies {
        implementation 'org.springframework.cloud:spring-cloud-starter-gateway:2.2.6.RELEASE'
    }
}
```
subprojects 代码块中的入餐是 project，dependencies 方法的调用会代理给这个入参 project。  
很显然这个方法的代码块会被执行多次，根据 sub project 解决执行多少次。

```groovy
rootProject.name = 'platform-starter'
include 'proto'
include 'proto:abc'
include 'proto:xyz'
include 'server'
```
以这种项目结构为例，项目的 allprojects 和 subprojects 的关系
1. platform-starter   
   allprojects: platform-starter, proto, proto:abc, proto:xyz, server  
   subprojects: proto, proto:abc, proto:xyz, server
2. proto  
   allprojects: proto, proto:abc, proto:xyz, server  
   subprojects: proto:xyz, server

project api 也可以获取 project 的关联 project
1. getSubprojects: 获取当前项目的所有子项目
2. getParent: 获取当前项目的直接父项目
3. ...: 还有其他 project 查询 project api

#### 5. task
1. getTasks(): 获取当前项目的所有 task，返回 TaskContainer。TaskContainer 作为 task 容器，有很多关于 task 增删改查的 api。
2. task 方法
```groovy
task([group: "print", type: Copy, dependsOn: "otherTask"], "printCopy") {
    println(it.name)
}

//当然 type 声明不能 configure 里
task("printCopy1") {
    group("print")
    dependsOn "otherTask"
    println(it.name)
}
```

#### 6. file 和 dir 相关 api
1. getBuildDir: 获取构建当前项目的构建目录路径
2. setBuildDir: 设置当前项目的构建目录路径
3. getRootDir: 获取根项目的目录路径
4. getProjectDir: 获取当前项目的目录路径
5. file(Object path): 获取指定路径的 File，路径可以是相对于当前项目，可以是绝对路径。
6. fileTree(Object baseDir): 指定 base 路径，返回以基路径的 fileTree，fileTree 功能很强大，提供很多文件过滤，匹配相关的方法。

---
### 7. gradle plugin
如果掌握了上述 project api，那么对于 gradle plugin 是非常简单的。  
先看 plugin api 定义： 
```groovy
public interface Plugin<T> {
    /**
     * Apply this plugin to the given target object.
     *
     * @param target The target object
     */
    void apply(T target);
}
```
范型 T，就是 Project。  
1. 所以 apply plugin: "some-plugin"，就是把当前 project 传给 plugin 实例然后执行 apply(project) 方法
2. apply 方法里，拿到 project 实例就可以做任意的事情。

#### 自定义 gradle plugin
1. 直接在 build.gradle 定义 Plugin，这种方式在正式项目基本很少用。
```groovy
class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
        project.task('hello') {
            doLast {
                println 'Hello from the GreetingPlugin'
            }
        }
    }
}

// Apply the plugin
apply plugin: GreetingPlugin
```
./gradlew hello 执行 task 即可看到 println 结果。  
就在我写这个例子的时候发现一个 bug。= =  [custom plugin failed in build.gradle ](https://github.com/gradle/gradle/issues/15673)

2. 在独立项目中开发 plugin，最终打成 jar 包给其他项目使用。这种方式最常使用的。
```
.
│
├── ... 
│
├──src
│    └── main
│         ├── groovy
│         │   └── com
│         │       └── example
│         │           └── plugin
│         │               └── GreetingPlugin.groovy
│         ├── resources
│               └── META-INF
│                     └── gradle-plugins
│                           └── gretting.properties
└── build.gradle
```
在 gretting.properties 指定 plugin 实现类
```
implementation-class=com.example.plugin.GreetingPlugin
```
最终打包上传 jar，跟 spring-boot plugin 一样，在 buildscript # dependencies，使用 classpath 引入依赖  
然后在 apply plugin
```
apply plugin: "gretting" //这里使用 properties 文件的名称，去掉 properties 后缀
```

---
### 8. gradle 生命周期
gradle 自身是有生命周期的，分为3个阶段。
1. 初始化阶段
加载 settings.gradle 文件并执行，这一阶段就清楚的知道当前项目一共有多少个模块（project）

2. 配置阶段
前面说了，每个 build.gradle 文件对应一个 project 对象，那么这一阶段就会执行 build.gradle 中的代码，包括 task#configure 方法

3. 执行阶段
在配置阶段就能确定了 task 之间的依赖关系，根据 gradle 命名行参数决定执行哪个 task

#### 知道 gradle 生命周期 有什么用呢？
在 gradle 对象（ project#getGradle() 获取）中定义了各种钩子函数，project 对象也定义两个钩子函数 beforeEvaluate，afterEvaluate。  
gradle 钩子函数
1. buildStarted 
2. beforeSettings
3. settingsEvaluated
4. projectsLoaded
5. beforeProject
6. projectsEvaluated
7. afterProject
8. buildFinished
9. 其中还包括各种 Listener

project 钩子函数  
1. beforeEvaluate: 在执行 project 配置之前执行。它必须在父级以上的 project 来配置。  
因为直接配置自己的 build.gradle，执行 build.gradle 已经是在 Evaluate ing 了，所以为了能在 Evaluate 之前，需要写到 parent project 上。
其实这个用的很少。

2. beforeEvaluate: 在执行 project 配置之后执行。这个方法对于插件开发来说就非常有用了。  


---
参考资料：  
[gradle 官方文档](https://docs.gradle.org/current/userguide/userguide.html)  
[gradle java plugin](https://docs.gradle.org/current/userguide/java_plugin.html)  
[Build Lifecycle](https://docs.gradle.org/current/userguide/build_lifecycle.html)









