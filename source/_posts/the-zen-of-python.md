---
title: the_zen_of_python
date: 2019-07-24 09:51:15
tags:
 - python
keywords:
  - python
categories:
  - python
description:
  - python禅道
summary_img:
  - https://raw.githubusercontent.com/louielong/blogPic/master/img0BVyYQ1sS8U8GkC920F3.png
---

一、前言

看到“GitHubDaily”公众号推送一个关于[Python-100-Days](https://github.com/jackfrued/Python-100-Days)的一个项目，觉得里面的东西蛮有意思，之前由于工作需要也玩过一段时间的python，但是没有很系统的学习，所以打算花上一段时间根据这个项目进行较为系统的学习。

在教程的第一节就是关于python之禅的介绍，搜索了一下网上的翻译，觉得很有意思这里也转载记录一下。

二、python禅道

打开终端输入`python`，`import this`即可看到Tim Peters的**Then Zen of Python**

```shell
python3
Python 3.5.2 (default, Nov 12 2018, 13:43:14)
[GCC 5.4.0 20160609] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import this
The Zen of Python, by Tim Peters

Beautiful is better than ugly.
Explicit is better than implicit.
Simple is better than complex.
Complex is better than complicated.
Flat is better than nested.
Sparse is better than dense.
Readability counts.
Special cases aren't special enough to break the rules.
Although practicality beats purity.
Errors should never pass silently.
Unless explicitly silenced.
In the face of ambiguity, refuse the temptation to guess.
There should be one-- and preferably only one --obvious way to do it.
Although that way may not be obvious at first unless you're Dutch.
Now is better than never.
Although never is often better than *right* now.
If the implementation is hard to explain, it's a bad idea.
If the implementation is easy to explain, it may be a good idea.
Namespaces are one honking great idea -- let's do more of those!
```

在网上找到关于这段话的翻译，转载自豆瓣[The Zen of Python ](https://www.douban.com/group/topic/3740034/),

>Python之禅  
>赖勇浩翻译
>
>Beautiful is better than ugly.   
>优美胜于丑陋（Python 以编写优美的代码为目标）
>Explicit is better than implicit.
>明了胜于晦涩（优美的代码应当是明了的，命名规范，风格相似）
>Simple is better than complex.
>简洁胜于复杂（优美的代码应当是简洁的，不要有复杂的内部实现）
>Complex is better than complicated.
>复杂胜于凌乱（如果复杂不可避免，那代码间也不能有难懂的关系，要保持接口简洁）
>Flat is better than nested.
>扁平胜于嵌套（优美的代码应当是扁平的，不能有太多的嵌套）
>Sparse is better than dense.
>间隔胜于紧凑（优美的代码有适当的间隔，不要奢望一行代码解决问题）
>Readability counts.
>可读性很重要（优美的代码是可读的）
>Special cases aren't special enough to break the rules. Although practicality beats purity.
>即便假借特例的实用性之名，也不可违背这些规则（这些规则至高无上）
>Errors should never pass silently. Unless explicitly silenced.
>不要包容所有错误，除非你确定需要这样做（精准地捕获异常，不写 except:pass 风格的代码）
>In the face of ambiguity, refuse the temptation to guess.
>当存在多种可能，不要尝试去猜测
>There should be one-- and preferably only one --obvious way to do it.
>而是尽量找一种，最好是唯一一种明显的解决方案（如果不确定，就用穷举法）
>Although that way may not be obvious at first unless you're Dutch.
>虽然这并不容易，因为你不是 Python 之父（这里的 Dutch 是指 Guido ）
>Now is better than never. Although never is often better than *right* now.
>做也许好过不做，但不假思索就动手还不如不做（动手之前要细思量）
>If the implementation is hard to explain, it's a bad idea. If the implementation is easy to explain, it may be a good idea.
>如果你无法向人描述你的方案，那肯定不是一个好方案；反之亦然（方案测评标准）
>Namespaces are one honking great idea -- let's do more of those!
>命名空间是一种绝妙的理念，我们应当多加利用（倡导与号召）

也有简洁版的如下：

>The Zen of Python, 
>蛇宗三字经
>作者：Tim Peters
>翻译：元创
>
>Beautiful is better than ugly. 
>美胜丑
>Explicit is better than implicit. 
>明胜暗
>Simple is better than complex. 
>简胜复
>Complex is better than complicated. 
>复胜杂
>Flat is better than nested. 
>浅胜深
>Sparse is better than dense. 
>疏胜密
>Readability counts. 
>辞达意
>Special cases aren't special enough to break the rules. 
>不逾矩
>Although practicality beats purity. 
>弃至清
>Errors should never pass silently. 
>无阴差
>Unless explicitly silenced. 
>有阳错
>In the face of ambiguity, refuse the temptation to guess. 
>拒疑数
>There should be one-- and preferably only one --obvious way to do it. 
>求完一
>Although that way may not be obvious at first unless you're Dutch. 
>虽不至，向往之
>Now is better than never. 
>敏于行
>Although never is often better than *right* now. 
>戒莽撞
>If the implementation is hard to explain, it's a bad idea. 
>差难言
>If the implementation is easy to explain, it may be a good idea. 
>好易说
>Namespaces are one honking great idea -- let's do more of those! 
>每师出，多有名

在查找的过程中也看到了比较有意思的是上面那段话在python源码[this.py](https://github.com/python/cpython/blob/master/Lib/this.py)中并不是明文存储的，而是使用了凯撒加密法，将每个字符前移13位然后再跟26取余，源码如下：

```python
s = """Gur Mra bs Clguba, ol Gvz Crgref
Ornhgvshy vf orggre guna htyl.
Rkcyvpvg vf orggre guna vzcyvpvg.
Fvzcyr vf orggre guna pbzcyrk.
Pbzcyrk vf orggre guna pbzcyvpngrq.
Syng vf orggre guna arfgrq.
Fcnefr vf orggre guna qrafr.
Ernqnovyvgl pbhagf.
Fcrpvny pnfrf nera'g fcrpvny rabhtu gb oernx gur ehyrf.
Nygubhtu cenpgvpnyvgl orngf chevgl.
Reebef fubhyq arire cnff fvyragyl.
Hayrff rkcyvpvgyl fvyraprq.
Va gur snpr bs nzovthvgl, ershfr gur grzcgngvba gb thrff.
Gurer fubhyq or bar-- naq cersrenoyl bayl bar --boivbhf jnl gb qb vg.
Nygubhtu gung jnl znl abg or boivbhf ng svefg hayrff lbh'er Qhgpu.
Abj vf orggre guna arire.
Nygubhtu arire vf bsgra orggre guna *evtug* abj.
Vs gur vzcyrzragngvba vf uneq gb rkcynva, vg'f n onq vqrn.
Vs gur vzcyrzragngvba vf rnfl gb rkcynva, vg znl or n tbbq vqrn.
Anzrfcnprf ner bar ubaxvat terng vqrn -- yrg'f qb zber bs gubfr!"""

d = {}
for c in (65, 97):
    for i in range(26):
        d[chr(i+c)] = chr((i+13) % 26 + c)

print("".join([d.get(c, c) for c in s]))
```

大致的对应如下：

>ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz
>↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
>NOPQRSTUVWXYZABCDEFGHIJKLMnopqrstuvwxyzabcdefghijklm

这里也简单的作为一个学习开始的记录，勉励自己能够坚持学习下去，用python做一些有意思的小项目玩一玩。

加油↖(^ω^)↗

![加油](https://raw.githubusercontent.com/louielong/blogPic/master/img1355839526-2935297530.jpg)