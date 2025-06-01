---
title: Hyper-V虚拟机新增虚拟硬盘
tags:
  - Linux
  - Hyper-V
  - 虚拟机
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202506011906515.png'
toc: true
abbrlink: c6d39566
date: 2025-06-01 19:04:59
---

笔者在使用Hyper-V虚拟机进行日常运维和环境搭建时，经常会遇到存储空间不足或需要独立数据盘的情况。近期，在为某个Kubernetes集群部署Rook-Ceph分布式存储解决方案时，就遇到了需要为虚拟机动态新增多个虚拟硬盘的需求。这个过程虽然看似简单，但对于初次接触或不熟悉Hyper-V操作的用户来说，每一步的细节都值得仔细推敲。因此，本文旨在详细记录从零开始为Hyper-V虚拟机添加虚拟硬盘的整个过程，并附上详细的图文步骤，希望能为遇到类似需求的朋友提供清晰直观的指引，帮助他们高效地扩展虚拟机存储能力。

<!-- more -->

## 步骤

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202506011854382.png)
![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202506011854058.png)
![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202506011855966.png)
![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202506011855980.png)
![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202506011855096.png)
![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202506011856544.png)
![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202506011857978.png)
![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202506011857349.png)
![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202506011858403.png)
![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202506011903925.png)

## 结语

至此，为Hyper-V虚拟机新增虚拟硬盘的所有步骤已经详细演示完毕。通过本文的图文指南，相信您已经能够轻松地为您的虚拟机扩容，无论是为了部署复杂的分布式存储系统如Rook-Ceph，还是仅仅为了增加日常应用的数据存储空间，这些操作都是必不可少的。掌握这一技能将大大提升您在虚拟化环境下的运维效率和灵活性。希望这篇记录能确切地帮助到您，使您在面对类似需求时能够得心应手，顺利完成任务。后续笔者将继续分享更多关于Hyper-V和Linux运维的实践经验。
