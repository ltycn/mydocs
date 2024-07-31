---
sidebar_label: APPTrace
sidebar_class_name: lightweight tools
---

# APPTrace

APPTrace是用于测量windows软件开启速度的小工具。

在一些大型软件的启动过程中，我发现在呈现最终用户可交互的界面之前，通常会出现若干个子程序子窗口，用于加载程序所需的额外组件。而从用户启动程序开始的这一秒，主进程的窗口标题也会随时间不断变化。

这个小程序要做的，是通过监控主程序窗口标题的变化，来达到记录应用启动时间的效果。