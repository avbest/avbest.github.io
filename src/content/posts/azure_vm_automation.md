---
title: 关于 azure 虚拟机自动化任务的问题
published: 2026-09-10
tags: []
category: 'Azure'
draft: false
---

最近终于把azure学生订阅送的每年$100额度用起来了，但是这个虚拟机怎么自动开机成了问题，因为不需要它一直开机，所以说要定时开关机省钱，要不然 $100是不够撑一年的.

自动关机是很好办的，Operations > Auto-shutdown里面就可以设置

![](https://i.imgs.ovh/2026/09/10/95b66e9a89346c908f51a26a72cbade0.png)

但是自动开机却成了个难事，因为不知道为什么，Automation > Tasks里面创建的自动化任务是不好使的，运行不起来，每次都报401或者404，研究了一晚上到凌晨一点也没想出办法了，后来终于明白了，不能靠这个，所以在此整理出一个办法来.

首先要创建一个自动化账户，大体思路是把它链接一个schedule，然后定期执行开机任务.

侧边栏，Create a Resource，搜索automation

![](https://i.imgs.ovh/2026/09/10/6ee84da71238b12bd1f8bd7ec1fe96b5.png)

第一个就是，进去之后创建就行，注意区域建议选Southeast Asia，因为国内大学的教育订阅能使用的区域有限，资源组一定要选和你虚拟机同一个组.

等待部署完毕，点进Process Automation > Runbooks，然后Create

![](https://i.imgs.ovh/2026/09/10/5ad19e042cb9c867757044a775a30391.png)

第一个默认Create new，第二个随便写个名字你将来能认出来就行，第三个选PowerShell，咱们用不上py.

注意，选完PowerShell之后底下会出来一个Runtime Environment，点Create New，然后名字起一个，Language选PowerShell，Runtime Version选最新的就行.

剩下就看着来，创建完成之后点进去，Edit > Edit in portal.

命令这么写

```
az login --identity
az vm start -g <vm_grp> -n <vm_name>
```

把<vm_grp>和<vm_name>换成你自己的虚拟机组和名字就行.

之后点Save，然后Publish

接着需要让这个账户有权限操作咱们的虚拟机，在automation账户里，点Account Settings > Identity，在System assigned里面，把Status开成On，然后系统会产生一个Object ID.

![](https://i.imgs.ovh/2026/09/10/90d4a150520e1be3b470fc1a55f14acc.png)

回到虚拟机，点Access Control，然后Add role assignment.

第一块Role，搜Virtual machine，然后选Virtual Machine Contributor.

![](https://i.imgs.ovh/2026/09/10/ac1b93a191073807a257c77407c0da0e.png)

下一步Members，选Managed Identity，然后点底下Select Members，在新出来的窗口里的Managed Identity选Automation Account，里面就有个你创建的账户了，点它，然后一路默认就行了.

![](https://i.imgs.ovh/2026/09/10/8707c8780aa3c106cf09d8be655224f0.png)

这样这个自动化账户就有操作你虚拟机的权限了，至此任务已经能正常工作了，如果你不放心也可以进去试试.

下一步是制定运行周期，在runbook里面，点Resources > Schedules，然后Add a schedule.

![](https://i.imgs.ovh/2026/09/10/57b6867d4fabf41fe74486cca2a0e4f4.png)

这两个都要选，第一个点进去之后自己创建一个周期就行了，相信你会做这个，第二个点进去选默认的就行，然后OK.

这样启动虚拟机的任务就自动挂到一个周期上啦.