---
title: NFV的一些思考
date: 2019-11-04 11:38:19
tags:
 - NFV
 - SASE
keywords:
 - NFV
categories:
 - NFV
description:
 - 最近以来的一些NFV的思考
summary_img:
 - https://raw.githubusercontent.com/louielong/blogPic/master/img1-1Z514152050319.png
---

先前看到`litght reading`的NFV已死，也尝试进行了翻译[NFV已死](nfv-is-dead.html)，从文中不断看到SASE、云原生这样的词汇。2017年初由于工作原因从IOT转向了NFV相关工作，随后接触到了Openstack、OPNFV、ONAP，这段时间的工作使自己对于网络、云计算、NFV、docker容器、k8s有了很多的认识，也为自己在超级算力中心的整体规划设计以及建设中积累了宝贵的经验。

话说回来，在从事NFV研究的一年多时间里，NFV始终是雷声大雨点小，我所在的OPNFV社区主要关注NFV中的基础设计层即NFVI层的测试，我参与了OPNFV的Pharos项目以及OVP项目的首批内测。在多次测试活动中也了解了三大运营商以及国内主流厂商如华为、中兴、华三等厂商的NFV产品。但是NFV呼声虽高，却没有看到太多的市场，运营商期望通过通用x86+软件的方式来去除厂商黑盒的绑定，但是最后还是掉落到了硬件厂商的陷阱里，期望的硬件+虚拟化+云平台的三层解耦在现实中也只是泡影，最终还是被厂商的硬件(x86)+云平台(Openstack)+网元(VNF)所绑定。运营商都会有自研的NFVO编排器，期望通过自己的编排器与厂商A的VNF和厂商B的云平台层进行解耦调度。在我们的测试中基本的网元LCM生命管理都没什么问题，但是涉及到高级的特性如：自愈、热迁移等都会出现问题[1]。同时在ONAP的设想中有一个VNF测试项目，期望将厂商自研的VNF网元做成一个类似于应用市场的APP，使得运营商可以直接在应用市场里选择需要的网元，但也面临许多的挑战。 

NFV已死的文中提到了SASE(Secure Access Service Edge，安全访问服务边缘)将会取代NFV，我找了下关于SASE的描述，在[Cato and the Secure Access Service Edge: Where Your Digital Business Network Starts](https://www.sdxcentral.com/articles/sponsored/syndicated/blog/cato-and-the-secure-access-service-edge-where-your-digital-business-network-starts/2019/10/)中提到了SASE的四个特点。

> 它提供了一个单一网络，可在任何地方连接并保护任何企业资源（物理，云和移动）。 在这种情况下，SASE Cloud具有四个主要特征：它是身份驱动的，“云原生”的，支持所有边缘的并且在全球范围内分布：
>
> ​     身份驱动。用户和资源身份，而不仅仅是IP地址，决定了网络体验和访问权限级别。服务质量，路由选择，应用风险驱动的安全控制-所有这些都由与每个网络连接关联的身份驱动。通过让公司为用户开发一套网络和安全策略，而不论设备或位置如何，该方法都可以减少运营开销。
> ​    云原生架构。 SASE体系结构利用了关键的云功能（包括弹性，适应性，自修复和自维护），提供了一个可分摊客户成本以实现最大效率的平台，可轻松适应新兴业务需求，并可在任何地方使用。
> ​    支持所有边缘。 SASE为所有公司资源（数据中心，分支机构，云资源和移动用户）创建一个网络。例如，SD-WAN设备支持物理边缘，而移动客户端和无客户端浏览器访问则可以连接用户。
> ​    全球分布。为确保全面的网络和安全功能随处可用，并向所有边缘提供最佳体验，SASE云必须在全球范围内分布。因此，Gartner指出，他们必须扩展自己的足迹，以向企业边缘提供低延迟的服务。

云原生也是这两年来提到的非常多的概念，结合云计算以及火热的docker容器、k8s编排、微服务等，服务提供商在开发应用时要更多的结合云和容器的一些特性来提供服务。边缘计算也是近来热门的话题，从18年ETSI将MEC的定义为Multi-Access Edge Computing 多接入边缘计算而不是先前大家都认为的Mobile Edge Compute移动边缘计算。这样看起来SASE确实很适应潮流的发展。

但是这并不是说NFV已经死亡，借助虚拟化技术的发展，网络功能虚拟化仍将继续发展，在云中东西向流量防护以及安全方面NFV还是有发展余地的，此外5G 的各个网元也是NFV的主场。NFV现在的冷淡也只是没有像最开始吹的那么神，什么网络功能都能虚拟化，由于性能和稳定的的不足，一些核心网络还是没有办法全上NFV，但是在边缘接入上NFV仍然能够有的放矢。我认为更多的只是NFV技术的慢慢趋于成熟，概念上不再新颖，炒的人少了而已。

Ps：话又说回来，确实NFV的市场前景越来越不乐观，因此我也么有更多的专注在NFV方面了 - -。



【1】[2018 SDN+NFV+IPv6 Fest白皮书](https://www.sdnctc.com/download/resource_download/id/6)