---
layout: page
titles:
  en: About
  zh: 关于
  zh-Hans: 关于
  zh-Hant: 關於
key: page-about
---

欢迎来到AirGuanZ的个人主页！

## Projects

### VoxelWorld

VoxelWorld是一个仿照著名体素游戏minecraft的体素世界模拟器。
以DirectX11为图形API（未使用游戏引擎），现支持以下特性：

- 随着摄像机移动动态加载和卸载的无限大世界
- 日夜变换和彩色光源
- 体素环境光遮蔽
- 基于维诺图和柏林噪声的地形生成，支持生物群系
- 角色肢体动作（骨骼动画）

[项目地址](https://github.com/AirGuanZ/VoxelWorld)

### TinyOS

TinyOS是一个以学习为目的创建的、运行在x86单核计算机上的小型操作系统。
其中以下特性由本人设计与实现：

- 分页式内存管理
- 内核线程调度
- 内核进程和用户进程
- 系统调用
- 信号量等基本互斥与同步机制
- 进程消息队列
- 键盘驱动程序

[项目地址](https://github.com/TinyOSOrg/TinyOS)

### AGZParserGen

AGZParserGen是一个基于LR(1)算法的语法分析器生成器。
AGZParserGen基于C++实现，可通过模板参数定制词法单元流的类型和数据，
支持直接从文法脚本构建Parser，也可将构建的Parser导出为不同语言的代码。

[项目地址](https://github.com/AirGuanZ/AGZParserGen)
