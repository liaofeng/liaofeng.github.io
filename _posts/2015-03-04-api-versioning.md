---
layout: post
title: API Versioning
categories: 编程
tags: API
---

# API Versioning

## 是什么？

`API Versioning`即带上版本号的API。同一个API，通过version加以区别，可以返回不同的数据、实现不同的操作。

## 为什么？

为什么API需要带上版本号呢？  

### 之前我是这么做的

之前我写API，如果是通过`GET`获取资源，习惯通过字段（`field`）来定义返回哪些信息。  
比如，需要获取用户信息：`GET /api/users/1?fields=id&fields=name`，得到结果：

    {
        "id": 1,
        "name": "foo"
    }

(或者通过`GET /api/users/1?fields=id,name`实现，用`,`来分隔需要返回的字段)

随着时间的推移，App获取用户信息时，需要API返回用户的出生年月。  
如此一来，API增加一个字段`birthday`，对应的API请求变为：  
`GET /api/users/1?fields=id&fields=name&fields=birthday`  
得到结果：  

    {
        "id": 1,
        "name": "foo",
        "birthday": "1990-01-01T00:00:00"
    }

也就是说，每次API的变更都是通过新增`field`来实现。如果在App的改版中某个`field`不再使用了，API还要返回这个字段，因为老版本的App还在使用。

### 问题来了

增加field和通过fields指定返回的数据内容，很不够。

说两种它无法满足的情况：

* name返回的字段变了，需要返回：`name: {"firstname": "foo", "lastname": "bar"}`
* 某个API返回时的`key`写错了，比如`name`错写为`neam`

第一种情况，如果需要更改返回值的数据结构类型，新老API就无法兼容了。比如，原本返回字符串，要改为返回`dict`，原本返回`list`，要改为返回`dict`。  
第二种情况，如果某个返回值的`key`写错了，因为要兼容老版本的App，这个错误要么一直沿用下去，要么新增一个字段。

这两种情况，都是更改同一种资源的同一个属性，但都需要新增字段来返回不同的值，不够灵活，不够`RESTful`。

### 示例

Versionized API最大的好处是灵活。针对不同版本的API请求返回不同的数据。

如果某个App有两个已发布版本，`V1`和`V2`。发现`V1`App出现了Bug，此时要对所有来自`V1`App的Api请求做特殊处理，这时API请求带的版本号`version`就有作用了。比如：

    version = request.version
    if version == '1':
        return {
            "id": 1,
            "username": "foo"
        }
    elif version == '2':
        return {
            "id": 1,
            "username": "bar"
        }

## 如何实现

这是个纠结的问题。实现的方式很多，但在选择实现方式时，大伙在哪种实现方式最`RESTful`的时候，各有说法。

[Best practices for API versioning?](http://stackoverflow.com/questions/389169/best-practices-for-api-versioning)，这个问题对如何实现API Versioning有了深入的讨论。但搞笑的是，这个问题被关闭了，因为：

>
Many good questions generate some degree of opinion based on expert experience, but answers to this question will tend to be almost entirely based on opinions, rather than facts, references, or specific expertise.

如何实现Versioning API依赖于个人喜好。

下面我们看一下几种常用的实现方式，以获取用户信息的API `GET localhost:8000/api/users/1` 为例。

### Request Parameter

在API请求的时候带上`version`参数，标志所请求的API的版本号：

`curl "localhost:8000/api/users/1?version=1"`

这种方式实现起来最简单，需要修改的代码很少。但不够`RESTful`。

### HTTP Header

请求的时候，在HTTP Header带上版本信息。

先看一下不带version的Header：

`curl "localhost:8000/api/users/1" -H "Accept: application/json`

带上版本：

`curl "localhost:8000/api/users/1" -H "Accept: application/json; version=3"`

这种实现方式又叫`Content Negotiation`，在`Accept`头部信息带上请求API的版本号。  
还有一种实现方式，通过自定义头部信息，比如`XVersion`/`Accept-Version`。

比较常用的实现`Accept`头部信息长这样：

```
Accept: application/vnd.github[.version].param[+json]
```

Header中指定`vendor`、数据格式和版本，这是我倾向于选择的实现方案，因为`Accept`头部用来指定数据返回的格式，比如`application/json`、`application/javascript`。资源带上版本号，就好比如资源有不同的展示方式，比如HTML、JSON、XML。所以将version放在这里比较自然。

这种实现方式的缺点是在头部信息中指定version不够直观，不方便测试，也不便分享。

### URI

通过设计API的时候，在URI里植入版本信息，比如：

* `/api/v1/users/1`
* `/api/v2/users/1`

这好比将同一个API实现了两遍，不同的实现做不同的事情，没有技术难度。

有个小问题，如果我们有个已经发布出去的API：`/api/users/1`，需要给这个API的URI加上上述版本信息，那就只能新写一个API `/api/v1/users/1`，然后将 `/api/users/1` redirect到新的API。所以采用这种实现方式，最好在一开始设计API的时候就在URI中带上版本信息。


## 总结

总得来说API Versioning必不可少，一个健康发展的App，其Server端API的变更是不可避免的。实现起来没有技术难点，或许有些技术细节跟语言及框架有关。但哪种实现方式最好，没有答案。希望新出来的**HTTP 2**能够提供这方的支持，抽空看一下。
