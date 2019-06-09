---
title: "Apache HAWQ的事务处理"

# View.
#   1 = List
#   2 = Compact
#   3 = Card
view: 2

# Optional featured image (relative to `static/img/` folder).
header:
    caption: "My caption :smile:"
    image: "headers/bubbles-wide.jpg"


date: "2017-03-23T15:30:00"

abstract: ""
abstract_short: "HAWQ Transaction Introduction"
event: "Apache HAWQ Meetup @ Beijing"
event_url: "https://cwiki.apache.org/confluence/display/HAWQ/Community"
location: "Beijing, China"

# Is this a featured talk? (true/false)
featured: false

url_pdf: "https://cwiki.apache.org/confluence/download/attachments/61320901/HAWQ%20Transaction%20Introduction.pdf?version=1&modificationDate=1490600276000&api=v2"
url_slides: ""
url_video: "https://v.qq.com/x/page/v0391aslvhs.html"


---

这个是在3月23日举办的HAWQ技术研讨会中本人做的技术演讲，本次演讲首先为大家介绍了PostgreSQL的事务处理中的多版本并发控制，然后结合了HAWQ的应用场景，介绍在HAWQ怎么实现事务并发控制，以及并行插入优化等问题，并解释了一些HAWQ使用的注意事项和将来可以考虑实现的一些方案。


# 追加评论(2019年5月30日)

需要说明一下：
PPT里面引用了第五、六、七页的例子是直接从Bruce Momjian的“MVCC Unmasked”的报告里通过图片拷贝过来的，原来是PPT格式，我在页面的注释处写了引用的出处了。但是在放到HAWQ的wiki之前，公司的人把它转换成PDF格式的，导致所有页面的注释没能导出来。

我负责公司的内部的tech talk，由于这周没有人分享，考虑到接下来的工作多个组可能都要涉及到这些知识，我就准备自己上去分享下相关的背景知识等。由于我们组活儿也比较紧，我在开会的前一天才有时间花了半天左右的时间准备PPT，所以我就从网上找些资料，把相关的东西串起来，有些图片直接从官方的公开文档里拷贝过来。

然后就发生了让我很郁闷的一件事情。有个同事过来跟我说，说所有的图片啥的不是自己画的都必须要有引用出处。还说我这个链接里的公开资料直接引用别人的东西，不注明出处涉及到剽窃行为。对此表示很气愤！先暂且不说我是写了引用出处的，也不是我的原因导致了这个结果。就算是我确实没写出处，我这次演讲的目的本来就是讲HAWQ的事务实现，只是介绍这个实现之前，必须先了解PostgreSQL的事务实现（本来PostgreSQL的事务实现也不是我实现的，根本就不是我原创的东西，何来剽窃一说？）也不知道达到剽窃这么高的一个高度吧？

说起这件事，让我想起来了5月21日华为老总任正非的访谈，说到自主创新的问题，他如是说：**自主创新如果是一种精神，我支持；如果是一种行动，我就反对！**
看到那个访谈，我立刻在微信的朋友圈转发，比附上我的留言：**战略上藐视敌人，战术上要重视敌人！战略上自主创新，战术上还是要站在现有巨人的肩膀上，搞最前沿的创新！**

有些人以为要自主创新就要从每个螺丝钉开始，从最基础的每个东西都是自主的才行。殊不知我们的重心要放在最前沿的创新上，放在最需要我们花力气的地方！可惜这些人永远看不到别人的意图！只知道拿一些条条框框来限制你，不考虑时间、机会、成本，站在道德的制高点，扮成维护完美的圣斗士！我只想说：不要扮演圣人了，让你内部技术分享你不分享，别人分享你就在那边指指点点，你觉得合适吗？