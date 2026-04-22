# Arthas 使用手册（含 E10 场景实战）

本文档基于你现有的 E10 使用记录整理，目标不是只会敲几条命令，而是帮助你形成一套稳定的 Arthas 排查方法。

适用场景：

1. 线上或测试环境快速确认“部署的代码到底是不是我以为的那份代码”。
2. 阅读运行中类的源码、方法签名、类加载信息。
3. 排查接口入参、出参、异常、慢调用、调用来源。
4. 定位线程热点、CPU 飙高、对象实例、Spring Bean。

## 1. Arthas 是什么

Arthas 是一个 Java 诊断工具，可以在不重启应用的情况下，直接连接到运行中的 JVM 进程，查看类、方法、线程、JVM 状态，并对方法调用做实时观测。

对日常排查最有价值的点主要有三类：

1. 看运行中的类和方法。
2. 看方法执行时的入参、返回值、异常、耗时。
3. 看线程、内存、类加载器等运行态信息。

## 2. 快速上手

### 2.1 找到目标进程

```bash
# 方式 1：按进程名搜索
ps -ef | grep E10

# 方式 2：如果机器安装了 jdk，也可以直接列出 Java 进程
jps -l
```

### 2.2 连接目标进程

```bash
# 直接指定 pid 连接
java -jar arthas-boot.jar 1111176

# 你在 E10 中的实际示例
/data/weaver/jdk/bin/java -jar /data/weaver/e-monitor/app/arthas/arthas-boot.jar 1111176
```

### 2.3 进入后最常用的基础命令

```bash
# 查看帮助
help

# 查看系统总览，包括内存、线程、GC、Tomcat 等摘要
dashboard

# 退出当前 Arthas 客户端，但不关闭目标 JVM 中的 Arthas 服务
quit

# 停止 Arthas 服务
stop
```

## 3. 常用命令与英文全拼

下面这张表建议反复看。很多命令不是缩写，但知道英文原意后，理解会快很多。

| 命令 | 英文全称 | 含义 | 最常用场景 |
| --- | --- | --- | --- |
| `sc` | `Search Class` | 搜索类 | 找类、确认类是否加载、查看类加载器 |
| `sm` | `Search Method` | 搜索方法 | 看类里有哪些方法、方法签名 |
| `jad` | `Java Decompiler` | 反编译运行中类 | 看线上实际执行代码 |
| `watch` | `watch` | 观察方法执行数据 | 看入参、返回值、异常 |
| `trace` | `trace` | 跟踪方法内部调用耗时 | 看慢在哪一层调用 |
| `stack` | `stack` | 查看调用栈 | 看是谁调用了当前方法 |
| `monitor` | `monitor` | 监控方法统计信息 | 看 QPS、成功率、平均耗时 |
| `tt` | `Time Tunnel` | 时间隧道 | 记录调用现场，事后回放/分析 |
| `thread` | `thread` | 查看线程信息 | 查 CPU 高线程、死锁、阻塞 |
| `dashboard` | `dashboard` | 仪表盘 | 进程总体健康度概览 |
| `vmtool` | `Virtual Machine Tool` | JVM 工具命令 | 查实例对象、配合 Spring Bean 排查 |
| `ognl` | `Object-Graph Navigation Language` | 对象图导航表达式语言 | 取对象字段、调用无副作用方法 |

补充几个高频术语：

| 术语 | 全称 | 说明 |
| --- | --- | --- |
| `JVM` | `Java Virtual Machine` | Java 虚拟机 |
| `JVMTI` | `JVM Tool Interface` | JVM 提供给诊断工具的接口，`vmtool` 依赖它 |
| `RT` | `Response Time` 或 `Run Time` | 在 Arthas 统计里通常理解为方法耗时 |

## 4. 先学会一套标准排查顺序

遇到问题时，建议按下面顺序走，不容易乱：

1. 先用 `sc` 确认类是否真的加载、由谁加载。
2. 再用 `sm` 看方法签名，避免方法重载时打错。
3. 用 `jad` 看运行中代码，确认线上代码和本地源码是否一致。
4. 用 `watch` 看入参、返回值、异常。
5. 如果不知道调用来源，用 `stack`。
6. 如果方法慢，用 `trace` 或 `monitor`。
7. 如果问题偶发且现场容易丢，用 `tt` 抓现场。
8. 如果怀疑线程或对象问题，再用 `thread`、`vmtool`。

这套顺序几乎覆盖大多数接口排查场景。

## 5. 源码阅读：从类到方法到运行中代码

### 5.1 `sc`：先找到类

`sc` 的核心用途是确认“这个类到底有没有被 JVM 加载”。

```bash
# 模糊搜索类名
sc *BeiSen*

# 查看类的详细信息
sc -d com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApiUtil
```

命令说明：

```bash
# sc = Search Class
# -d = detail，显示详细信息，比如类加载器、接口、注解、字段等
```

实战价值：

1. 判断类是否真的存在于当前进程。
2. 判断是否有多个同名类被不同类加载器加载。
3. 为后续 `jad`、`watch`、`trace` 提供准确类名。

### 5.2 `sm`：再看方法

```bash
# 查看类中有哪些方法
sm com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApiUtil

# 模糊搜索方法
sm com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApiUtil getAccessToken
```

命令说明：

```bash
# sm = Search Method
# 第一个参数是类名
# 第二个参数可以是方法名或方法匹配表达式
```

实战价值：

1. 看方法是否存在。
2. 看是否有重载。
3. 确认 `watch`、`trace`、`stack` 要监控的目标方法名。

### 5.3 `jad`：看运行中的真实代码

`jad` 是 Arthas 最实用的命令之一。它不是看你本地源码仓库，而是直接把 JVM 里已经加载的 class 反编译出来。

```bash
# 反编译类
jad com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApiUtil

# 你原始记录中的另一个例子
jad com.weaver.ebuilder.form.view.list.dao.datarecycle.EbCommonDataRecycle
```

命令说明：

```bash
# jad = Java Decompiler
# 用于反编译 JVM 中已经加载的类
```

实战价值：

1. 确认线上部署的代码是否是最新版本。
2. 确认 if/else、异常处理、日志逻辑是否真的存在。
3. 排查“本地有改动，但线上看起来没生效”的问题。

### 5.4 源码阅读推荐流程

```bash
# 1. 先找类
sc *BeiSen*

# 2. 看类详细信息
sc -d com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApiUtil

# 3. 看方法
sm com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApiUtil

# 4. 反编译类
jad com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApiUtil
```

## 6. 接口输入输出排查：`watch` 是主力命令

如果你的目标是排查接口请求传了什么、方法内部收到什么、最终返回了什么、有没有异常，`watch` 是最常用的命令。

### 6.1 最常见写法

```bash
# 查看入参、返回值、异常，抓 5 次
watch com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApiUtil getAccessToken '{params,returnObj,throwExp}' -n 5
```

这条命令就是你当前记录里的核心命令，建议记熟。

### 6.2 `watch` 的常用观察点

```bash
# 方法执行前看入参
watch com.example.UserService createUser '{params}' -b -n 5 -x 2

# 方法正常返回后看入参和返回值
watch com.example.UserService createUser '{params,returnObj}' -s -n 5 -x 2

# 方法抛异常时看入参与异常对象
watch com.example.UserService createUser '{params,throwExp}' -e -n 5 -x 2

# 同时看当前对象、入参、返回值、异常
watch com.example.UserService createUser '{target,params,returnObj,throwExp}' -n 5 -x 2
```

命令说明：

```bash
# watch = 观察方法执行数据
# -b = before，在方法执行前观察
# -s = success，在方法正常返回后观察
# -e = exception，在方法抛异常后观察
# -n 5 = 最多输出 5 次，强烈建议在线上加这个限制
# -x 2 = 展开对象属性层级为 2，层级越大输出越详细
```

### 6.3 `watch` 里最常见的内置变量

| 表达式 | 含义 | 适用场景 |
| --- | --- | --- |
| `params` | 方法入参数组 | 看调用方传了什么 |
| `returnObj` | 返回值对象 | 看接口输出 |
| `throwExp` | 异常对象 | 看抛了什么异常 |
| `target` | 当前方法所在对象实例 | 看成员变量状态 |
| `#cost` | 本次调用耗时，单位毫秒 | 只关注慢调用 |

### 6.4 按条件过滤，避免输出过多

```bash
# 只看耗时超过 200ms 的调用
watch com.example.UserService createUser '{params,returnObj,#cost}' '#cost > 200' -n 10 -x 2

# 只看第一个参数不为空的调用
watch com.example.UserService createUser '{params,returnObj}' 'params[0] != null' -n 10 -x 2
```

命令说明：

```bash
# 第二个单引号里的内容是 OGNL 条件表达式
# 只有条件为 true 时，当前调用才会输出
```

### 6.5 `watch` 排查接口的推荐姿势

当你怀疑“接口入参到了 controller/service 后变了”、“第三方接口返回不符合预期”、“某次调用报错但日志看不清”时，可以按下面顺序来：

```bash
# 1. 先看进入方法前收到的参数
watch com.example.ApiService callExternal '{params}' -b -n 3 -x 2

# 2. 再看正常返回后的结果
watch com.example.ApiService callExternal '{params,returnObj}' -s -n 3 -x 2

# 3. 如果有异常，再看异常对象
watch com.example.ApiService callExternal '{params,throwExp}' -e -n 3 -x 3
```

如果你明明写了条件表达式却一直没输出，可以加 `-v` 看条件判断过程：

```bash
# 用 -v 输出条件表达式的求值过程，适合排查“为什么没命中”
watch com.example.ApiService callExternal '{params,returnObj}' 'params[0] != null' -n 3 -x 2 -v
```

### 6.6 `watch` 的几个关键注意点

1. `-b` 阶段还没有返回值，所以这时看不到 `returnObj`。
2. `params` 在方法执行前后可能不同，尤其是参数对象被修改时。
3. 生产环境里不要一上来就 `-x 4`、`-x 5`，很容易把输出打爆。
4. 一定尽量加 `-n` 或条件过滤。

## 7. `stack`：不知道是谁调过来的，就看调用栈

如果你已经知道问题出在某个方法，但不知道到底是谁触发了它，`stack` 非常适合。

```bash
# 查看谁在调用 getAccessToken
stack com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApiUtil getAccessToken

# 只看耗时超过 100ms 的调用栈
stack com.example.UserService createUser '#cost > 100'
```

命令说明：

```bash
# stack = 调用栈
# 主要用来回答“是谁调了我”
```

适合回答的问题：

1. 这个方法到底是 controller 调的，还是定时任务调的？
2. 是哪个业务入口触发了这段逻辑？
3. 某个工具类为什么在奇怪的场景下被调用了？

## 8. `trace`：方法慢，到底慢在哪一层

`trace` 的目标不是简单看入参，而是看方法内部每一层调用耗时。

```bash
# 跟踪方法内部调用耗时
trace com.example.UserService createUser

# 只看耗时超过 100ms 的调用
trace com.example.UserService createUser '#cost > 100'

# 只抓 5 次，避免输出太多
trace com.example.UserService createUser -n 5
```

命令说明：

```bash
# trace = 跟踪调用路径耗时
# #cost = 当前匹配调用的耗时，单位毫秒
```

适合回答的问题：

1. 慢是在查数据库，还是调第三方接口？
2. 是当前方法自身慢，还是它调用的下游方法慢？
3. 同一个方法偶尔慢，慢的时候到底卡在哪一步？

## 9. `monitor`：统计方法成功率、失败率、平均耗时

如果你不是想看某一次调用，而是想观察一段时间内的方法整体表现，用 `monitor`。

```bash
# 每 5 秒输出一次监控统计
monitor -c 5 com.example.UserService createUser
```

命令说明：

```bash
# monitor = 监控统计
# -c 5 = 每 5 秒统计并输出一次
```

通常会看到这些指标：

1. `total`：总调用次数。
2. `success`：成功次数。
3. `fail`：失败次数。
4. `avg-rt`：平均耗时。
5. `fail-rate`：失败率。

适合回答的问题：

1. 最近 1 分钟这个方法是否经常失败？
2. 平均耗时有没有突然升高？
3. 某个接口是不是持续低概率报错？

## 10. `tt`：偶发问题抓现场，事后再分析

`tt` 是 `Time Tunnel`，可以把某次方法调用现场保存下来，之后慢慢分析。

这是排查偶发问题、间歇性报错的神器。

### 10.1 先录制调用现场

```bash
# 记录 10 次调用
tt -t com.example.UserService createUser -n 10
```

命令说明：

```bash
# tt = Time Tunnel
# -t = 记录调用现场
# -n 10 = 最多记录 10 次
```

### 10.2 查看记录列表

```bash
# 查看已经记录下来的调用
tt -l
```

### 10.3 看某一次调用详情

```bash
# 查看索引为 1000 的那次调用详情
tt -i 1000
```

### 10.4 对录制到的调用再做表达式分析

```bash
# 查看该次调用的入参与返回值
tt -i 1000 -w '{params,returnObj,throwExp}' -x 2
```

### 10.5 删除记录，避免占内存

```bash
# 删除某条记录
tt -d 1000

# 删除全部记录
tt --delete-all
```

`tt` 的典型使用场景：

1. 某个接口偶发返回空值，但你当时没看清现场。
2. 某次请求抛异常，但日志里缺字段。
3. 想回看某次调用到底传了什么对象。

注意：

1. `tt` 会保存调用快照，不要长时间无限制录制。
2. 用完记得删记录。
3. 如果涉及重放能力，要非常谨慎，不要对有副作用的方法做随意重放。

## 11. `thread` 与 `dashboard`：排查线程和 CPU

### 11.1 先看总览

```bash
# 查看 JVM 总览
dashboard
```

建议重点看：

1. 线程数是否异常升高。
2. GC 是否频繁。
3. 堆内存是否接近上限。

### 11.2 看最忙的线程

```bash
# 查看最忙的 3 个线程
thread -n 3

# 每隔 1000ms 采样一次，再看最忙线程
thread -n 3 -i 1000
```

### 11.3 看阻塞线程或死锁线索

```bash
# 查找阻塞线程
thread -b
```

适合回答的问题：

1. CPU 飙高到底是哪几个线程导致的？
2. 是业务线程卡住了，还是锁竞争严重？
3. 某个任务是不是一直在阻塞？

## 12. `vmtool`：看对象实例，必要时排查 Spring Bean

如果你已经知道类名，但想看 JVM 中到底有哪些实例对象，可以用 `vmtool`。

```bash
# 获取某个类的实例
vmtool --action getInstances --className com.example.UserService --limit 10
```

命令说明：

```bash
# vmtool = Virtual Machine Tool
# --action getInstances = 获取某个类当前在 JVM 中的实例对象
# --limit 10 = 最多返回 10 个实例
```

进一步如果你在 Spring 环境里，常见做法是先取到 `ApplicationContext`，再拿 Bean 做查看。但这一步要求你对类加载器和上下文更熟，建议先在测试环境熟悉后再用于生产。

## 13. OGNL 表达式怎么理解

Arthas 很多命令都支持 OGNL 表达式，比如 `watch`、`tt -w`。

`OGNL` 全称是 `Object-Graph Navigation Language`，可以理解为“用表达式读取对象字段和对象关系”的小语言。

常见示例：

```bash
# 查看第一个参数
watch com.example.UserService createUser 'params[0]' -n 3

# 查看第一个参数中的 name 字段
watch com.example.UserService createUser 'params[0].name' -n 3

# 查看返回值中的 code 和 message
watch com.example.UserService createUser '{returnObj.code,returnObj.message}' -n 3

# 查看当前对象中的成员变量
watch com.example.UserService createUser 'target.someField' -n 3
```

理解要点：

1. `params` 是数组，所以常写成 `params[0]`、`params[1]`。
2. `target` 是当前对象实例，相当于 Java 里的 `this`。
3. 表达式越复杂，排查前越要确认不会触发副作用。

## 14. 源码阅读、接口排查、性能排查的命令对照

| 目标 | 优先命令 | 说明 |
| --- | --- | --- |
| 看类有没有加载 | `sc` | 先确认类是否存在 |
| 看类的实际代码 | `jad` | 看运行中 class 的反编译结果 |
| 看类里有哪些方法 | `sm` | 防止方法名写错或有重载 |
| 看接口入参 | `watch -b` | 在执行前观察 |
| 看接口返回值 | `watch -s` | 正常返回后观察 |
| 看异常 | `watch -e` | 只在抛异常时输出 |
| 看谁调用了当前方法 | `stack` | 看调用来源 |
| 看慢调用卡在哪一层 | `trace` | 看内部调用耗时 |
| 看一段时间的整体统计 | `monitor` | 看成功率与平均耗时 |
| 抓偶发问题现场 | `tt` | 先录制，再回看 |
| 看线程热点 | `thread` | 看 CPU 高线程或阻塞 |

## 15. E10 场景下的实战命令模板

下面这组命令是对你原始笔记的系统化整理。

### 15.1 找进程并连接

```bash
# 找到 E10 进程
ps -ef | grep E10

# 连接指定进程
/data/weaver/jdk/bin/java -jar /data/weaver/e-monitor/app/arthas/arthas-boot.jar 1111176
```

### 15.2 先看类和源码

```bash
# 搜索 BeiSen 相关类
sc *BeiSen*

# 查看目标类详情
sc -d com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApiUtil

# 查看该类的方法
sm com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApiUtil

# 反编译运行中代码
jad com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApiUtil
```

### 15.3 排查接口入参与返回值

```bash
# 看 getAccessToken 的入参、返回值、异常
watch com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApiUtil getAccessToken '{params,returnObj,throwExp}' -n 5 -x 2

# 如果想只看进入方法时的参数
watch com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApiUtil getAccessToken '{params}' -b -n 5 -x 2

# 如果想只看异常
watch com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApiUtil getAccessToken '{params,throwExp}' -e -n 5 -x 2
```

### 15.4 不知道是谁调了它

```bash
# 看 getAccessToken 的调用来源
stack com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApiUtil getAccessToken
```

### 15.5 怀疑这个方法慢

```bash
# 看内部调用耗时
trace com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApiUtil getAccessToken

# 只看耗时超过 200ms 的慢调用
trace com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApiUtil getAccessToken '#cost > 200'
```

### 15.6 偶发问题抓现场

```bash
# 记录 5 次调用现场
tt -t com.weaver.intcenter.hr.dataInterface.source.impl.beisen.util.BeiSenApiUtil getAccessToken -n 5

# 列出记录
tt -l

# 查看某次调用详情
tt -i 1000 -w '{params,returnObj,throwExp}' -x 2
```

## 16. 生产环境使用建议

这部分非常重要，很多 Arthas 问题不是命令不会写，而是范围没收住。

### 16.1 控制输出量

```bash
# 好习惯：限制次数
watch com.example.UserService createUser '{params,returnObj}' -n 3 -x 2

# 好习惯：按耗时过滤
trace com.example.UserService createUser '#cost > 100'
```

建议：

1. 优先加 `-n`。
2. 优先加条件表达式。
3. 对大对象不要盲目提高 `-x` 层级。

### 16.2 优先精确匹配

如果类名、方法名写得过于宽泛，可能匹配太多类或方法，影响排查效率。

建议先：

1. 用 `sc` 精确确认类名。
2. 用 `sm` 确认方法名。
3. 再执行 `watch`、`trace`、`tt`。

如果你怀疑命中了子类而不是当前类，可以考虑关闭“默认匹配子类”的行为：

```bash
# 关闭子类匹配，只匹配当前类本身
options disable-sub-class true
```

命令说明：

```bash
# disable-sub-class = 禁止子类匹配
# 默认很多命令会匹配当前类及其子类
# 关闭后更适合精确定位单个类
```

### 16.3 注意类加载器问题

同一个类名可能被不同类加载器加载。出现“明明类存在，但命令没打到目标方法”的情况时，优先怀疑类加载器。

排查思路：

```bash
# 查看类详细信息，关注 classLoaderHash
sc -d com.example.UserService
```

### 16.4 命令没输出时先别慌

常见原因：

1. 类名写错。
2. 方法名写错。
3. 方法根本没被调用到。
4. 匹配到了错误的类加载器。
5. 条件表达式太严格。

先回退到最简单命令验证：

```bash
sc *UserService*
sm com.example.UserService
watch com.example.UserService createUser '{params}' -n 1
```

## 17. 一组建议背熟的 Arthas 命令

如果你想尽快达到“熟练使用”的水平，建议优先把下面这些命令练熟：

```bash
sc *关键字*
sc -d 全限定类名
sm 全限定类名
jad 全限定类名

watch 全限定类名 方法名 '{params}' -b -n 3 -x 2
watch 全限定类名 方法名 '{params,returnObj}' -s -n 3 -x 2
watch 全限定类名 方法名 '{params,throwExp}' -e -n 3 -x 2

stack 全限定类名 方法名
trace 全限定类名 方法名
trace 全限定类名 方法名 '#cost > 100'
monitor -c 5 全限定类名 方法名

tt -t 全限定类名 方法名 -n 5
tt -l
tt -i 1000 -w '{params,returnObj,throwExp}' -x 2

dashboard
thread -n 3
thread -b
```

## 18. 官方文档入口

如果要继续深入，建议直接看官方文档对应章节：

1. `https://arthas.aliyun.com/doc/`
2. `https://arthas.aliyun.com/doc/sc.html`
3. `https://arthas.aliyun.com/doc/sm.html`
4. `https://arthas.aliyun.com/doc/jad.html`
5. `https://arthas.aliyun.com/doc/watch.html`
6. `https://arthas.aliyun.com/doc/stack.html`
7. `https://arthas.aliyun.com/doc/trace.html`
8. `https://arthas.aliyun.com/doc/monitor.html`
9. `https://arthas.aliyun.com/doc/tt.html`
10. `https://arthas.aliyun.com/doc/thread.html`
11. `https://arthas.aliyun.com/doc/vmtool.html`

## 19. 最后给你的实战建议

想真正熟练，不是背所有命令，而是把下面这个套路练成肌肉记忆：

1. `sc` 找类。
2. `sm` 看方法。
3. `jad` 看真实代码。
4. `watch` 看入参、出参、异常。
5. `stack` 看调用来源。
6. `trace` 看慢在哪。
7. `tt` 抓偶发现场。
8. `thread`、`dashboard` 看进程级问题。

如果你能把上面 8 步熟练串起来，Arthas 在大多数 Java 接口排查场景里就已经够用了。
