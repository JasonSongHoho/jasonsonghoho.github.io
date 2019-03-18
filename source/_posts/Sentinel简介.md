---
title: Sentinel 使用简介
date: 2019-03-18 16:04:27
tags:
- Java
- 中间件

---

{% asset_img  sentinel-logo.png %}

### Sentinel 是什么
Sentinel 是面向分布式服务架构的轻量级 **流量控制框架**，可以从 **流量控制、熔断降级、系统负载保护** 等多个维度来保护服务的稳定性。

#### 流量控制
可以理解为限制某种请求的最大 QPS 或它能同时发起的最大线程数。控制行为可以是 **直接拒绝、排队等候、缓慢启动** 等。

#### 熔断降级
当某种请求的连续的几次请求都超时或表现出其他异常，影响到服务的健康时，可以在接下来的一段时间内拒绝相同的请求，以保护服务健康。

#### 系统负载保护
从整体维度对应用入口流量进行控制，结合应用的 Load、总体平均 RT(RequestTime)、入口 QPS 和线程数等几个维度的监控指标，**让系统的入口流量和系统的负载达到一个平衡**，让系统尽可能跑在最大吞吐量的同时保证系统整体的稳定性。

{% asset_img  sentinel功能.png %}

### Sentinel 基本概念
#### 资源
资源是 Sentinel 的关键概念。 **它可以是 Java 应用程序中的任何内容**，例如，由应用程序提供的服务，或由应用程序调用的其它应用提供的服务，甚至可以是一段代码。

#### 规则
围绕资源的实时状态设定的规则，可以包括流量控制规则、熔断降级规则以及系统保护规则。所有规则可以 **动态实时调整** 。

### Quick Start
#### 1.添加 POM 依赖
Sentinel 的使用可以分为两个部分:
- 核心库（Java 客户端）：核心功能实现包。不依赖任何框架/库，能够运行于 Java 7 及以上的版本的运行时环境。
- 控制台（Dashboard）：Dashboard主要负责管理推送规则；监控；管理机器信息等。

```
<!-- 核心库 -->
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-core</artifactId>
    <version>1.5.0</version>
</dependency>
<!-- 与控制台交互用 -->
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-transport-simple-http</artifactId>
    <version>1.5.0</version>
</dependency>

```

#### 2.定义资源
```
public static void main(String[] args) {
    initFlowRules();
    while (true) {
        Entry entry = null;
        try {
	    entry = SphU.entry("HelloWorld");
            /*您的业务逻辑 - 开始*/
            System.out.println("hello world");
            /*您的业务逻辑 - 结束*/
	} catch (BlockException e1) {
            /*流控逻辑处理 - 开始*/
	    System.out.println("block!");
            /*流控逻辑处理 - 结束*/
	} finally {
	   if (entry != null) {
	       entry.exit();
	   }
	}
    }
}
```
#### 3.定义规则
```
private static void initFlowRules(){
    List<FlowRule> rules = new ArrayList<>();
    FlowRule rule = new FlowRule();
    rule.setResource("HelloWorld");
    rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
    // Set limit QPS to 20.
    rule.setCount(20);
    rules.add(rule);
    FlowRuleManager.loadRules(rules);
}
```
启动该 demo 工程，参数添加 `-Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=data-gateway`

#### 4.启动控制台观察结果

下载 [alibaba/Sentinel](git@github.com:alibaba/Sentinel.git) 工程，本地启动 sentinel-dashboard 模块，运行参数添加 `-Dserver.port=8080` 。

{% asset_img  sentinel控制台.png sentinel控制台  %}


更多 demo 可参考 [Sentinel Examples](https://github.com/alibaba/Sentinel/tree/master/sentinel-demo)


### 规则详情
#### 流量控制规则 (FlowRule)
##### 重要属性：

| Field | 说明 | 默认值 |
|--------|--------|--------|
|resource|资源名，资源名是限流规则的作用对象	||
|count|限流阈值|QPS 模式|
|grade|限流阈值类型，QPS 或线程数模式||
|limitApp|流控针对的调用来源|default，代表不区分调用来源|
|strategy|判断的根据是资源自身，还是根据其它关联资源 (refResource)，还是根据链路入口|根据资源本身|
|controlBehavior|流控效果（直接拒绝 / 排队等待 / 慢启动模式）|直接拒绝|

##### demo:
```
private void initFlowQpsRule() {
    List<FlowRule> rules = new ArrayList<>();
    FlowRule rule = new FlowRule(resourceName);
    // set limit qps to 20
    rule.setCount(20);
    rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
    rule.setLimitApp("default");
    rules.add(rule);
    FlowRuleManager.loadRules(rules);
}
```
更多内容可参考[流量控制](https://github.com/alibaba/Sentinel/wiki/%E6%B5%81%E9%87%8F%E6%8E%A7%E5%88%B6)


#### 熔断降级规则 (DegradeRule)
##### 重要属性：

| Field | 说明 | 默认值 |
|--------|--------|--------|
|resource|资源名，资源名是限流规则的作用对象	||
|count|阈值,单位为 ms||
|timeWindow|降级的时间，单位为 s| |
|grade|降级模式，根据 RT 降级还是根据异常比例降级|RT|

##### demo:
```
private void initDegradeRule() {
    List<DegradeRule> rules = new ArrayList<>();
    DegradeRule rule = new DegradeRule();
    rule.setResource(KEY);
    rule.setGrade(RuleConstant.DEGRADE_GRADE_RT);
    // 若连续 5 个请求的响应时间均超过 300 ms ，将会被熔断 10S
    rule.setCount(300);
    rule.setTimeWindow(10);
    rules.add(rule);
    DegradeRuleManager.loadRules(rules);
}
```
更多内容可参考[熔断降级](https://github.com/alibaba/Sentinel/wiki/%E7%86%94%E6%96%AD%E9%99%8D%E7%BA%A7)


#### 系统保护规则 (SystemRule)
##### 重要属性：

| Field | 说明 | 默认值 |
|--------|--------|--------|
|highestSystemLoad|最大的 load1，参考值|-1 (不生效)|
|avgRt|所有入口流量的平均响应时间|-1 (不生效)|
|maxThread|入口流量的最大并发数	|-1 (不生效)|
|qps|所有入口资源的 QPS|-1 (不生效)|


##### demo:
```
private void initSystemRule() {
    List<SystemRule> rules = new ArrayList<>();
    SystemRule rule = new SystemRule();
    rule.setHighestSystemLoad(10);
    rules.add(rule);
    SystemRuleManager.loadRules(rules);
}
```
更多内容可参考[系统自适应限流](https://github.com/alibaba/Sentinel/wiki/%E7%B3%BB%E7%BB%9F%E8%87%AA%E9%80%82%E5%BA%94%E9%99%90%E6%B5%81)


#### 授权规则 (AuthorityRule)
##### 重要属性：

| Field | 说明 | 默认值 |
|--------|--------|--------|
|resource|资源名，资源名是限流规则的作用对象	||
|limitApp|对应的黑名单/白名单，不同 origin 用 , 分隔，如 appA,appB||
|strategy|限制模式，AUTHORITY_WHITE 为白名单模式，AUTHORITY_BLACK 为黑名单模式|白名单|

##### demo:
```
AuthorityRule rule = new AuthorityRule();
rule.setResource("test");
rule.setStrategy(RuleConstant.AUTHORITY_WHITE);
rule.setLimitApp("appA,appB");
AuthorityRuleManager.loadRules(Collections.singletonList(rule));
```
更多内容可参考[黑白名单控制](https://github.com/alibaba/Sentinel/wiki/%E9%BB%91%E7%99%BD%E5%90%8D%E5%8D%95%E6%8E%A7%E5%88%B6)

#### 热点规则 (ParamFlowRule)
##### 重要属性：

| Field | 说明 | 默认值 |
|--------|--------|--------|
|resource|资源名，必填||
|count|限流阈值，必填||
|paramIdx|热点参数的索引，必填，对应 SphU.entry(xxx, args) 中的参数索引位置||
|paramFlowItemList|参数例外项，可以针对指定的参数值单独设置限流阈值，不受前面 count 阈值的限制。仅支持基本类型| |
|grade|限流模式|QPS 模式|

##### demo:
```
ParamFlowRule rule = new ParamFlowRule(resourceName)
    .setParamIdx(0)
    .setCount(5);
// 针对 int 类型的参数 PARAM_B，单独设置限流 QPS 阈值为 10，而不是全局的阈值 5.
ParamFlowItem item = new ParamFlowItem().setObject(String.valueOf(PARAM_B))
    .setClassType(int.class.getName())
    .setCount(10);
rule.setParamFlowItemList(Collections.singletonList(item));

ParamFlowRuleManager.loadRules(Collections.singletonList(rule));
```
更多内容可参考[热点参数限流](https://github.com/alibaba/Sentinel/wiki/%E7%83%AD%E7%82%B9%E5%8F%82%E6%95%B0%E9%99%90%E6%B5%81)




