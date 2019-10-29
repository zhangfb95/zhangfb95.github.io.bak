---
layout: post
title:  2019-09-07半月会议心得
date:   2019-09-07 22:01:30 +0800
categories: 半月会议
tag: 其他
---

* content
{:toc}

本次半月会议，有新同事的入职心得分享，也有3位转正同事分享了他们的心得。印象比较深刻的是候熙的转正分享，幽默诙谐，努力坚持。分享中，他提到了新疆的辛苦（天天吃肉，没有蔬菜）和自己的坚持（朋友觉得太远，但是自己却对公司有感情）。他希望公司变得更好，把自己应该要做的事情做到极致。这次的分享，他未提到具体的技术，但是通过一些真实的情况和进展让我们对公司有了一个新的认识。年初我来公司的时候，我们正在如火如荼地做斑马信用项目、贵阳放管服项目，也在分析新疆的项目。那时候，我们没有产品的概念，缺少整体的战略和布局。虽然做了很多事情，但是最终的目标却是比较模糊的，我们做的东西凝聚力不够强，没有让客户满意。而这次，我们看到了一些很可观的成果。斑马信用已成功上线，正在大力推广。新疆项目快要进入最终验收。我们的付出和辛苦得到了客户的认同，我们做的事情正在造福人类。

接下来是符号同学的分享，主要包括两个部分：
1. 交通领域的基础知识
2. 斑马信用的简单介绍

技术出身的我们，对于交通领域的知识还是较为欠缺的。通过这次分享，我们了解了交通领域也会有各种需求，广大市民的，政府的，商户的。同时，近年来，政府也在交通领域有一些政策上比较好的支持，比如加强信用建设、加强信息化建设等。背靠海康威视这颗大树，我们也可以在交通领域有一定的市场资源。我们公司立足于软件，着眼于大数据和最新最前沿的技术，我们能提供一个很好的基于C、G、B三端的交通大脑。而对于政府在交通领域的各种组织机构，我也有了一定的了解。以前对于公交、高速、红绿灯、路政、交警等这些名词的认识还是太肤浅了。符号同学站在全局的、更高的层级上给我们分享了组织结构的划分。如：交通主要部门（交通部、交通厅、交通局）、行业管理部门（公路局、运管局、高管局）、从业企业（高速集团、运输集团、公交集团）和大众公民（驾驶人和行人）。在互联网领域，我们经常会听到几个词语 -- 跨界、融合、创新和开放。这些词语带来了互联网的思维，我们在交通领域能提供这样的一种能力。公司最核心的价值在于数据，尤其是数据采集、分析、挖掘和模型。数据的价值有时是无法估量的，甚至是能无限扩大的。我们通过开放思维、共享数据、提供能力，为业界和跨业界的合作伙伴提供更近一步的价值。我们可以打通保险行业，还可以打通4S店等汽车后市场，通过互利互赢的方式达成战略上的合作，为社会和谐带来增值。

到了下午，我们召开了一个很有意思的迭代总结会。在王澍的主持之下，我们将参会人员分成了两组。每一组将斑马信用v1.6.0迭代做得好的、可改进的和不好的方面以卡片的形式写出来。然后通过小组长来进行聚合汇总，并对每组的卡片进行宣讲。然后由王澍同学将两个小组的结果再进一步聚合，将我们能做的、我们不能做的（但能变相解决的）、后续可做的事项进行归纳总结。对于需求变更这个老生常谈的问题，我们讨论了很久。起初，大家都有不同的想法，有人认为应该由开发人员负责跟踪到底，有人认为应该由测试来做（因为他们是质量保证部门），还有人说应该通知到项目经理，由项目经理来跟进。各种不同的想法，五花八门，零散，没有统一。这些想法都很好，也都能解决我们的部门问题。这些讨论也从侧面说明大家都有自己的想法。但是，我认为最关键的点，还在于如何高效、持续地跟进并落地变更的需求。怎么做呢？

1. 在开发阶段，由Scrum Master负责跟进，和产品确认，并通知到Story的参与者，尤其是开发。
2. 在测试阶段，由测试人员负责跟进，在和产品达成一致的情况下，将变更的效果通知到Story的参与者。
3. 所有的阶段，都需要通知到项目经理，由其进行全面地统筹。

这种激烈的讨论带来了立杆见影的效果。即：我们统一了想法，达成了共识，形成了流程。而这也是蒲总提到的，属于PMO职责的一部分，非常好。

让我印象深刻的还有关于迭代版本的输出。我们没有统一工具，每个组甚至每个组内不同的人，都可能将这些输出放置在不同的地方，甚至只记忆在脑子里面。这导致的问题就是，上线的过程会比较混乱，配置和脚本可能会有遗漏、不一致和重复的情况，上线顺序和质量没法保证。同时，对于后期要做的回顾和追溯，我们并没有办法，只能凭借经验和记忆。这是一件极端可怕的事情，同时也是一种现状。我觉得，公司在做大之前，一定要统一一些东西，尤其是这种很重要的迭代输出。然后呢，在PMO的带领下，我们还是达成了一致。结论就是，统一使用SVN工具来管理每个迭代版本的输出。每个产品线会有各自不同的目录，每个产品线下又有不同的版本，每个版本下还有不同的参与端，每个参与端需要整理并定义各自的规范（文件夹、文件命名、文件格式等都需要统一）。

这种规范，让我认识到了统一带来的价值。相应地，我也在反思公司和个人的关系。一个人成不了公司，一个成功的公司必须要有很多人的努力、沉淀和付出。在此期间，我们需要统一一些东西，统一思想、统一工具、统一规范、统一平台。

这次的总结会，让我印象深刻，也让我收获了很多。我相信参会的人也会印象深刻。希望后续还可以多多召开类似这样的会议，剔除掉不好的，改进需要改进的，让好的变得更好。