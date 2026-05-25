---
title: 从零构建 C++ 分布式 RPC 框架：基于 Muduo 的微服务与 Pub/Sub 架构实战
date: 2026-05-23 10:10:00
tags: [C++, Muduo, RPC, 微服务, 设计模式]
categories: [底层原理与架构设计]
---

在分布式后端研发体系中，构建高可用、可扩展的通信基座是微服务架构的核心命题。本项目基于 C++17 与 `muduo` 高性能网络库，设计并实现了一套支持同步/异步 RPC 调用、服务治理（注册与发现）以及消息发布订阅（Pub/Sub）的轻量级微服务框架。

本文将系统剖析该框架的分层架构、核心自定义协议设计、服务流转机制，以及在多线程高并发场景下的工程治理与容错实践。

## 一、 系统核心架构与分层设计

框架整体采用**自底向上**的分层架构，严格遵循控制面与数据面分离的原则，底层基于 `epoll` 边缘触发（ET）事件驱动模型保障高吞吐量。

### 1. 架构三层模型

| 架构层级 | 核心组件抽象 | 模块职责与工程实现 |
| :--- | :--- | :--- |
| **抽象层 (Interface)** | `BaseMessage`, `BaseServer`, `BaseConnection`, `BaseProtocol` | 定义网络端点、连接通道与通信协议的纯虚基类，确立系统契约。 |
| **实现层 (Implementation)** | `MuduoServer`, `LVProtocol`, `Dispatcher`, `JsonMessage` | 依托 Muduo 完成网络 I/O 读写，实现基于 JSON 的序列化载体及路由分发逻辑。 |
| **业务层 (Business)** | `RpcClient`, `RegistryServer`, `TopicManager` | 暴露顶层 API，屏蔽底层网络细节，开发者通过 Stub/Proxy 模式直接发起服务调用或注册回调。 |

### 2. 设计模式的工程化应用

为满足工业级框架的可维护性与扩展性要求，系统底层解耦深度应用了以下设计模式：
* **桥接模式 (Bridge)**：将网络端点的抽象接口与具体的 Muduo I/O 实现彻底分离。该设计确保了未来可平滑替换底层网络引擎（如迁移至 ASIO 或 brpc 底层）。
* **工厂模式 (Factory)**：通过 `MessageFactory` 集中管理反序列化对象的生命周期，屏蔽复杂协议类的实例化细节。
* **模板方法模式 (Template Method)**：在 `MessageCallback` 中定义消息处理骨架，将具体的 `OnMessage` 纯虚执行逻辑下放至派生类，由 `Dispatcher` 完成多态路由。
* **观察者模式 (Observer)**：应用于 Pub/Sub 模块，Topic 节点作为 Subject，通过非阻塞回调机制向挂载的 `Subscribe` 订阅者列表广播数据变更事件。

---

## 二、 协议定制与消息序列化

针对基于 TCP 字节流传输固有的粘包与半包问题，框架自研了定长头部协议（LV, Length-Value 的工业变体），并在协议头中引入消息路由与生命周期标识。

### 1. 物理报文结构

网络传输中的单次完整 TCP 请求/响应报文被严格定义为以下二进制结构：

| 字段 | 长度 (Bytes) | 描述 (说明) |
| :--- | :--- | :--- |
| **Total_Length** | 4 | 报文总长度（解决粘包/半包边界划分）。 |
| **Mtype** | 4 | 报文类型枚举（区分 RPC、服务注册、Topic 动作）。 |
| **ID_Length** | 4 | UUID 字符串的长度。 |
| **UUID** | 32 | 16 进制字符串（8位随机数+8字节自增序号），确保分布式全链路 Request/Response 精准映射。 |
| **Body** | 变长 | 序列化后的 JSON 业务负载（Value）。 |

### 2. 消息序列化载体

基于基类 `BaseMessage`，针对不同业务场景派生出不同的 JSON 结构体：
* **RpcRequestMessage**：承载 `_method` (目标函数签名) 与 `_params` (入参 JSON)。
* **ServiceRequestMessage**：承载服务治理动作宏 `_servicemtype`，及向 Registry 上报的 Provider 实例元数据 `_host` (IP:Port)。
* **TopicRequestMessage**：承载 Pub/Sub 原语操作宏 `_topicmtype` 及广播负载 `_message`。

---

## 三、 核心服务模块流转机制

### 1. RPC 远程过程调用 (Remote Procedure Call)

**执行链路：**
1. **调用发起**：调用端通过 `RpcClient::Call()` 发起请求，传入方法签名与回调存根（支持 Future 同步阻塞或 Callback 异步非阻塞）。`RpcCaller` 生成分布式唯一的 UUID 并封包。
2. **网络路由**：服务端 `muduo::net::Buffer` 接收字节流，交由 `LVProtocol` 进行边界校验与解包，随后透传至 `Dispatcher`。`RpcRouter` 基于哈希表字典索引，拉起对应的业务执行上下文。
3. **闭环唤醒**：服务端原路返回 `RpcResponse`。调用端网络层收到响应后，通过 `Requestor` 中的 UUID 映射表，唤醒挂起的线程或执行异步回调。

### 2. 服务注册与动态发现 (Service Governance)

采用 CP 架构理念的集中式协调中心（RegistryServer）确保服务拓扑的一致性。
* **注册与心跳**：Provider 启动时调用 `RegisterMethod` 向 Registry 写入节点元数据。Registry 维持 Provider 的长连接作为心跳依据。
* **动态发现与软负载**：Consumer 发起 RPC 前查阅本地路由表。若发生 Cache Miss，则触发 `ServiceDiscovery` 向远端拉取全量 Provider 列表。调用时采用 Round-Robin (轮询) 算法进行客户端软负载均衡。
* **故障剔除**：若 Provider 节点宕机或 TCP 异常断开，Muduo 底层触发 `OnConnection` 关闭回调，Registry 将立即清除脏数据，并向所有 Watch 该服务的 Consumer 广播下线通知。

### 3. 发布订阅机制 (Pub/Sub)

基于内存队列实现“单点发布，多点扇出”的异步解耦模型。
由 `TopicManager` 全局管控 Topic 生命周期，提供四大原语语义：`Create` (初始化主题空间), `Subscribe` (注册长连接监听), `Publish` (向队列投递负载), `Cancel` (销毁资源)。

---