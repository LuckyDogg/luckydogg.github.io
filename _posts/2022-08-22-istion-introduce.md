---
layout: post
title: "Istio Introduce"
subtitle: "Istio 学习之路"
date: 2022-08-22
author: "Chaos"
header-img: "img/genshin-bg.jpeg"
tags: 
    - istio
    - kubernetes
---


## **Kubernetes vs Service Mesh**

![Kubernetes vs Service Mesh](/img/Kubernetes_vs_Service_Mesh.png)

## 流量转发

K8s集群中的每个节点都部署了一个kube-proxy组件，该组件与K8s API Server 进行通信，获取集群中的服务信息，然后设置iptables规则，将服务请求直接发送到对应的Endpoint(属于同一组服务的pod)

## 服务发现

<!-- ![Untitled](%E6%9C%8D%E5%8A%A1%E7%BD%91%E6%A0%BC%E7%AE%80%E4%BB%8B%205a4223b86fb4477fad3252b007498f57/Untitled%201.png) -->

istio可以跟踪K8s中的服务注册，也可以在控制平面中通过平台适配器与其他服务发现系统对接；然后生产数据平面的配置（使用CRD，这些配置存储在etcd中），数据平面的透明代理。数据平面的透明代理以sidecar容器的形式部署在每个应用服务的pod中，这些代理都需要请求控制平面同步代理配置。代理之所以 “透明”，是因为应用容器完全不知道代理的存在。过程中的 kube-proxy 组件也需要拦截流量，只不过 kube-proxy 拦截的是进出 Kubernetes 节点的流量，而 sidecar 代理拦截的是进出 pod 的流量。

## **服务网格的劣势**

由于 Kubernetes 的每个节点上都运行着很多 pod，所以在每个 pod 中放入原有的 kube-proxy 路由转发功能，会增加响应延迟——由于 sidecar 拦截流量时跳数更多，消耗更多的资源。为了对流量进行精细化管理，将增加一系列新的抽象功能。这将进一步增加用户的学习成本，但随着技术的普及，这种情况会慢慢得到缓解。

## **服务网格的优势**

kube-proxy 的设置是全局的，无法对每个服务进行细粒度的控制，而 service mesh 通过 sidecar proxy 的方式将 Kubernetes 中的流量控制从服务层中抽离出来–可以实现更大的弹性。

## **Envoy**

Envoy 是 Istio 中默认的 sidecar 代理。Istio 基于 Enovy 的 xDS 协议扩展了其控制平面。在讨论 Envoy 的 xDS 协议之前，我们需要先熟悉 Envoy 的基本术语。

<!-- ![Untitled](%E6%9C%8D%E5%8A%A1%E7%BD%91%E6%A0%BC%E7%AE%80%E4%BB%8B%205a4223b86fb4477fad3252b007498f57/Untitled%202.png) -->

### **基本术语**

下面是您应该了解的 Enovy 里的基本术语：

- **Downstream（下游）**：下游主机连接到 Envoy，发送请求并接收响应，即发送请求的主机。
- **Upstream（上游）**：上游主机接收来自 Envoy 的连接和请求，并返回响应，即接受请求的主机。
- **Listener（监听器）**：监听器是命名网地址（例如，端口、unix domain socket 等)，下游客户端可以连接这些监听器。Envoy 暴露一个或者多个监听器给下游主机连接。
- **Cluster（集群）**：集群是指 Envoy 连接的一组逻辑相同的上游主机。Envoy 通过[服务发现](http://www.servicemesher.com/envoy/intro/arch_overview/service_discovery.html#arch-overview-service-discovery)来发现集群的成员。可以选择通过[主动健康检查](http://www.servicemesher.com/envoy/intro/arch_overview/health_checking.html#arch-overview-health-checking)来确定集群成员的健康状态。Envoy 通过[负载均衡策略](http://www.servicemesher.com/envoy/intro/arch_overview/load_balancing.html#arch-overview-load-balancing)决定将请求路由到集群的哪个成员。

## **Istio 中的流量管理**

Istio 中定义了以下 CRD 来帮助用户进行流量管理。

- 网关。网关描述了一个运行在网络边缘的负载均衡器，用于接收传入或传出的 HTTP/TCP 连接。
- 虚拟服务（VirtualService）。VirtualService 实际上是将 Kubernetes 服务连接到 Istio 网关。它还可以执行额外的操作，例如定义一组流量路由规则，以便在主机寻址时应用。
- DestinationRule。DestinationRule 定义的策略决定了流量被路由后的访问策略。简单来说，它定义了流量的路由方式。其中，这些策略可以定义为负载均衡配置、连接池大小和外部检测（用于识别和驱逐负载均衡池中不健康的主机）配置。
- EnvoyFilter。EnvoyFilter 对象描述了代理服务的过滤器，可以自定义 Istio Pilot 生成的代理配置。这种配置一般很少被主用户使用。
- ServiceEntry。默认情况下，Istio 服务 Mesh 中的服务无法发现 Mesh 之外的服务。ServiceEntry 可以在 Istio 内部的服务注册表中添加额外的条目，从而允许 Mesh 中自动发现的服务访问并路由到这些手动添加的服务。

## **核心观点**

- Kubernetes 的本质是应用生命周期管理，具体来说就是部署和管理（伸缩、自动恢复、发布）。
- Kubernetes 为微服务提供了一个可扩展、高弹性的部署和管理平台。
- 服务网格是基于透明代理，通过 sidecar 代理拦截服务之间的流量，然后通过控制平面配置管理它们的行为。
- 服务网格将流量管理与 Kubernetes 解耦，不需要 kube-proxy 组件来支持服务网格内的流量；通过提供更接近微服务应用层的抽象来管理服务间的流量、安全性和可观察性。
- xDS 是服务网格的协议标准之一。
- 服务网格是 Kubernetes 中服务的一个更高层次的抽象。

# 服务网络架构

<!-- ![Untitled](/img/Untitled_03.png) -->

服务网格中分为**控制平面**和**数据平面**，当前流行的两款开源的服务网格 Istio 和 Linkerd 实际上都是这种构造，只不过 Istio 的划分更清晰，而且部署更零散，很多组件都被拆分，控制平面中包括 Mixer、Pilot、Citadel，数据平面默认是用Envoy；而 Linkerd 中只分为 Linkerd 做数据平面，namerd 作为控制平面。

## **控制平面**

控制平面的特点：

- 不直接解析数据包
- 与控制平面中的代理通信，下发策略和配置
- 负责网络行为的可视化
- 通常提供API或者命令行工具可用于配置版本化管理，便于持续集成和部署

## **数据平面**

数据平面的特点：

- 通常是按照无状态目标设计的，但实际上为了提高流量转发性能，需要缓存一些数据，因此无状态也是有争议的
- 直接处理入站和出站数据包，转发、路由、健康检查、负载均衡、认证、鉴权、产生监控数据等
- 对应用来说透明，即可以做到无感知部署

## **Istio 架构解析**

- Istio 可以在虚拟机和容器中运行
- Istio 的组成
    - Pilot：服务发现、流量管理
    - Mixer：访问控制、遥测
    - Citadel：终端用户认证、流量加密
- Service mesh 关注的方面
    - 可观察性
    - 安全性
    - 可运维性
- Istio 是可定制可扩展的，组件是可拔插的
- Istio 作为控制平面，在每个服务中注入一个 Envoy 代理以 Sidecar 形式运行来拦截所有进出服务的流量，同时对流量加以控制
- 应用程序应该关注于业务逻辑（这才能生钱），非功能性需求交给 Service Mesh
