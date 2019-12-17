---
title: IETF介绍及RFC Draft撰写
date: 2018-12-27 17:15:34
tags:
  - IETF
  - RFC
keywords:
  - RFC
  - Internet Draft
categories:
  - IETF
description:
  - RFC Internet Draft 撰写
summary_img:
 - https://raw.githubusercontent.com/louielong/blogPic/master/imgietf-logo.e4b6ca0dd271.gif
---
## 一、前言

最近由于工作需要，要写一篇RFC draft，谨以此文记录一下过程中查到的资料以及相关工作。

在通信和计算机行业一谈到标准提到最多的就是RFC，没接触RFC之前一直不明白RFC到底是什么。RFC的英文是“Request for Comments”，即“请求建议”，当某家机构或团体开发出了一套标准或提出对某种标准的设想，想要征询外界的意见时，就会在Internet上发放一份RFC，对这一问题感兴趣的人可以阅读该RFC并提出自己的意见；绝大部分网络标准的指定都是以RFC的形式开始，经过大量的论证和修改过程，由主要的标准化组织所指定的，但在RFC中所收录的文件并不都是正在使用或为大家所公认的，也有很大一部分只在某个局部领域被使用或并没有被采用，一份RFC具体处于什么状态都在文件中作了明确的标识。

**IETF**互联网工程任务小组（**英语：Internet Engineering Task Force**）负责互联网标准的开发和推动。它的组织形式主要是大量负责特定议题的工作组，每个都有一个指定主席（或者若干副主席）。工作组再用主题组织为**领域**（area）；每个领域都有一个**领域指导**（area director，AD），大多数领域还有两个副AD；AD任命工作组主席。AD和IETF主席构成**Internet Engineering Steering Group**（IESG），负责IETF的整体运作。[1]


关于IETF的更多详细介绍可以查看《IETF之道》[2][3]，里面详细介绍了IETF的运作及相关组织架构。下面摘录一下专业术语如下：

| 术语  | 含义                         | Meaning                                             |
| ----- | ---------------------------- | --------------------------------------------------- |
| AD    | 领域负责人                   | Area Director                                       |
| BCP   | 当前最佳实践                 | Best Current Practice                               |
| BOF   | 专题讨论会                   | Birds of a Feather                                  |
| FAQ   | 常见问题                     | Frequently Asked Question(s)                        |
| FYI   | 仅供参考（RFC）              | For Your Information (RFC)                          |
| IAB   | 互联网架构委员会             | Internet Architecture Board                         |
| IAD   | IETF行政主管                 | IETF Administrative Director                        |
| IANA  | 互联网号码分配机构           | Internet Assigned Numbers Authority                 |
| IAOC  | IETF行政监督委员会           | IETF Administrative Oversight Committee             |
| IASA  | IETF行政支持活动             | IETF Administrative Support Activity                |
| ICANN | 互联网名称与数字地址分配机构 | Internet Corporation for Assigned Names and Numbers |
| I-D   | 互联网草案                   | Internet-Draft                                      |
| IESG  | 互联网工程指导组             | Internet Engineering Steering Group                 |
| IETF  | 互联网工程任务组             | Internet Engineering Task Force                     |
| IPR   | 知识产权                     | Intellectual property rights                        |
| IRTF  | 互联网研究任务组             | Internet Research Task Force                        |
| ISOC  | 互联网协会                   | Internet Society                                    |
| RFC   | IETF正式发布的文稿名称       | Request for Comments                                |
| STD   | 标准（RFC）                  | Standard (RFC)                                      |
| WG    | 工作组                       | Working Group                                       |



## 二、撰写RFC

在撰写RFC之前需要先了解一下RFC的流程，首先需要撰写一个符合IETF格式的文档xml、txt、pdf格式都可以，但是推荐使用xml格式，后续借助xml2rfc能够更好的转换成标准格式草案。

### 2.1 RFC流程

RFC处理过程：一个RFC文件在成为官方标准前一般至少要经历三个阶段：建议标准、草案标准、因特网标准。

> 第一步RFC的出版是作为一个Internet草案发布，可以阅读并对其进行注释。准备一个RFC草案，我们要求作者先阅读IETF的一个文档"Considerations for  Internet Drafts". 它包括了许多关于RFC以及Internet草案格式的有用信息。作者还应阅读另外一个相关的文档RFC 2223 "Instructions to Authors"一旦文档有了一个ID号后，你就可以向rfc-editor@rfc-editor.org发送e-mail，说你觉得这个文档还可以，能够作为一个有价值或有经验的RFC文档。RFC编辑将会向IESG请求查阅该文档并给其加上评论和注释。你可以通过RFC队列来了解你的文档的进度。一旦你的文档获得通过，RFC编辑就会将其编辑并出版。如果该文档不能出版，则会有email通知作者是什么原因。作者有48个小时来校对RFC编辑的意见。我们强烈建议作者要检测拼写错误和丢字的错误，应该确保有引用，联系和更新相关的信息。如你的文档是一个MIB，我们则要你对你的代码作最后一次检测。一旦RFC文档出版，我们就不会对其进行更改，因此你应该对你的文档仔细的检查。有时个别的文档会被正从事同一个项目的IETF工作组收回，如是这种情况，则该作者会被要求和IETF进行该文档的开发。在IETF中, Area  Directors (ADs) 负责相关的几个工作组。这些工作者所开发的文档将由ADs进行校阅，然后才作为RFC的出版物。如要获得关于如何写RFC文档和关于RFC的Internet标准制定过程的更多详细信息，请各位参见：[RFC 2223](https://tools.ietf.org/html/rfc2223) "Instructions to RFC Authors"。

流程图如下：

![RFC处理流程](https://raw.githubusercontent.com/louielong/blogPic/master/imgUTjMKtQ.jpg)



### 2.2 RFC种类

撰写草案并希望草案成为IETF标准的人必须阅读[BCP9](https://tools.ietf.org/html/rfc2026)，以便能够跟进其文档在整个过程中的进展情况。您可以在IETF Datatracker上跟踪文档进展情况，网址为<http://datatracker.ietf.org>。 BCP9（以及对它进行更新的各种其他文档）对常常被人误解、甚至是被资深IETF参与者误解的话题进行详细论述： 不同类型的RFC将经历不同的流程并有不同的排名。有六种RFC：

| 英文                                                  | 中文                              |
| ----------------------------------------------------- | --------------------------------- |
| Proposed standards                                    | 建议的标准                        |
| Internet standards (sometimes called "full standards")| 互联网标准（有时称为“完全标准”）|
| Best current practices (BCP) documents                | 当前最佳实践 (BCP) 文档           |
| Informational documents                               | 信息性文档                        |
| Experimental documents                                | 实验性文档                        |
| Historic documents                                    | 历史性文档                        |

**[NOTE]**Only the first two, proposed and full, are standards within the IETF. A good summary of this can be found in the aptly titled [[RFC 1796](https://datatracker.ietf.org/doc/rfc1796)], "Not All RFCs Are Standards".

**[备注]**只有前面两种标准（建议的标准和完全标准）属于IETF内的标准。在题为《并非所有RFC都是标准》[RFC 1796](http://www6.ietf.org/tao-translated-zh.html#RFC1796)的文档中可找到相关摘要。

### 2.3 如何粗暴的撰写一篇IETF Internet Draft

撰写一篇标准的IETF draft除了需要对想要撰写的Draft内容有规划外还需要了解Draft提交的标准格式，但是其相关内容繁杂且啰嗦(如果想优雅的撰写一篇RFC Draft请参阅[如何优雅的撰写一篇RFC Dratf](#jump0))，本着太长不看原则，这里使用简单“粗暴”的方式完成一篇标准格式的Draft就是接下来要介绍的(ps.这里的粗暴仅只将Draft格式转换为IETF的RFC格式)

首先需要参照标准的RFC格式自行完善想要写的RFC draft内容可以是word格式，主要包括：摘要（Abstract）、说明（Introduction）、正文(Content)、引用（Reference）、作者信息（Author’s Address）、致谢（Acknowledgement）等。需要额外指出的是作图必须用ASCII码的形式画图，内容必须是**全英文**，尤其是**标点符号**这点会在xml转换成rfc时尤为重要。

#### 2.3.1 xml格式套用

选择一份RFC draft的xml格式作为模板将写好的文档套用进去，如使用[draft-wang-nfvrg-network-slice-diverse-standards-00.xml](https://www.ietf.org/id/draft-wang-nfvrg-network-slice-diverse-standards-00.xml)，将其保存在本地，推荐使用`notepad++`或者`sublime txt`打开，当然如果你熟练使用`记事本`也没问题，关于xml格式的官方说明见[RFC7749](http://xml2rfc.tools.ietf.org/rfc7749.html)，下面做简要说明。

##### 2.3.1.1 xml模板头

以下部分为xml模板头文件，除了文档名`docName`其他不需要更改。

```xml
<?xml version="1.0" encoding="US-ASCII"?>
<!-- This is built from a template for a generic Internet Draft. Suggestions for
     improvement welcome - write to Brian Carpenter, brian.e.carpenter @ gmail.com
     This can be converted using the Web service at http://xml.resource.org/ -->
<!DOCTYPE rfc SYSTEM "rfc2629.dtd">
<!-- You want a table of contents -->
<!-- Use symbolic labels for references -->
<!-- This sorts the references -->
<!-- Change to "yes" if someone has disclosed IPR for the draft -->
<!-- This defines the specific filename and version number of your draft (and inserts the appropriate IETF boilerplate -->
<?rfc sortrefs="yes"?>
<?rfc toc="yes"?>
<?rfc symrefs="yes"?>
<?rfc compact="yes"?>
<?rfc subcompact="no"?>
<?rfc topblock="yes"?>
<?rfc comments="no"?>
<rfc category="info"
     docName="draft-long-nfv-nfv-decoupling-test-00"
     ipr="trust200902">
```

![IETF Draft xml template](https://raw.githubusercontent.com/louielong/blogPic/master/imgW1fXOks.png)

##### 2.3.1.2 Draft文档命名

Draft的命名遵循如下的格式

> draft name: draft-’authorname’-'groupname’-'drafttopic’-'version’.xml
>   (example: draft-long-nfvrg-nfv-decoupling-test-00.xml

- draft：表明这只是一个draft草案
- authorname：作者名，通常是第一作者的姓氏
- groupname：想提交草案审核讨论的工作组
- drafttopic：草案核心话题内容，多个关键词之间用`-`间隔
- version：版本号，初始版本号为`00`，随着后续的改进，版本号逐次增加

##### 2.3.1.3 作者信息

作者的信息，如下所示基本都是正常填空即可

```xml
    <author fullname="xxx" initials="Y." surname="Long">
      <organization>xxxx</organization>
      <address>
        <postal>
          <street>xxxxxx</street>
          <city>Beijing</city>
          <code>101111</code>
          <country>P. R. China</country>
        </postal>
        <email>xxx@xxx</email>
      </address>
    </author>
```

效果图

![作者信息](https://raw.githubusercontent.com/louielong/blogPic/master/imgTy92Cp4.png)

##### 2.3.1.4 文档信息

文档信息包括：

- date：提交日期，这个关乎到6个月草案过期时间的计算
- area：领域，草案所属的领域
- workgroup：草案打算提交的工作组
- keyword：关键词

```xml
    <date day="26" month="Dec" year="2018"/>
    <area>Networking</area>
    <workgroup>nfvrg</workgroup>
    <keyword>NFV Decoupling Test</keyword>
```

##### 2.3.1.5 正文格式

- 文档正文，正文内容使用`<middle>xxxx </middle>`格式
- 章节，使用`<section>xxx</section>`格式即可，多级标题直接嵌套即可
- 段落，使用`<t>xxx</t>`的格式，保证tag闭合即可
- 文章内引用，使用`<xref target="ETSI_NFV_GS_002"/>`，需要指出的是参考文献需要显示列出该参考文献信息，否则会在xml转rfc时出现`warnning`

##### 2.3.1.6 列表

列表格式如下所示，使用`<list> </list>`标签

```xml
<t><list style="symbols">
    <t>Virtualized Network Function (VNF).</t>
    <t>Element Management (EM).</t>
    <t>NFV Infrastructure, including: Hardware and virtualized resources,
        and Virtualization Layer.</t>
    <t>Virtualized Infrastructure Manager(s) (VIM).</t>
    <t>NFV Orchestrator.</t>
    <t>VNF Manager(s).</t>
    <t>Service, VNF and Infrastructure Description.
        Operations and Business Support Systems (OSS/BSS).</t>
    </list></t>
```

效果图

![ETSI NFV architecture](https://raw.githubusercontent.com/louielong/blogPic/master/imgtUEFgvZ.png)

对于列表的格式使用`style`标签，可选参数有

> "empty"
>
> For unlabeled list items; it can also be used for  indentation purposes (this is the default value when there is an  enclosing list where the style is specified).
>
> "hanging"
>
> For lists where the items are labeled with a piece of text.
>
> The label text is specified in the "hangText" attribute of the <[t](http://xml2rfc.tools.ietf.org/rfc7749.html#element.t)> element ([Section 2.38.2](http://xml2rfc.tools.ietf.org/rfc7749.html#element.t.attribute.hangText)).
>
> "letters"
>
> For  ordered lists using letters as labels (lowercase letters followed by a  period; after "z", it rolls over to a two-letter format). For nested  lists, processors usually flip between uppercase and lowercase.
>
> "numbers"
>
> For ordered lists using numbers as labels.
>
> "symbols"
>
> For unordered (bulleted) lists.
>
> The  style of the bullets is chosen automatically by the processor (some  implementations allow overriding the default using a Processing  Instruction).

2.3.1.7 图示格式

图例格式如下所示，采用`<figure> </figure>`标签，配合`<artwork></artwork>` CDATA表示直接显示的数据

```xml
<t><figure  align="center">
    <artwork name="Network Function Virtualization"><![CDATA[
                                   +----------------+
                                   |    Vendor      +--+
                                   |      C         |  |
                                   +------+---------+  |
                                          |            |
                +---------------+  +------+---------+  |
                |    Vendor     +--+    Vendor      |  |
                |      H        |  |      H         |  |
                +------+--------+  +------+---------+  |
                       |                  |            |
                +------+------------------+---------+  |
                |              Vendor               +--+
                |                F                  |
                +-----------------------------------+
              Figure 3: NFV decoupling test architecture]]></artwork>
    </figure></t>
```

效果图

![图例](https://raw.githubusercontent.com/louielong/blogPic/master/imgDBCpFP1.png)

2.3.1.8 表格格式

表格格式如下所示，采用`<texttable> </texttable>`标签，标题使用`<ttcol> </ttcol>`,单元格使用`<c> </c>`

```xml
		<texttable align="center">
			<ttcol>Testcases</ttcol><ttcol>Vendor F</ttcol><ttcol>Vendor H</ttcol>
		<c>Onboard</c><c>PASS</c><c>PASS</c>
		<c>Instantiate</c><c>PASS</c><c>PASS</c>
		<c>Scale in(manual)</c><c>PASS</c><c>PASS</c>
		<c>Scale out(manual)</c><c>PASS</c><c>PASS</c>
		<c>Terminate</c><c>PASS</c><c>PASS</c>
		</texttable>
```

效果图

![表格](https://raw.githubusercontent.com/louielong/blogPic/master/imgyhiLozn.png)

##### 2.3.1.7 引用

正文后的标签`<back></back>`，填写引用文献信息，文献的格式参见[Reference Examples](https://www.rfc-editor.org/ref-example/)，需要给出文章名、作者、时间等信息。

1）RFC文档引用方式

如果是标准的RFC文献可以使用[官方参考](https://www.rfc-editor.org/rfc-index2.html)直接导出。

![IETF Ref 示例](https://raw.githubusercontent.com/louielong/blogPic/master/imgVNPjSPy.png)

格式如下

```xml
<?rfc include="https://www.rfc-editor.org/refs/bibxml/reference.RFC.8174.xml"?>
```

效果图

![RFC 8174 引用效果图](https://raw.githubusercontent.com/louielong/blogPic/master/imgF1yEJkc.png)

2）非RFC文档引用

非RFC文档需要手动填写相关信息，如下所示

```xml
	  <reference anchor="ETSI_GS_NFV_002"				 target="https://www.etsi.org/deliver/etsi_gs/NFV/001_099/002/01.02.01_60/gs_NFV002v010201p.pdf">
        <front>
          <title>ETSI GS NFV 002: "Network Functions Virtualisation (NFV); Architectural Framework"</title>
          <author>
            <organization>ETSI NFV ISG</organization>
          </author>
          <date month="December" year="2014"/>
        </front>
      </reference>
```

效果图

![非RFC文档引用](https://raw.githubusercontent.com/louielong/blogPic/master/img0ZBvLkX.png)



2.3.1.8 其他注意事项

**转义符**，文章中出现的如下字符必须转义，否则在转换过程中或出错，注意字符后的`;`也是需要的

|转义后字符          | 转义前字符  |
|--------------------|-------------|
|`&amp;` 或 `&#38;`  |&            |
|`&lt;` 或 `&#60;`   |<            |
|`&gt;` 或 `&#62;`   |>            |
|`&quot;`            |"            |
|`&nbsp;`            |空格         |
|`&copy;`            |版权符 &copy;|
|`&reg;`             |注册符 &reg; |
**【Note】**其中&lt;和&gt;最常用，对于I-D内容中包含的xml元素时，需要对<和>进行转义。

**还有最重要的是务必不要出现中文字符，在后续检测中报错的很大一部分问题出在未知的中文标点符号中。**

#### 2.3.2 XML转RFC

在完成以上步骤后，需要进行xml转rfc检测，检测过后的xml格式才能提交，简单点的是使用线上的[xml2rfc](http://xml2rfc.tools.ietf.org/)工具进行debug测试

![xml2rfc debug](https://raw.githubusercontent.com/louielong/blogPic/master/img4yiaq9L.png)

如果出现问题就看debug信息，最基本的错误是tag没有闭合，中文字符，引用错误等。

如果转换没有问题，则可以提交审阅了，链接：[Internet-Draft submission](https://datatracker.ietf.org/submit/)



### 2.4 <span id="jump0">如何优雅的撰写一篇IETF Draft</span>

想要优雅的撰写Draft，你只需要仔细阅读以下文献即可

- [Internet-Drafts](https://www.ietf.org/standards/ids/)，I-D指南
- [Guidelines to Authors of Internet-Drafts](https://www.ietf.org/standards/ids/guidelines/)，撰写Draft指南
- [RFC 7749, The "xml2rfc" Version 2 Vocabular](http://xml2rfc.tools.ietf.org/rfc7749.html)，xml2rfc格式说明
- [RFC5385](https://tools.ietf.org/html/rfc5385)，介绍了如何从word文档转换为I-D RFC的标准文档格式
- [IETF RFC tools](https://www.rfc-editor.org/materials/tools_ids_rfcs_94.pdf)，RFC工具



## 【参考链接】

1）[IETFwiki介绍](https://zh.wikipedia.org/wiki/%E4%BA%92%E8%81%94%E7%BD%91%E5%B7%A5%E7%A8%8B%E4%BB%BB%E5%8A%A1%E7%BB%84)

2）[IETF之道中文](http://www6.ietf.org/tao-translated-zh.html#rfcs.ids)

3）[The Tao of IETF](https://www.ietf.org/about/participate/tao/)

4）[互联网最新研究方向与IETF](https://www.wxwenku.com/d/105646092)

5）[撰写IETF Internet Draft的技巧](http://blog.donews.com/Linyi/archive/2009/03/09/1475585.aspx)


