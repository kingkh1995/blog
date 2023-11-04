# [首页](/blog/)

> Gradle

***

1. 需要借助maven管理依赖；
2. 基于tasks构建的DAGs；
3. 生命周期分为：初始化、配置、执行；
4. 不需要安装Gradle，直接使用Gradle Wrapper构建。

### Build Cache

会缓存构建任务的输出，对任务的输入做哈希计算作为缓存键，从本地和远程缓存中查找是否存在构建缓存。

```Groovy
buildCache {
    local {
        directory = new File(rootDir, 'build-cache')
        removeUnusedEntriesAfterDays = 30
    }

    remote(HttpBuildCache) {
        url = 'https://example.com:8123/cache/'
        credentials {
            username = 'build-cache-user-name'
            password = 'build-cache-password'
        }
    }
}
```

### gradle wrapper

在gradle/wrapper下创建gradle-wrapper.properties文件，用于下载gradle wrapper，然后使用gradlew或gradlew.bat执行构建任务。

### settings.gradle (Settings.class)

项目配置信息

```Groovy
rootProject.name = 'demo' // 项目名
include('app', 'list', 'utilities') // 子模块
```

### build.gradle (Project.class)

```Groovy
plugins {
    // spring boot插件，需要java或war插件，指定打包方式
    // 添加了bootJar/bootRun任务，等同于spring-boot-maven-plugin插件
    id 'java'
    id 'org.springframework.boot' version 'xxx'
    id 'maven-publish' // maven发布插件，publishToMavenLocal任务等同install，publish任务等同deploy。
}

// 指定spring boot的主类（可选）
springBoot {
    mainClass = 'com.example.demo.Application'
}

dependencies {
    implementation 'com.google.guava:guava:xxx'
}

task hello {
    doLast {
        println 'Hello World!'
    }
}
```

#### 依赖注入

依赖注入，支持两种配置方式，String和Map。

```Groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter:2.7.2'
    implementation group:'org.springframework.boot', name: 'spring-boot-starter', version: '2.7.2'
}
```

- 配置类型：
    - api/compileOnlyApi：效果等同于compile；
    - implementation：对应compile，**依赖不会暴露给其他模块**，其他模块只能在运行期间使用这个依赖；
    - compileOnly：效果等同于provided；
    - runtimeOnly：效果等同于runtime；
    - developmentOnly：效果等同与optional=true
    - testImplementation/testCompileOnly/testRuntimeOnly：测试阶段依赖；
    - annotationProcessor：注解处理器依赖。

#### 依赖管理

```groovy
repositories { // 指定使用maven仓库
    mavenLocal() // 本地仓库，直接读取M2_HOME环境变量配置
    maven { // 指定本地文件目录作为maven仓库
        url 'file://d:\\repo'
    }
    maven {
        url 'https://maven.aliyun.com/repository/public/'
    }
    mavenCentral()
}
// 声明当前项目使用的依赖
plugins {
    // 添加spring依赖管理插件，自动从spring boot版本中导入spring-boot-dependencies bom。
    id 'io.spring.dependency-management' version 'xxx'
}

dependencyManagement {
    imports {
        mavenBom 'org.springframework.cloud:spring-cloud-dependencies:xxx'
    }
}
```

#### buildscript

用于声明当前gardle脚本自身使用的配置和依赖

```groovy
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'org.springframework.boot:spring-boot-gradle-plugin:2.3.4.RELEASE' 
    }
}
```

#### allProjects

用于声明当前项目及子项目共享的配置及依赖

```groovy
allProjects {
    repositories {
        mavenCentral()
    }
    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter:xxx'
    }
}
```

### check & test
test属于verification任务，会自动检测并执行所有单元测试，并且作为任何正式软件构建过程的一部分。
check通常视作“生命周期”任务（形同maven），所以他实际上不做任何事情，只是用于聚合verification任务，默认情况下只包含test任务。

### 跳过测试

- 命令行：
    ```
    gradle build -x test
    ```

- 脚本：
    ```Groovy
    test.onlyIf { 
        !project.hasProperty('skipTest') 
    }
    ```

### build

- buildNeeded：用于构建当前项目及其所有的直接和间接依赖项
- buildDependents：用于构建当前项目以及所有依赖于它的其他项目，包含buildNeeded
- build：用于构建整个项目，包括所有子项目