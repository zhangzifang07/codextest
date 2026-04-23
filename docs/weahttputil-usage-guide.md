# WeaHttpUtil 使用文档

## 1. 文档目的

本文基于你提供的两个反编译类整理：

- `com.weaver.common.util.http.WeaHttpUtil`
- `com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApi`

目标不是只解释工具类源码，而是帮助你在泛微二开中稳定使用 `WeaHttpUtil` 请求第三方接口，包括：

- 获取 token
- 调用 JSON 接口
- 调用表单接口
- 追加请求头
- 处理返回值
- 做统一异常日志
- 参考北森接口写出自己的封装类

## 2. 先说结论

从你给出的源码可以确认，`WeaHttpUtil` 的定位很简单：

- 它是一个 HTTP 请求入口工厂。
- 通过 `createGet/createPost/createPut/createDelete/createPatch` 创建请求对象。
- 默认底层实现走的是 `HUTOOL`。
- 实际发请求时，主要依赖返回的 `WeaHttpRequest` 对象做链式调用。

你在二开里最常见的使用方式基本就是下面两种：

```java
String result = WeaHttpUtil.createPost(url)
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer " + token)
    .body(jsonBody)
    .execute()
    .body();
```

```java
String result = WeaHttpUtil.createPost(url)
    .header("Content-Type", "application/x-www-form-urlencoded")
    .bodyWithNameValuePair(paramMap)
    .execute()
    .body();
```

## 3. WeaHttpUtil 已确认的方法

根据反编译代码，可以确认 `WeaHttpUtil` 暴露了以下静态方法：

```java
public static WeaHttpRequest createGet(String url)
public static WeaHttpRequest createPost(String url)
public static WeaHttpRequest createPut(String url)
public static WeaHttpRequest createDelete(String url)
public static WeaHttpRequest createPatch(String url)
```

对应关系很好理解：

- `createGet(url)`：GET 请求
- `createPost(url)`：POST 请求
- `createPut(url)`：PUT 请求
- `createDelete(url)`：DELETE 请求
- `createPatch(url)`：PATCH 请求

## 4. WeaHttpRequest 常用链式能力

你给的源码没有包含 `WeaHttpRequest` 全量定义，但从北森调用代码里，已经可以确认以下方法可用：

- `header(String key, String value)`：设置请求头
- `body(String body)`：直接设置请求体字符串，常用于 JSON
- `bodyWithNameValuePair(Map<?, ?> paramMap)`：以表单键值对方式提交，常用于 `application/x-www-form-urlencoded`
- `execute()`：执行请求
- `execute().body()`：获取响应体字符串

也就是说，你现在可以把它理解成：

1. 先通过 `WeaHttpUtil` 创建请求对象。
2. 再链式追加 header、body。
3. 最后 `execute().body()` 取回字符串结果。

## 5. 标准调用模板

### 5.1 POST + JSON 请求

这是最常见的第三方接口调用方式，也是北森接口最常用的方式。

```java
import com.alibaba.fastjson.JSON;
import com.weaver.common.util.http.WeaHttpUtil;

Map<String, Object> paramMap = new HashMap<String, Object>();
paramMap.put("id", "1001");
paramMap.put("name", "test");

String result = WeaHttpUtil.createPost("https://api.xxx.com/order/create")
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer " + accessToken)
    .body(JSON.toJSONString(paramMap))
    .execute()
    .body();
```

适用场景：

- 请求体是 JSON
- 第三方接口要求 `Content-Type: application/json`
- 需要带 token

### 5.2 POST + 表单请求

获取 token 的接口非常常见地采用这个模式。你给出的 `getAccessToken` 就是标准示例。

```java
import com.weaver.common.util.http.WeaHttpUtil;

Map<String, String> paramMap = new HashMap<String, String>();
paramMap.put("grant_type", "client_credentials");
paramMap.put("app_key", appKey);
paramMap.put("app_secret", appSecret);

String result = WeaHttpUtil.createPost("https://openapi.xxx.com/token")
    .header("Content-Type", "application/x-www-form-urlencoded")
    .bodyWithNameValuePair(paramMap)
    .execute()
    .body();
```

适用场景：

- OAuth2 获取 token
- 老式表单提交接口
- 第三方要求 `x-www-form-urlencoded`

### 5.3 GET 请求

虽然你给的北森代码里没展示 GET 示例，但 `createGet` 是明确存在的，因此常规用法可按同样模式写。

```java
String result = WeaHttpUtil.createGet("https://api.xxx.com/user/detail?id=1001")
    .header("Authorization", "Bearer " + accessToken)
    .execute()
    .body();
```

如果接口需要很多查询参数，建议你自己先把 URL 拼好，再交给 `createGet`。

### 5.4 PUT / DELETE / PATCH 请求

这几个接口的调用风格与 POST 基本一致，只是入口方法不同。

```java
String result = WeaHttpUtil.createPut("https://api.xxx.com/user/1001")
    .header("Content-Type", "application/json")
    .body("{\"status\":\"disabled\"}")
    .execute()
    .body();
```

```java
String result = WeaHttpUtil.createDelete("https://api.xxx.com/user/1001")
    .header("Authorization", "Bearer " + accessToken)
    .execute()
    .body();
```

```java
String result = WeaHttpUtil.createPatch("https://api.xxx.com/user/1001")
    .header("Content-Type", "application/json")
    .body("{\"mobile\":\"13800000000\"}")
    .execute()
    .body();
```

## 6. 北森接口调用模式拆解

你贴出来的 `BeiSenApi` 非常适合作为二开模板。它的套路可以总结成 6 步。

### 6.1 组装参数

```java
Map<String, Object> paramMap = new HashMap<String, Object>();
paramMap.put("oIds", Arrays.asList(oId));
paramMap.put("enableTranslate", true);
```

注意点：

- 列表参数通常直接放 `List`
- 布尔参数直接放 `true/false`
- 时间参数通常先格式化为字符串

### 6.2 拼接接口地址

```java
String apiHref = href + "/TenantBaseExternal/api/v5/JobPost/GetByOIds";
```

建议：

- `href` 作为基础地址
- 路径单独拼接
- 不要把完整 URL 写死到每个方法里，方便切换环境

### 6.3 记录请求日志

```java
logger.info("apiHref：{}，paramMap：{}", apiHref, JSON.toJSONString(paramMap));
```

建议至少打印：

- 实际请求地址
- 实际请求参数
- 必要时打印关键 header，但不要打印敏感密钥

### 6.4 发起请求

```java
String data = WeaHttpUtil.createPost(apiHref)
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer " + accessToken)
    .body(JSON.toJSONString(paramMap))
    .execute()
    .body();
```

### 6.5 解析返回结果

```java
BeisenResultDto result = JSON.parseObject(
    data,
    new TypeReference<BeisenResultDto>() {},
    new Feature[0]
);

if (result.getCode() == null || !"200".equals(result.getCode())) {
    throw new Exception(result.getMessage());
}
```

这里说明一个重要经验：

- `WeaHttpUtil` 只负责把响应拿回来
- 成功失败判断，通常还是要你自己根据业务响应结构处理
- 不同第三方平台的成功码可能是 `200`、`0`、`success=true`，不要想当然

### 6.6 返回业务数据

```java
return result.getData();
```

如果是分页滚动接口，还会顺便更新游标：

```java
scrollId.setLength(0);
scrollId.append(result.getScrollId());
```

## 7. 建议你在二开里固定使用的封装套路

如果你经常对接三方接口，建议不要在业务代码里到处直接写 `WeaHttpUtil.createPost(...)`，而是封装成一个专门的 API 类。

例如：

```java
package com.xxx.integration.demo;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.weaver.common.util.http.WeaHttpUtil;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.HashMap;
import java.util.Map;

public class DemoOpenApi {

    private static final Logger logger = LoggerFactory.getLogger(DemoOpenApi.class);

    public String getAccessToken(String baseUrl, String appKey, String appSecret) throws Exception {
        Map<String, String> paramMap = new HashMap<String, String>();
        paramMap.put("grant_type", "client_credentials");
        paramMap.put("app_key", appKey);
        paramMap.put("app_secret", appSecret);

        String result;
        try {
            result = WeaHttpUtil.createPost(baseUrl + "/token")
                .header("Content-Type", "application/x-www-form-urlencoded")
                .bodyWithNameValuePair(paramMap)
                .execute()
                .body();
        } catch (Exception e) {
            logger.error("获取 token 请求异常，url:{}, param:{}", baseUrl + "/token", JSON.toJSONString(paramMap), e);
            throw e;
        }

        try {
            JSONObject json = JSON.parseObject(result);
            String token = json.getString("access_token");
            if (token == null || "".equals(token)) {
                throw new Exception("access_token 为空");
            }
            return token;
        } catch (Exception e) {
            logger.error("获取 token 解析失败，result:{}", result, e);
            throw e;
        }
    }

    public String postJson(String url, String accessToken, Map<String, Object> paramMap) throws Exception {
        try {
            logger.info("请求 url:{}, param:{}", url, JSON.toJSONString(paramMap));
            String result = WeaHttpUtil.createPost(url)
                .header("Content-Type", "application/json")
                .header("Authorization", "Bearer " + accessToken)
                .body(JSON.toJSONString(paramMap))
                .execute()
                .body();
            logger.info("响应 result:{}", result);
            return result;
        } catch (Exception e) {
            logger.error("POST JSON 请求失败，url:{}, param:{}", url, JSON.toJSONString(paramMap), e);
            throw e;
        }
    }
}
```

这个写法的好处：

- token 获取逻辑统一
- 日志格式统一
- 出错位置更清楚
- 后续切换第三方平台时，改动范围小

## 8. 常见请求场景模板

### 8.1 场景一：先取 token，再调用业务接口

```java
String token = getAccessToken(baseUrl, appKey, appSecret);

Map<String, Object> paramMap = new HashMap<String, Object>();
paramMap.put("userId", "1001");

String result = WeaHttpUtil.createPost(baseUrl + "/api/user/query")
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer " + token)
    .body(JSON.toJSONString(paramMap))
    .execute()
    .body();
```

### 8.2 场景二：根据 ID 列表批量查询

这个和北森 `GetByOIds` 很像。

```java
Map<String, Object> paramMap = new HashMap<String, Object>();
paramMap.put("oIds", Arrays.asList("1001", "1002", "1003"));

String result = WeaHttpUtil.createPost(baseUrl + "/api/demo/GetByIds")
    .header("Content-Type", "application/json")
    .body(JSON.toJSONString(paramMap))
    .execute()
    .body();
```

### 8.3 场景三：按时间窗滚动拉取数据

这个和北森 `GetByTimeWindow` 系列方法完全一致。

```java
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss");

Map<String, Object> paramMap = new HashMap<String, Object>();
paramMap.put("startTime", sdf.format(startTime));
paramMap.put("stopTime", sdf.format(stopTime));
paramMap.put("scrollId", scrollId.toString());
paramMap.put("capacity", 100);

String result = WeaHttpUtil.createPost(baseUrl + "/api/demo/GetByTimeWindow")
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer " + token)
    .body(JSON.toJSONString(paramMap))
    .execute()
    .body();
```

如果三方接口返回新的 `scrollId`，就继续更新它，直到没有数据为止。

### 8.4 场景四：查单条详情

```java
String result = WeaHttpUtil.createGet(baseUrl + "/api/order/detail?id=20260423001")
    .header("Authorization", "Bearer " + token)
    .execute()
    .body();
```

## 9. 返回值处理建议

`WeaHttpUtil` 返回给你的通常是原始字符串，所以落地时建议固定分 3 层处理。

### 9.1 第一层：请求是否发成功

这一层主要靠 `try-catch` 捕获连接异常、超时异常、DNS 异常、证书异常等。

```java
String data;
try {
    data = WeaHttpUtil.createPost(url)
        .header("Content-Type", "application/json")
        .body(JSON.toJSONString(paramMap))
        .execute()
        .body();
} catch (Exception e) {
    logger.error("请求第三方接口异常，url:{}, param:{}", url, JSON.toJSONString(paramMap), e);
    throw e;
}
```

### 9.2 第二层：返回报文是否能解析

```java
JSONObject json;
try {
    json = JSON.parseObject(data);
} catch (Exception e) {
    logger.error("返回报文不是合法 JSON，result:{}", data, e);
    throw e;
}
```

### 9.3 第三层：业务码是否成功

```java
if (!"200".equals(json.getString("code"))) {
    throw new Exception("第三方接口返回失败：" + json.getString("message"));
}
```

这三层要分开写，不然后期排查非常难。

## 10. 推荐的异常与日志写法

北森代码的写法整体是值得借鉴的，建议你保留下面这些信息：

- 接口名称
- 请求 URL
- 请求参数
- 原始返回报文
- 完整异常堆栈

推荐模板：

```java
try {
    // 发请求
} catch (Exception e) {
    logger.error("第三方接口调用异常; url:{}; param:{}", url, JSON.toJSONString(paramMap), e);
    throw e;
}

try {
    // 解析返回
} catch (Exception e) {
    logger.error("第三方接口返回解析异常; url:{}; param:{}; result:{}", url, JSON.toJSONString(paramMap), data, e);
    throw e;
}
```

不要只打印 `e.getMessage()`，尽量把异常对象本身也传给日志框架。

## 11. Content-Type 怎么选

这是二开里最容易踩坑的地方之一。

### 11.1 JSON 接口

```java
.header("Content-Type", "application/json")
.body(JSON.toJSONString(paramMap))
```

适用于：

- 大多数现代开放平台业务接口
- 参数结构复杂
- 有嵌套对象、数组

### 11.2 表单接口

```java
.header("Content-Type", "application/x-www-form-urlencoded")
.bodyWithNameValuePair(paramMap)
```

适用于：

- token 获取
- OAuth2 标准认证
- 历史系统接口

如果 `Content-Type` 与请求体格式不一致，很多第三方接口会直接报参数错误。

## 12. Authorization 怎么带

从北森代码可以确认，Bearer Token 的写法如下：

```java
.header("Authorization", "Bearer " + accessToken)
```

常见情况：

- Bearer Token：`Authorization: Bearer xxx`
- 自定义 Token：`header("token", token)`
- AppKey/AppSecret：有时放在 header，有时放在 body，按对方文档来

不要凭经验乱写，尤其注意：

- 是否需要 `Bearer ` 前缀
- `Authorization` 首字母大小写
- token 是否有过期时间

## 13. 时间参数怎么传

北森接口用了：

```java
new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss")
```

示例：

```java
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss");
paramMap.put("startTime", sdf.format(startTime));
paramMap.put("stopTime", sdf.format(stopTime));
```

建议：

- 时间格式严格跟第三方文档保持一致
- 不要自己脑补时区
- 如果对方要求毫秒、UTC、时间戳，必须按要求改

## 14. 二开实战建议

### 14.1 统一封装第三方接口类

建议每个第三方平台一个独立类，例如：

- `XXXOpenApi`
- `BeiSenApi`
- `DingTalkApi`
- `FeishuApi`

不要把所有平台的调用都堆在一个工具类里。

### 14.2 统一封装 token 获取逻辑

很多平台都有 token 缓存、过期刷新、失败重试需求。哪怕你现在先写最简版，也建议单独封装方法。

### 14.3 参数对象尽量明确

如果接口很复杂，建议不要一直用 `Map<String, Object>`，可以定义 DTO：

```java
public class DemoQueryDto {
    private String userId;
    private Boolean withDisabled;
}
```

这样更容易维护。

### 14.4 不要把敏感信息打进日志

尤其是：

- appSecret
- accessToken
- 身份证号
- 手机号

建议打印脱敏值或只打印前几位。

### 14.5 统一判断第三方成功码

不要让控制层、业务层到处自己判断：

```java
if ("200".equals(code)) { ... }
```

建议在接口封装层就把失败转成异常抛出去。

## 15. 常见坑位清单

### 15.1 `Content-Type` 写错

表现：

- 对方接口报“参数为空”
- 对方接口报“不支持的媒体类型”

### 15.2 token 头格式不对

表现：

- 401
- 403
- “invalid token”

### 15.3 参数名拼错

表现：

- 返回成功但数据为空
- 返回“字段不存在”

### 15.4 JSON 字段类型不对

例如接口要数组，你传成了字符串：

```java
paramMap.put("oIds", Arrays.asList(oId));   // 正确
paramMap.put("oIds", oId);                  // 可能错误
```

### 15.5 日期格式不对

表现：

- 返回时间窗为空
- 返回参数校验失败

### 15.6 只打印异常，不打印请求上下文

后果：

- 线上问题难排查
- 不知道到底请求了哪个地址、传了什么参数

## 16. 一个通用的三方调用模板

你后面二开时，可以直接按这个模板改。

```java
public String callThirdApi(String url, String token, Map<String, Object> paramMap) throws Exception {
    String result;
    try {
        logger.info("request url:{}, param:{}", url, JSON.toJSONString(paramMap));
        result = WeaHttpUtil.createPost(url)
            .header("Content-Type", "application/json")
            .header("Authorization", "Bearer " + token)
            .body(JSON.toJSONString(paramMap))
            .execute()
            .body();
        logger.info("response result:{}", result);
    } catch (Exception e) {
        logger.error("调用第三方接口异常, url:{}, param:{}", url, JSON.toJSONString(paramMap), e);
        throw e;
    }

    try {
        JSONObject json = JSON.parseObject(result);
        String code = json.getString("code");
        if (!"200".equals(code)) {
            throw new Exception("接口返回失败，code=" + code + ", message=" + json.getString("message"));
        }
        return result;
    } catch (Exception e) {
        logger.error("解析第三方接口返回失败, url:{}, result:{}", url, result, e);
        throw e;
    }
}
```

## 17. 基于当前源码可得出的最终认知

### 17.1 已经可以确定的部分

- `WeaHttpUtil` 是泛微对 HTTP 请求的一层简单封装入口
- 默认走 `HUTOOL`
- 支持 `GET/POST/PUT/DELETE/PATCH`
- 常用链式方法至少包括 `header`、`body`、`bodyWithNameValuePair`、`execute().body()`
- 足够完成绝大多数三方接口集成

### 17.2 目前还不能完全确定的部分

因为你还没有贴出 `WeaHttpRequest`、`HutoolHttpRequest`、`ApacheHttpRequest` 的完整源码，所以以下能力暂时不能 100% 下结论：

- 是否支持连接超时、读取超时单独设置
- 是否支持文件上传
- 是否支持代理配置
- 是否支持 HTTPS 证书跳过校验
- 是否支持自定义 cookie
- `execute()` 返回对象的完整结构

如果你后面再反编译出这几个类，我可以继续帮你把文档补成“完整 API 手册版”。

## 18. 建议你下一步怎么用

如果你准备开始实际二开，推荐顺序如下：

1. 先按本文模板写一个你自己的 `XXXOpenApi` 封装类。
2. 优先打通一个最简单的 token 接口。
3. 再打通一个 POST JSON 业务接口。
4. 最后统一补日志、异常、返回码判断。

这样你会最快形成稳定模板，后面接别的第三方接口基本就是复制改参数。

---

如果你愿意，我下一步可以继续帮你补两份内容：

- 一份“`WeaHttpUtil` 对接三方接口最佳实践代码模板”
- 一份“北森 `BeiSenApi` 全方法中文说明表”
