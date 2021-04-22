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

### 设计要点
  1. 将版本号放在path中，如/api/v1；
  1. 公共请求参数放在Headers中，如token，客户端应用标识等；
  1. 不同的语境下返回不同的http状态码，正常返回2XX，客户端请求错误返回4XX，服务器或程序错误返回5XX，统一错误提示信息；
  1. 如果有资源从属关系，必须在path中声明上级资源，因为如果上级资源不存在了则不允许的访问下级资源，如/company/{company_id}/employee/{employee_id}；
  1. 如果存在资源索引关系，也在path中声明，如/company/{company_id}/department/{department_id}/employee/{employee_id}；
  1. 其他参数放到queryString中，这些参数应该都是可选参数，如offset，limit，orderby等；
  1. GET接口如果参数过于复杂，可以使用POST，或者将queries定义为一种资源，先通过POST保存queries，再通过query_id去获取查询结果；
  1. 资源名称使用复数还是单数，建议单个资源使用单数，多个资源使用复数。