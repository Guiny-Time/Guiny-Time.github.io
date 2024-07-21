---
title: AI Agent 与自动化工作流
date: 2024-06-17 14:04:11
tags: ['人工智能','AIGC','Agent','智能体工作流']
categories: 工作记录
cover: https://cdn.midjourney.com/9d623c95-0439-4d88-a90c-ba158d279570/0_0.png
hide: true
---
{% note info simple %}
**请注意**
**本文还在施工中**
{% endnote %}

# 为什么我们需要Agent？

最近听了国家天文台李瑀旸博士的分享会（真的是非常好的一个人TT），组里的一个任务是用coze搭建一个科幻小说脚本->图册->视频的完整智能体工作流，今天决定落实一下这个工作，遂写之。

相信大家平常在用LLM的时候经常遇到一个问题：给LLM一个指令，在一段时间内工作良好，但之后它就开始胡言乱语了。其中原因主要有二：

1. LLM对长上下文的处理能力较差
2. LLM的上下文环境存在内存限制

对长上下文的处理能力较差，导致当我们输入长文本或者要求LLM输出长文本时可能无法很好地得到预期答案；而对上下文环境存在内存限制，导致LLM在一段对话之后可能会忘记之前的信息或指令，这种限制是由于模型的架构和内存需求所决定的。

这就导致，当我们试图用LLM处理一个大型、复杂的任务（比如写一本长篇小说）时效果不好，缺乏任务全局的**规划**（Planning）和对指令的**记忆**（Memory）。而接下来提到的AI Agent，则可以理解成LLM + 规划 + 记忆 + 工具的一种解决方案。

![](https://cfiles.51aspx.com/wp-content/uploads/2023/12/20231205063132829.png)

总的来说，Agent是一种以LLM作为内核、能够自主进行规划、调用工具和记忆的工具。它可以做到分解大型任务、对过去的行为进行自我反思、形成长期记忆、使用外部API辅助工作....非常接近一个真实的人处理问题的方式。而且Agent是高度自定义的，你完全可以把它训练成自己想要的样子。

Agent flow，也就是用Agent将不同大模型的能力衔接起来整理成一个工作流，就是这篇文章要介绍的事情。

# 搭建闭源Agent
闭源智能体平台的好处是提供了众多可直接调用的Plugins，比如coze和阿里云百炼。本文用coze，不过在用的时候遇到了很玄学的问题，就是不知道为什么在chrome搭workflow的时候一直报trigger_error（似乎是php的报错），但换Edge之后就好了。可能和我浏览器选项卡开太多了有关系。

# 资料
<a href="https://lilianweng.github.io/posts/2023-06-23-agent/">LLM Powered Autonomous Agents</a>
<a href="https://www.coze.com/docs/guides/use_workflow?_lang=zh">coze document</a>
![](Oq9s_i_无边框.png)