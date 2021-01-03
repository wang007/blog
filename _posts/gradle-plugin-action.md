---
title: gradle plugin 实战
date: 2021-01-03 15:48:50
tags:
   - java
   - gradle 
   - gradle plugin  
---

## 零
在上一篇文章中分享了 gradle 上手，这篇文章继续分享使用 gradle plugin 解决实际项目上遇到的问题。

1. 使用 gradle plugin 生成 grpc client 代码
2. 使用 gradle plugin 简化 项目中 build.gradle 的配置

<!-- more -->

## 1. 使用 gradle plugin 生成 grpc client 代码
> 首先 protoc 本身有插件机制的，无奈找了下资料全部都是 C++ 写的，为了不引入 C++ 的技术栈，所以用 gradle plugin 方式来做。

1. 需求： 公司项目 rpc 使用的是 grpc，自己封装了 grpc client 连接管理。grpc-gen 可以生成 grpc stub 作
为 client 调用，但是使用生成的 stub 调用服务有一层模板代码，希望使用代码生成的方式自己生成这个模板代码。
2. grpc-gen 插件通过 generateProto task 生成 stub 源码到 src 目录下，这个 task 最终让 compileJava task 依赖它一并参与编译。
3. grpc-client 代码生成 gradle plugin 的思路就是读取生成的 stub 代码作为 meta data 来生成 grpc client。

由于生成的 stub 是java 源代码，当时的思路是先让 stub 代码先编译完，然后去编译完的 build 目录下的 class 目录找编译后的 stub class。
通过反射 class 读取信息来生成 grpc client。  

定义 grpc-client task 为 generateEndpoint，这种方式的 task 依赖是
```
                      ── generateProto task
generateEndpoint ──> │
                      ── classes task
```
实现方式：
1. 先去 stub 源码目录找 stub 对应的类名（即 目录 + java 源文件名）
2. 根据 类名再去 classes 目录找对应的 stub class。
3. 反射读取 stub class。使用 javassist 类库生成 grpc-client class 文件 和 使用 javapoet 类库生成 grpc-client java 源文件。  

Q & A   
为什么要生成 class 文件？  
因为要读取 classes 目录的文件，此时项目已经完成了编译，所以必须手动生成 class 文件加到 classes 目录中。  

既然都有 class 文件了，为什么还需要再生成 java 源文件呢？  
为了方便使用方查询生成之后的代码是怎样的。

### 改进版本
虽然根据上述方式实现出来并在公司内部使用了一段时间，但是这种方式显然不能一种优雅的解决方案，更大的问题是升级到 gradle 高版本时报一些依赖问题。  
所以为了更优雅的方式实现，决定不使用读取 classes 目录 class文件的方式来实现，而是直接读取 stub java 源文件解析生成 grpc-client java 源文件并参与编译。  

实现方式：
1. 读取 grpc stub 所有的 java 源文件
2. 使用 javaparser 类库从 java 源文件中读取并获得 meta data。
3. 根据 meta data 生成 grpc-client java 源文件。
4. 把 grpc-client 源文件对应的目录加到 sourcesets 中，参与编译。
5. 相当于之前的版本，这种方式更简单些。例如：不需要再使用自定义 classLoader 的方式去加载 class 文件。

> 说到 classLoader，当时这里踩了个坑。 classLoader 默认的 parent 是 SystemClassLoader，而不是当前 new XxxClassLoader 所在的类的 classLoader。

这种方式的实现， task 依赖关系：
```

            compileJava                                     
                 │
            ──────────
           │          │
           ↓          ↓
generateEndpoint ──> generateProto task 
```


改进版本的伪代码：
```groovy
class EndpointGenPlugin implements Plugin<Project> {
    void apply(Project project) {
        //配置项，都有默认值，遵循约定大于配置的原则
        project.extensions.add("endpointGen", new EndpointGen())
        
        project.afterEvaluate {
            def opt = project.endpointGen //读取配置项
            if (!opt.enable) {
                //log
                return
            }
            project.task(group: "proto", "generateEndpoint") { 
                //配置阶段
                
                dependsOn  "generateProto" //依赖 generateProto task
                def compileJavaTask = project.tasks.getByName("compileJava")
                compileJavaTask.dependsOn(it) //让 compileJava 依赖当前 task
                
                def genBaseDir = project.protobuf.generatedFilesBaseDir
                // grpc service 目录
                String grpcGenDir = genBaseDir + opt.readFromGrpcSubDir
                //ep 生成目录
                String epGenDir = genBaseDir + opt.writeToEndpointSubDir
                
                //配置 task inputs，outputs
                inputs.dir(new File(grpcGenDir))
                outputs.dir(new File(epGenDir))
                
                //添加到编译目录
                compileJavaTask.source new File(epGenDir)
                
                //task 执行阶段的代码
                doLast {
                    //获取 stub java 源文件
                    def javaSrcFiles = project.fileTree(grpcGenDir).getFiles().toList()
                    //生成 meta data
                    def clientModels = EndpointClientModel.parseSrcToModel(javaSrcFiles, projectPkgName)
                    //根据 meta data 生成 grpc-client java 源文件
                    clientModels.forEach {
                        if (GenEndpointClient.gen(epGenDir, it, project) == null) {
                            throw new GradleException("generate ${it.pkg}.${it.className} java source file failed")
                        }
                    }
                } //doLast
            } //task
        }
    }
}
```


## 2. 使用 gradle plugin 简化 项目中 build.gradle 的配置
需求：随着微服务的拆分，项目会越来越多。而且项目 build.gradle 配置也比较多，对于 build.gradle 本身来说
就是重复代码，而且散落在各种项目上。所以为了解决这个问题，开发一个 gradle 插件来简化各个项目的 build.gradle 
并提供各种约定俗成的配置和简化过后的配置。  

最终效果：项目只需要 apply 这个 plugin，然后管理依赖就行了。其他的配置，包括常用插件，常用 task，插件的参数配置等全部自动化了，build.gradle 也清晰很多。

1. grpc 配置  
正常情况下，grpc gradle 配置需要写一堆的配置。通过插件集成之后，
只需要调用 applyProtobuf() 方法即可，甚至可以根据项目名来判断自动调用 applyProtobuf()。例如项目名为 proto

2. docker 打镜像配置
docker 打镜像也是集成到 gradle 中，使用的是 gradle-docker-plugin。也是需要大段的配置，同时 还有 build 脚本也写了一大堆打镜像的逻辑。
通过插件集成之后，只需要调用 applyDocker() 方法即可，甚至可以根据项目名来判断自动调用 applyDocker()。例如项目名为 server。

3. jar 打包上传
通常打 jar 和上传也是需要大段配置代码，使用的是 maven-publish 插件。通过插件集成之后，只需要调用 applyPublish() 方法即可

4. 还有很多细节的处理，这里就不一一举例了。

最终可以把这些操作统一成 task，作为项目打包的统一入口。 那么 CI 的配置就会非常简单，只需要执行：
```
./gradlew packet
``` 
就可以完成统一的打包，非常简单，也解决 java 开发者上手配置这些 task 的问题。

其实要完成这个统一打包的功能很简单，就是让 packet task 依赖 打包上传 jar 的 task，打 docker 镜像和上传镜像的 task。
简单而强大。

---
参考文档：  
[javassist](https://github.com/jboss-javassist/javassist)  
[javapoet](https://github.com/square/javapoet)  
[javaparser](https://github.com/javaparser/javaparser)  
[gradle-docker-plugin](https://github.com/bmuschko/gradle-docker-plugin)
