# 第1章 开方的GitHub API

介绍如何使用GitHub API读写数据。

## 1.1 cURL

cURL是一个命令行HTTP工具。

GitHub API最基本的端点： https//api.github.com

`$curl https://api.github.com`

```json
{
"current_user_url": "https://api.github.com/user",
"current_user_authorizations_html_url":
"https://github.com/settings/connections/applications{/client_id}",
"authorizations_url": "https://api.github.com/authorizations",
"code_search_url":
"https://api.github.com/search/code?q={query}{&page,per_page,sort,order}",
"emails_url": "https://api.github.com/user/emails",
"emojis_url": "https://api.github.com/emojis",
...
}
```

## 1.2 列举API路径

GitHub API是超媒体API。

超媒体API的响应中，包含：
1. 一个映射，列出了接下来可能会发起请求的地址。（单超媒体API可能会动态调整所用的端点，不过它的目标之一是以不重新编写代码为前提）
2. 不仅包含URL，还有为URL提供参数的方式。

## 1.3 JSON格式

JSON.org网站: http://www.json.org/
json键值对只能使用双引号。

### 1.3.1 在命令行中解析JSON

使用jq工具：https://stedolan.github.io/jq/

`$ https://api.github.com | jq '.current_user_url'`
就会返回"https://api.github.com/user"，
.表示键过滤器，
注意可以使用-s选项开启静默模式。

*更复杂的请求：*

`$ curl -s https://api.github.com/users/xrd/repos`

```jason
[
{
"id": 19551182,
"name": "a-gollum-test",
"full_name": "xrd/a-gollum-test",
"owner": {
"login": "xrd",
"id": 17064,
"avatar_url":
"https://avatars.githubusercontent.com/u/17064?v=3",
...
}
]
```
`$ curl -s https://api.github.com/users/xrd/repos | jq '.[0].owner.id'`
*17064*

### 1.3.2 cURL的调试开关

-i 打印请求首部
-v 打印请求和响应首部(>请求，<响应)

## 1.4 重要的首部

三个指明API频率限制的首部：
X-RateLimit-Limit、X-RateLimit-Remaining和X-RateLimit-Reset。

获取文本或blob内容时，要用X-GitHub-Media-Type。

发起请求时，可以在请求中发送Accpet首部，指明想使用的格式。

## 1.5 跟随超媒体API

使用API基端点返回的“映射”，手动生成另一个请求：

`$ curl -i https://api.github.com/`

```json
...
{
"current_user_url": "https://api.github.com/user",
...
"organization_url": "https://api.github.com/orgs/{org}",
...
}
```

使用组织的URL，把占位符换成"github"：

`$ curl https://api.github.com/orgs/github`
会返回GitHub的一些信息。

## 1.6 身份验证

两种方式：
1. 用户名和密码；
2. OAuth令牌。

### 1.6.1 用户名和密码验证

用户名验证使用HTTP基本验证实现，在cURL中使用-u标志指定。
`$ curl -u xrd https://api.github.com/rate_limit`

### 1.6.2 OAuth

OAuth是一种身份验证机制，可以指定让服务为OAuth令牌开方那些功能；还可以颁发多个令牌，与不同客户端绑定。

[TODO]

## 1.7 状态码

### 1.7.1 成功（200或201）

200：请求的目标地址和相关的参数正确
201：表示成功在服务器中创建了内容

### 1.7.2 不合规的JSON（400）

400：请求中发送的JSON无效

-d 开启载荷。
-X POST：表示修改请求方式为POST。

### 1.7.3 错误的JSON（422）

422：如果请求中有任何无效的字段，比如缺少某个键。

### 1.7.4 成功创建（201）

JSON中不需要空格。

### 1.7.5 完全没变化（304）

304：与200作用类似，告诉客户端请求成功，不过它还多提供了一些信息：告诉客户端自上次请求依赖数据没有变化。

### 1.7.6 GitHub API的频率限制

GitHub API请求频率限制：
匿名请求60/h
验证后请求5000/h

这里是指核心频率限制，还有搜索频率限制，分别为5/m 和 20/m。

### 1.7.7 获知频率限制

向/rate_limit发起GET请求即可。

## 1.8 使用条件请求规避频率限制

If-Modified-Since和If-None-Match条件首部，可以让GitHub返回304响应。如果返回的是304响应，则不会扣除频率限制。

`$ curl -i https://api.github.com/repos/twbs/bootstrap -H "If-Modified-Since: Sun, 11 Aug 2013 19:48:59 GMT"`

ETAG是一个HTTP首部，用于确定之前缓存的内容是否为最新版。