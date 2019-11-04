---
title: NFV已死-云杀死的
date: 2019-11-04 11:23:48
tags:
 - NFV
keywords:
 - NFV
 - SASE
categories:
 - NFV
description:
 - NFV已死
summary_img:
 - https://raw.githubusercontent.com/louielong/blogPic/master/img20191104112635.png
---

本文翻译自[Gartner: NFV Is Dead– the Cloud Killed It](https://www.lightreading.com/cloud/gartner-nfv-is-dead-andndash-the-cloud-killed-it-/d/d-id/754790?utm_source=linkedin&utm_medium=social_organic&utm_content=2f9905b7-b5d4-4bf4-bf4d-b7e846075d39&utm_campaign=GFMC_global_Big_Ideas_Blog_20180101)，可能存在翻译不准确的地方。

Enterprises are demanding a new generation of cloud-based wide-area networking services that's swallowing up SD-WAN, killing network functions virtualization (NFV) and challenging existing telco business and technology models, according to Gartner analysts. 

根据Gartner的分析师，企业需要新一代的基于云的广域网服务，这些服务将吞噬SD-WAN，杀死网络功能虚拟化（NFV）并挑战现有的电信业务和技术模型。

Gartner has given the new network delivery business model a name, and it's an ugly one: SASE, pronounced "sassy," which stands for the "Secure Access Service Edge." And if Gartner is right, the effect on service providers' business is going to be ugly too. 

Gartner给新的网络交付业务模型起了一个名字，一个非常丑陋的名字：SASE，发音为“ sassy”，代表“安全访问服务边缘”(Secure Access Service Edge)。 如果Gartner是正确的，那么对服务提供商业务的影响也将是丑陋的。

The SASE transformation has been building for years. Five years ago, almost all enterprise applications and data lived in the data center, Gartner analyst Joe Skorupa tells Light Reading. Branch office networking connected to the data center, as did remote workers. Whatever cloud access was necessary then went to the data center first, then out to the public Internet. 

SASE转型已经进行了多年。 Gartner分析师Joe Skorupa告诉Light Reading，五年前，几乎所有企业应用程序和数据都存在于数据中心。 分支机构网络连接到数据中心，远程工作者也是如此。 无论需要什么云访问权限，然后都首先访问数据中心，然后再访问公共Internet。

"Now, applications are pretty much everywhere," Skorupa says. Some are in the data center, some are outside of it. Mission-critical applications live in the cloud, including Workday, Microsoft Office 365, and custom applications written for Microsoft Azure and Amazon Web Services. "The data center is no longer the center of the universe," he says. 

“现在，到处都有应用程序，” Skorupa说。 有些在数据中心内，有些在数据中心外。 关键任务应用程序存在于云中，包括Workday，Microsoft Office 365和为Microsoft Azure和Amazon Web Services编写的自定义应用程序。 他说：“数据中心不再是宇宙的中心。”

Skorupa adds, "We have gone from having a 'data center' to having 'centers of data,' and they are all over the place."

Skorupa补充说：“我们已经从拥有“数据中心”变为拥有“中心的数据”，并且它们遍布各处。”

Likewise, consumers of data aren't just branch offices. Endpoints are mobile. "They're a sales executive sitting in a car with a cup of coffee and an iPad," Skorupa says. "They're not funneling through the data center. It's a hub and spoke. But the hub is the individual, which could be a person, could be an IoT device, and could be software."

同样，数据的使用者不仅仅是分支机构。 端点是移动的。 Skorupa说：“他们是一名销售主管，坐在汽车上喝着咖啡和iPad。” “他们并没有通过数据中心进行传输。它是一个枢纽。但是枢纽是个体，可以是人，可以是IoT设备，也可以是软件。”

The new network architecture requires different technologies to suit different needs, Skorupa says. For example, a home worker doesn't need SD-WAN because they're not balancing multiple links, but that worker does need quality-of-service guarantees to make video calls. On the other hand, a branch office requires SD-WAN for security and path selection. 

Skorupa说，新的网络架构需要不同的技术来满足不同的需求。 例如，家庭工作者不需要SD-WAN，因为他们没有多个链接需要平衡，但是该工作者确实需要服务质量保证才能进行视频通话。 另一方面，分支机构需要SD-WAN进行安全性和路径选择。

The changing nature of business requires changing security policies and technology as well, Skorupa says. "If it's a contractor using an untrusted laptop logging in from Southeast Asia at two o'clock Sunday morning directly into Salesforce, trying to get at the entire client database, you want to apply a lot of security policy against that," Skorupa says. 

Skorupa说，业务性质的变化也需要安全策略和技术的变化。 Skorupa说：“如果承包商使用的是不受信任的笔记本电脑，则需要在周日凌晨两点从东南亚直接登录到Salesforce，以获取整个客户数据库，那么您将对此应用大量的安全策略。”

Additionally, enterprise locations need intrusion detection and prevention services (IDS/IPS), data loss prevention (DLP), anti-spam, anti-malware, whitelisting, blacklisting and so on. "The overhead of trying to keep that stuff patched is a nightmare. You're always out of date. You're not going to put seven boxes stacked up -- and duct them to the back of my iPad when I'm traveling," Skorupa says. Cloud delivery is the only model that makes sense. 

此外，企业场所需要入侵检测和预防服务（IDS / IPS），数据丢失预防（DLP），反垃圾邮件，反恶意软件，白名单，黑名单等。 “试图修补这些东西的开销是一场噩梦。您总是错过最佳防护时机。您不会愿意在旅行时将他们七个盒子叠在一起并放在iPad的背面， ” Skorupa说。 云交付是唯一有意义的模型。

He adds, "The only way to apply policy anywhere and everywhere, scaling up and scaling down as needed, delivering a set of functions you need on demand, is to deliver it primarily cloud-based."

他补充说：“在任何地方和任何地方应用策略，按需扩展和按比例缩小，提供按需提供的功能集的唯一方法是基于云来提供它。”

**R.I.P. CPE**
 That means on-premises equipment needs to go from being the standard way of delivering enterprise services to a specialized case, says the Gartner man. 

Gartner负责人说，这意味着本地设备需要从提供企业服务的标准方式转变为特殊情况。

"The model says on-prem only when you must, cloud-delivered whenever you can," Skorupa says.

Skorupa说：“该模型仅在必要时才在企业内部显示，并在可能的情况下通过云交付。”

This "represents an existential threat to NFV" because NFV depends on selling expensive boxes that happen to be x86-based. The cost benefits promised initially for NFV failed to materialize because vendors simply refused to lower their prices by a lot, Skorupa says.

这“代表对NFV的生存威胁”，因为NFV依赖出售恰好基于x86的昂贵盒子。 Skorupa说，最初承诺为NFV提供的成本收益未能实现，因为供应商只是拒绝大幅降低价格。 

NFV proved "incredibly complicated," and while the telco industry struggled to make it work, "application consumption patterns changed and the branch was no longer the center of the universe, and a solution that was non-scalable and hard to maintain and expensive and complex winds up being obsoleted by something that is elastic and easy to maintain and it's cloud delivered," Skorupa says. 

事实证明NFV“异常复杂”，而电信业正努力使之运转，“应用程序消费模式发生了变化，分支机构不再是整个领域的中心， 一个不可扩展、难以维护且价格昂贵的解决方案，将被弹性、易于维护的东西淘汰了，并且使用了云交付，” Skorupa说。

There are cases where NFV makes sense. "But by and large the days of NFV have already come and gone. It's basically stillborn," Skorupa says. 

在某些情况下，NFV是有意义的。 “但是总的来说，NFV的时代已经过去了。它基本上已经死了，” Skorupa说。

In a July note, Gartner recommends several steps for technology and service providers to succeed in the new market. They need to transform offerings to a cloud-native architecture, transform business models to "cloud-native-as-a service," deliver "a clear vision" to the market, fill out their "portfolio organically, with the fewest acquisitions possible to minimize integration challenges and inconsistencies across services," and invest in distributed real estate, such as PoPs and colocation facilities, to place service as close to the access point as required. 

在7月的一份报告中，Gartner建议技术和服务提供商在新市场上取得成功的几个步骤。 他们需要将产品转变为云原生架构，将业务模型转变为“云原生即服务”，向市场提供“清晰的愿景”，通过最少的收购来有机地组合产品组合，以最大程度地减少集成挑战和服务之间的不一致，并投资于分布式资产（例如PoP和托管设施），以根据需要将服务放置在靠近接入点的位置。

Gartner names several vendors as already network-security focused, including Cato Networks, Fortinet, Forcepoint, Juniper and Versa Networks. Other SD-WAN vendors without cloud-delivered security are partnering with Zscaler, Palo Alto Networks and others. 

Gartner列出了多家已经关注网络安全的供应商，包括Cato Networks，Fortinet，Forcepoint，Juniper和Versa Networks。 其他没有云交付安全性的SD-WAN供应商正在与Zscaler，Palo Alto Networks等合作。

Of course, the industry being what it is, these vendors are going into paroxysms of joy by merely being mentioned by Gartner. Versa and Cato Networks put out press releases and statements on their websites, and zScaler devoted some discussion to the subject on an earnings call.

当然，无论是什么行业，这些供应商都只是被Gartner提及而陷入欢乐的阵营。 Versa和Cato Networks在其网站上发布了新闻稿和声明，而zScaler在财报电话会议上对该主题进行了一些讨论。

**Telcos behind the eight-ball**

**电信公司陷入困境**
 Cato Networks, for one, sees the shift to SASE as a competitive advantage. "Telcos are behind the eight-ball," Yishah Yovel, Cato CMO and chief strategist, tells Light Reading. Telco networks are based on appliances, and they're two years behind catching up on the cloud networking model. 

例如，Cato Networks将向SASE的转变视为一种竞争优势。 Cato首席营销官兼首席策略师Yishah Yovel对Light Reading表示：“电信公司陷入困境。” 电信网络是基于设备的，比追赶云网络模型落后了两年。

Telcos are disadvantaged because they don't own the code. "If I'm a Palo Alto or Zscaler, I have my own code. I already have some percentage of the SASE platform. Telcos don't operate this way. They integrate other people's code. That's very dangerous for them, unless they become more of a software player."

电信公司处于不利地位，因为它们不拥有代码。 “如果我是Palo Alto或Zscaler，我有我自己的代码。我已经拥有一定比例的SASE平台。电信公司无法以这种方式运行。它们会集成其他人的代码。这对他们来说非常危险，除非他们变得 更多的软件播放器。”

Looked at one way, Gartner's SASE pitch is nothing new. Indeed, when a Cato Networks spokesman brought it to my attention a few weeks ago, I initially scoffed.

从一种角度看，Gartner的SASE呼声渐涨并不是什么新鲜事物。 确实，几周前，当Cato Networks的一位发言人提请我注意时，我开始嘲笑它。 

Normally I would have been more polite, but I was in a bad mood on account of being still jet-lagged and sleep deprived from a trip to Dallas, to Light Reading's Network Virtualization & Software Defined Networking conference, which was all about the trends Gartner had apparently just discovered. And it wasn't the first year we've done that conference; far from it. So my first reaction to the tip was, "[Thank you, Captain Obvious!](https://knowyourmeme.com/memes/captain-obvious)"


通常，我本来会比较有礼貌，但是由于刚到达拉斯的时差，无法参加Light Reading的网络虚拟化和软件定义网络会议，所以我心情很不好，这也与Gartner刚刚发现的趋势有关。 这不是我们召开会议的第一年。所以我对小费的第一反应是：“谢谢，上尉船长！”

The software-defined networking (SDN) movement, launched at the beginning of the decade, was all about moving network intelligence into software for increased agility; the reason we don't hear much about that anymore is because the philosophy has become mainstream.

十年之初发起的软件定义网络（SDN）运动就是将网络智能转移到软件中以提高敏捷性。 我们之所以不再听到太多的原因是因为该哲学已成为主流。

More recently, AT&T, Orange and startup Rakuten are aggressively moving their networks to cloud architectures. Just last week, [Colt launched](https://www.lightreading.com/nfv/nfv-elements/colt-brings-virtual-networks-to-enterprise-premises/d/d-id/754763) a new line of universal CPE (uCPE) equipment, providing SD-WAN, firewall and other services to enterprises, based on NFV.

最近，AT＆T，Orange和创业公司Rakuten正在积极地将其网络迁移到云架构。 就在上周，Colt推出了新的通用CPE（uCPE）设备系列，为基于NFV的企业提供SD-WAN，防火墙和其他服务。

Still, NFV has attracted skeptics almost since its founding in 2012, and at about the same time Gartner issued its SASE note, we [reported](https://www.lightreading.com/nfv/nfv-strategies/smokey-and-the-nfv-bandit/a/d-id/753939) that critics were saying the technology is too rigid and monolithic for the cloud era, (though Prayson Pate, CTO, Edge Cloud, ADVA Optical Networking [took issue with our report](https://www.lightreading.com/nfv/nfv-strategies/smokey-and-the-nfv-bandit/a/d-id/753939)). 

尽管如此，NFV几乎自2012年成立以来就一直引起怀疑论者的关注，并且大约在同一时间，Gartner发布了SASE说明，据我们报道，批评者称该技术对于云时代而言过于僵化和整体化（尽管 Prayson Pate (ADVA光网络公司Edge Cloud 的CTO)对我们的报告对提出了质疑）。

However, my initial dismissal was misplaced. Gartner does a good job of weaving together and articulating several long-term trends shaping the service provider business and networks. Gartner deserves credit for stepping back and summarizing a decade of trends in a few pages. 

但是，我最初的解雇是错误的。 Gartner很好地组织并阐明了影响服务提供商业务和网络的几种长期趋势。 Gartner值得一提的是在几页中后退并总结了十年的趋势。

Also, Gartner is influential, particularly among enterprises who are service provider customers. Gartner's SASE coinage means ideas about wide-area network virtualization and cloudification have gone mainstream. Telcos are going to start hearing demand for SASE, and need to be prepared to meet it. 

此外，Gartner很有影响力，特别是在服务提供商客户企业中。 Gartner的SASE概念意味着有关广域网虚拟化和云化的想法已成为主流。 电信公司将开始听到对SASE的需求，并且需要做好准备以满足它。

For more about how AT&T, Orange, Rakuten and other service providers already cloudifying and virtualizing their networks, see these articles:

- [Colt Brings Virtual Networks to      Enterprise Premises](https://www.lightreading.com/nfv/nfv-elements/colt-brings-virtual-networks-to-enterprise-premises/d/d-id/754763)
- [AT&T's Wheelus: From Mechanization      to Automation](https://www.lightreading.com/automation/atandts-wheelus-from-mechanization-to-automation/d/d-id/754688)
- [Alaska's Rakuten Preps for 4G Launch](https://www.lightreading.com/mobile/4g-lte/alaskas-rakuten-preps-for-4g-launch/d/d-id/754538)
- [Why AT&T's Latest Open Source      Contribution Matters](https://www.lightreading.com/optical-ip/why-atandts-latest-open-source-contribution-matters/d/d-id/754484)
- [Common NFVI Telco Taskforce (CNTT)      Boasts Reference Milestone](https://www.lightreading.com/nfv/nfv-specs-open-source/common-nfvi-telco-taskforce-(cntt)-boasts-reference-milestone/d/d-id/754308)
- [AT&T on Track for 100% Core      Network Virtualization Next Year](https://www.lightreading.com/carrier-sdn/sdn-technology/atandt-on-track-for-100--core-network-virtualization-next-year/d/d-id/754104)
- [Rakuten Delays Launch of Cloud-Native      Network ](https://www.lightreading.com/mobile/5g/rakuten-delays-launch-of-cloud-native-network-/d/d-id/753940)
- [It's Groundhog Day! Explaining      AT&T's Microsoft & IBM Deals](https://www.lightreading.com/cloud/applications/its-groundhog-day!-explaining-atandts-microsoft-and-ibm-deals/a/d-id/753001)
- [AT&T & Microsoft Ink      'Extensive' Deal for Cloud, 5G, AI & Edge](https://www.lightreading.com/cloud/iot-and-edge/atandt-and-microsoft-ink-extensive-deal-for-cloud-5g-ai-and-edge/d/d-id/752852)
- [Is X by Orange Showing Us the OTT      Future for Telcos?](https://www.lightreading.com/nfv/nfv-strategies/is-x-by-orange-showing-us-the-ott-future-for-telcos/d/d-id/746400)