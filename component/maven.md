## [首页](/blog/)

### Maven生命周期

> 是抽象的，只声明了生命周期和阶段，但本身不做任何实际的工作，实际工作都交由插件来完成。执行生命周期中一个步骤时，会执行顺序执行该生命周期之前所有的步骤，每个Maven构建步骤都可以绑定一个或多个插件行为。

#### 三种生命周期

- default：validate -> deploy（不包含site）
- clean
- site

#### 主要的阶段

- mvn compile：编译类文件
- mvn pacakge：包含compile和test, 生成target目录，编译、测试代码，生成测试报告，最终生成jar/war文件
- mvn install：包含compile至package，打包完成后，验证测试结果，然后上传本地仓库，供本地项目使用
- mvn deploy：包含install，安装完成后，上传文件到私服仓库

### 依赖作用域

- compile：默认，作用域为all，依赖会进行传递；
- provided：在编译、测试时有效，在运行时无效，意味着该依赖不会被打包，不会进行依赖传递；
- runtime：在运行时和测试时有效，编译时无效，一般作为具体实现，在编译期间只需要知道其接口即可，会进行依赖传递；
- test：只在测试期间有效，不会进行依赖传递。

### spring boot maven plugin

> repackge插件会绑定到生命周期的package阶段，在原始打包形成的jar包基础上进行重新打包，新形成的jar包不但包含应用类文件和配置文件，而且还会包含应用所依赖的jar包以及Springboot启动相关类，以此来满足Springboot独立应用的特性，原始的包会被修改为jar.original