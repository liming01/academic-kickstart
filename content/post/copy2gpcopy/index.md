---
title: 从COPY的并行化到gpcopy工具的诞生（未完）
subtitle: 记录对gpcopy工具重要的人和事
summary:  记录对gpcopy工具重要的人和事
authors:
- admin
tags:
- gpdb
- gpcopy
- COPY ON SEGMENT
categories: []
date: "2017-03-13T11:00:00"
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal point options: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight
# image:
#   caption: 'Image credit'
#   focal_point: "Top"
#   preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

------ 作者：[李明](https://github.com/liming01) (email: mli@apache.org)

由于公司战略调整，数据库的工作重心从HAWQ转向Greenplumn，我悻悻离开了奋斗好几年的HAWQ组，来到了gpdb大组下的unite组。

# COPY的并行化（COPY ON SEGMENT）

来到新组，除了做一些bug的修复工作，我开发的第一个feature是Copy On Segment, 即COPY的并行化。COPY是PostgreSQL支持的SQL语句，能够将数据批量地导入或者导出数据库。gpdb是基于pg开发，以前的版本在原有的单节点的COPY命令，修改成多节点协调，所有数据都可以通过master节点导入导出，master节点再将被处理的数据引导到对应的segment节点上处理，等处理完毕后将执行的结果（成功行数、失败行数等）都统一汇总给master，最后再由master反馈给客户端。由于修改流程比较复杂，导致src/backend/commands/copy.c的代码从pg里的5k多行一下涨到了gpdb的8k多行。

原先的这种处理方式，所有的数据交互都需要通过master节点来完成，对master节点的性能影响很大。更为重要的问题是COPY的性能不能满足需求，gpdb的处理的数据量比单节点pg的数量可以高出两个数量级，而COPY的这种处理方式很大程度限制了它所能处理的最大数据流量，特别是在segment数量很大（比如超过100时）表现更为明显。

而COPY ON SEGMENT这个新特性就是为了解决这个问题而提出的，它将要处理的数据直接在segment节点导入或者导出，master节点只负责整个命令的协调（比如命令的启停、报错等）和最后执行的结果的汇总。COPY ... TO ... ON SEGMENT比较简单，直接将数据导出到各个segment上的文件，并将结果汇报给master就可以。COPY ... FROM ... ON SEGMENT就比较复杂了，需要了解整个COPY在gpdb中怎么处理，特别是在master和segment之间如何相互分工协作，再搞清楚整个大背景之后，再去提出一个好的方案，通过小改动去实现整个COPY流程的整合。具体的细节可以参见[COPY ... FROM ... ON SEGMENT的commit](https://github.com/greenplum-db/gpdb/commit/e254287ef997b2b33cdca5ec627bd9fc163d12e3)

# 工具gpcopy的诞生

除了做COPY ON SEGMENT这个新特性之外，我们组的还负责维护gptransfer、gpload、gpfdist、External Table等相关的模块。由于gptransfer和gpfdist的bug很多时候都很难调试，特别是在开了很多并行任务时更难追踪。

当完成了COPY ON SEGMENT的特性之后，基于基本认识：COPY相对于External Table的性能更好，更适合于批量导入导出数据，我们就有基于这个新特性构建一个新的传输数据的工具的想法。随着我们多次讨论它的性能优势，并对他进行小规模的模拟性能测试（至少有三分之一的性能提升），我们更加坚定了这样做的必要性。

与此同时，gpdb组投入了很多人力在进行炉火如荼的pg kernel升级，但是由于人手资源的缺乏，没能及时抽出人手搞对原有旧版gpdb的版本升级工具。这时候不知道经过谁的推荐（是不是我们组的前PM：[Jasper](https://github.com/lij55)？这里要感谢一下这个引荐人），我们得到上头的授权说我们可以投入人力和时间来搞gpcopy。于是我们组的[Adam](https://github.com/adam8157)利用一个星期的时间用golang语言写了个prototype，于是这个项目就诞生了。同时，我们的项目从刚开始就被定位为闭源项目。

# 追随gptransfer

从与数据库的交互接口上来看，gptransfer和gpfdist是用External Table与gpdb进行交互的。External Table虽然是逐行处理数据，但是它走的是gpdb/pg的普通查询计划，所以可以很容易的修改计划添加motion节点，来支持数据的重分布（比如segment节点数不一样，需要将全部数据根据它的数据分布策略重新计算，从而分发到新的segment节点上）。而COPY ON SEGMENT衍生自pg的COPY SQL语句，语句的处理不是走普通的查询计划优化，所以要支持数据的重新分布非常困难（目前gpdb无法在不增加motion计划节点的情况下启用inter-connect去实现segment之间的数据交换，至少从代码的修改量上来看很难实现）。

由于刚开始这个项目时我们组的PM [Brian](https://github.com/brianlugongyu)的定位是能够急迫解决gpdb的升级问题，我们就把相同segment数的gpdb之间的数据迁移作为我们的工作重点。除了重新实现gptransfer（用Python语言实现的，底层调用的gpfdist命令是用C写的）的所以相关功能，我们投入了很大的精力，在易用性、报错信息的容易理解性、性能等上面做出自己的亮点。特别值得一提的是gpcopy如果中间出现传输错误的表，会打印出重跑命令，用户可以通过拷贝来快速重新跑失败的任务，避免任务全部重新再跑一遍。另外由于gpcopy相对于gptransfer和gpfdist在并发控制和系统架构上要简单很多，这直接导致我们定位问题的难度降低很多、解决修复问题的速度也快了很多。

# 超越gptransfer

随着第一个版本的发布，用户的反馈非常好，很快用户就提出需求，希望全面替换gptransfer的功能。我们开始想办法支持不同segment数量的两个gpdb cluster之间的数据传输。期间很多好的想法被提出来，不断得尝试实现，特别是我们组的[Bob](https://github.com/baotingfang)通过很简单的方案就窍门地解决了从segment数量少的gpdb cluster导到segment数量多的gpdb cluster。还有针对小表直接走master节点，不通过segment去启动多个并行任务，来减少启动的时间。

针对从segment数量少的gpdb cluster导到segment数量多的gpdb cluster的情况，我们是在是没有办法，最后又回到了利用External Table来实现的老路上来。

这里有篇文章介绍了这个时候的gpcopy：[Greenplum数据迁移工具gpcopy升级到 1.1.0](https://cloud.tencent.com/developer/news/355959)

从这篇文章可以看出gpcopy的性能在源gpdb和目标gpdb的segment数量一样的情况下，性格提升好几倍。读者可能对这个数据不是很直观，我举个例子：在国内最大客户的gpdb的最大数据量的gddb cluster上用gpcopy导到本地升级了的gpdb版本上，所花的时间要1天多点，这个时间在周末时候进行升级完全满足需要，但若是换成用gptransfer或者其他工具来导数据的话，那基本上这个在线系统需要停机一周以上，这样的维护时间，哪个公司能够承担得了呢？

除了这篇文章里介绍的，我觉得还有两点值得说一下：

- 正则表达式的支持：通过制定正则表达式，将所有需要传输的表都列出来。

- 特殊字符的支持： 我和[Bob](https://github.com/baotingfang)花了很多时间想要试着去解决gptransfer的特殊字符问题，但是由于启动gpfdist的路径与表名存在映射关系，导致特殊字符会出现在目录的路径上，这需要再加一层的转义，转义的规则也不一样，必须符合操作系统的目录的转义。而gpcopy的转义虽然也需要几层：比如Shell命令转义、SQL的字符串转义和标识符的转义，pg_dump的特殊字符处理等，这些转义相互套了好几层之后变得满目全非，问题调试起来也非常不容易，但是这些问题同样在gptransfer里也是一样存在的，如果它要实现特殊字符的支持，这些功能也是必要条件。

有了这些更多的功能，再加上它更强的性能，更简单的问题发现和调试等优点，gpcopy牢牢占住了gptransfer的原来的位置。
以至于没过多久，PM们通过用户和技术支持团队的反馈，做出决定：不再支持gptransfer，同时将gptransfer的代码从gpdb中移除。（其实这点我是不赞成的，我觉得就算我们公司不想再对它技术支持，开源的源码也应该保留gptransfer。毕竟gpcopy是一个闭源项目）

# 更上一层楼

自从pg的COPY支持SQL语句，gpdb随着pg内核版本升级，也轻而易举地支持COPY（SQL）TO...了，但是同样这个特性也是只支持SQL查询的数据收集到master再又master分发到各个segment再到出去。如何支持ON SEGMENT从句，从而使得SQL查询的结果不用汇总到master，而直接在各个segment上直接导出来呢？这看起来又成了一个大问题。

很多时候下班晚走，我和[陈金豹](https://github.com/baobao0206)就常常聊起这个问题，聊起解决方案，他想到了可以参照CREATE TABLE AS SQL来实现。虽然我们都很憧憬着这个特性对gpcopy工具的新功能的影响，迫切期待，心急如焚得等待时机。那段时间[陈金豹](https://github.com/baobao0206)一直在从事pg内核版本升级代码merge等工作，而当时我们组在忙于另一个新项目gp2gp（gpdb和gpdb之间的联邦并行查询）。终于[陈金豹](https://github.com/baobao0206)自告奋勇，找了个机会抽出时间来自己完成了调研和实现。这是他实现的COPY (SQL) TO ...ON SEGMENT语句中支持SQL的[Pull Request](https://github.com/greenplum-db/gpdb/pull/6077)和[Commit](https://github.com/greenplum-db/gpdb/commit/bad6cebc942ad2abc77b36b4d3a1d55236e33a18)。这里特别要感谢他做出的贡献，不单单是因为他是别的组，更要感谢他对技术实现的痴迷和对技术商业价值的敏感！

同时在他的帮助下，我们组的[林文](https://github.com/linwen)和他一起努力将这个令人兴奋的特性backport到老版本gpdb5上。到这里，我们gpcopy就可以很容易支持针对特定表的特定查询（比如可以只查询某个特定时间段）进行并行导出了。有了这个功能，客户就可以用gpcopy来给数据做增量的数据迁移了。这个新功能一下子将gpcopy的从偶尔使用的工具变成了日常使用的工具，也是它从数据迁移的特性使用场景中拓展出其他的使用方法。

为了配合这个特性，我们组新来了[杨峻峰](https://github.com/JunfengYang)很快实现了针对追加方式的验证，只要验证目标表中新插入的数据，而不是全部数据。另外我们还实现了重命名目标表：利用正则表达式的组概念进行对目标表的改名。另外我们组又来了Chen Mulong，也开始加入对gpcopy进行精雕细琢。

通过这次新的版本发布，希望gpcopy能够迎来更广的使用场景，拥有有更好的未来！

最后，再次感谢为gpcopy项目做出贡献的同事们！