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
    // spring boot插件，需要java或war插件，表示打包方式。
    id 'java'
    id 'org.springframework.boot' version '2.7.2'
}

// 指定spring boot的主类
springBoot {
    mainClass = 'com.example.demo.Application'
}

dependencies {
    compile 'com.google.guava:guava:23.0'
    testCompile 'junit:junit:4.12'
}

repositories {
    jcenter()
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
    - api：等同于compile；
    - implementation：与compile对应，但是不会暴露给其他模块，其他模块只能在运行期间使用这个依赖，能做到编译隔离；
    - compileOnly：等同于provided；
    - compileOnlyApi：仅作用于编译期间的api类型；
    - runtimeOnly：等同于runtime；
    - testImplementation、testCompileOnly、testRuntimeOnly：测试阶段依赖。
    - annotationProcessor：注解处理器配置；

#### 依赖管理

```groovy
plugins {
    // 添加spring依赖管理插件
    id 'io.spring.dependency-management' version '1.0.12.RELEASE'
}

dependencyManagement {
    imports {
        mavenBom 'org.springframework.cloud:spring-cloud-dependencies:2021.0.3'
    }
}
```

#### buildscript

用于声明gardle脚本自身所需要使用的资源

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

#### 插件

两种方式

```groovy
apply plugin: 'org.springframework.boot'

plugins {
    id 'org.springframework.boot' version '2.3.4.RELEASE'
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