## [首页](https://kingkh1995.github.io/blog/)
> RESTful API (Representational State Transfer)

### 资源（Resources）
每种资源对应一个特定的URI

### 表现层（Representation）
URI应该只代表"资源"的位置，它的具体表现形式，应该在HTTP请求的头信息中用Accept和Content-Type字段指定，这两个字段才是对"表现层"的描述。

### 状态转化（State Transfer）
客户端要操作服务器，必须通过某种手段，让服务器端发生"状态转化"，四种基本操作：GET用来获取资源，POST用来新建资源，PUT用来更新资源（完全更新），DELETE用来删除资源。不常用的操作：PATCH用来部分更新资源，HEAD用来获取资源的元数据，OPTIONS用来查询与资源相关的选项和需求。

### 错误处理（Error handling）
服务器向用户返回HTTP状态码，如果状态码是4xx，就应该向用户返回出错信息。一般来说，返回的信息中将error作为键名，出错信息作为键值即可。

### RESTful架构：
  1. 每一个URI代表一种资源；
  1. 客户端和服务器之间，传递这种资源的某种表现层；
  1. 客户端通过四个HTTP动词，对服务器端资源进行操作，实现"表现层状态转化"。

### 设计误区
  1. 就是URI包含动词，应该把动作做成一种资源，改成名词的形式。
  1. 就是在URI中加入版本号，版本号可以在HTTP请求头信息的Accept字段中进行区分
