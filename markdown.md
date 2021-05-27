# [首页](https://kingkh1995.github.io/blog/)

[官方中文文档](https://markdown-zh.readthedocs.io/en/latest/)

***

## 段落和换行

使用一个或多个空行划分不同的段落

***

## 标题

```
// 使用多个#表示多级标题

# 一级标题
## 二级标题
### 三级标题
...

// 任意长度的 = 和 - 也可以表示一级标题和二级标题

这是一级标题
=============
这是二级标题
-------------
```

***

## 引用

```
> 这是一级引用
>> 这是二级引用
```

> 这是一级引用
>> 这是二级引用

***

## 列表

```
// 使用 * + - 都可以表示列表

* 文本1
+ 文本2
- 文本3

// 使用数字时会自动递增

1. 文本1
1. 文本2
1. 文本3
```

* 文本1
+ 文本2
- 文本3

1. 文本1
1. 文本2
1. 文本3

***

## 水平线

```
// 一行中只有三个以上连续或不连续的星号或下划线就可以生成一条水平线

文本
***
文本
___
```

文本
***
文本
___

***

## 链接

```
// 相同服务器下的资源使用相对路径或绝对路径，可选的链接标题使用引号包围

这是一个[内部资源链接](/index)

这是一个 [内联链接](https://www.baidu.com/ '可选的内联链接标题') 

这是一个 [引用链接][baidu]

// 引用链接需要定义链接标签，要求单独占一行，且不会在页面中展示

[baidu]: https://www.baidu.com/ '这是百度的标题'
```
这是一个[内部资源链接](/index)

这是一个 [内联链接](https://www.baidu.com/ '可选的内联链接标题') 

这是一个 [引用链接][baidu]

[baidu]: https://www.baidu.com/ '这是百度的标题'

***

## 强调

```
*这是斜体效果*

**这是加粗的效果**

***这是加粗又加斜的效果***
```

*这是斜体效果*

**这是加粗的效果**

***这是加粗的效果***

***

## 代码和代码块

只需要空行隔开并缩进一个制表符就可以表示普通代码块，使用 ``` 支持特定的格式的代码块。

```
    `这是普通代码` ``这也是普通代码``

        这是普通代码块

    ```
    普通代码块
    ```

    ```sql
    select * from table where id = 1;
    ```

    ```java
    // Hello World!
    public static void main(String[] args) {
        System.out.println("Hello World!");
    }
    ```
```

`这是普通代码` ``这也是普通代码``

    这是普通代码块

```
普通代码块
```

```sql
select * from table where id = 1;
```

```java
// Hello World!
public static void main(String[] args) {
    System.out.println("Hello World!");
}
```
***

## 图片

与链接类似，可以使用内联和引用的方式，如需要指定大小只能使用\<img>标签。

```text
![百度图片](https://www.baidu.com/img/PCtm_d9c8750bed0b3c7d089fa7d55720d6cf.png "百度图片")
```

![百度图片](https://www.baidu.com/img/PCtm_d9c8750bed0b3c7d089fa7d55720d6cf.png "百度图片")

***

## 反斜杠转义

语法中特殊的字符使用反斜杠 '\\' 转义为字面量


## 表格（部分不支持）

```
// 使用 | 来分隔不同的单元格，使用 - 来分隔表头和其他行

:--- 代表左对齐
:--: 代表居中对齐
---: 代表右对齐

|Tables|Are|Cool|
|:---|:--:|--:|
|col 3 is|right-aligned|$1600|
|col 2 is|centered|$12|
|zebra stripes|are neat|$1|
```
|Tables|Are|Cool|
|:---|:--:|--:|
|col 3 is|right-aligned|$1600|
|col 2 is|centered|$12|
|zebra stripes|are neat|$1|
