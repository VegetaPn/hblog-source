---
title: IDDD Note 1 - 上下文映射
tags: DDD
categories: ARCHTECTURE
date: 2019-01-14 21:02:56
---


# 上下文映射图

{% asset_img 1.png %}  

U: 上游  
D: 下游  

上下文映射图表示的是项目当前的状态，它不是一种企业架构，也不是系统拓扑图，它展现了一种组织动态能力，可以帮助我们识别出有碍项目进展的一些管理问题  
保持简单和敏捷  

## 上下文之间的关系  

- 合作关系（Partnership）：两个上下文紧密合作的关系，一荣俱荣，一损俱损
- 共享内核（Shared Kernel）：两个上下文依赖部分共享的模型
- 客户方-供应方开发（Customer-Supplier Development）：上下文之间有组织的上下游依赖
- 遵奉者（Conformist）：下游上下文只能盲目依赖上游上下文
- 防腐层（Anticorruption Layer）：一个上下文通过一些适配和转换与另一个上下文交互
- 开放主机服务（Open Host Service）：定义一种协议来让其他上下文来对本上下文进行访问
- 发布语言（Published Language）：通常与OHS一起使用，用于定义开放主机的协议
- 大泥球（Big Ball of Mud）：混杂在一起的上下文关系，边界不清晰
- 另谋他路（SeparateWay）：两个完全没有任何联系的上下文

ACL: 防腐层  
OHS: 开放主机服务  
PL: 发布语言  


