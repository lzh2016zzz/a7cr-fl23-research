# a7cr-fl23-research：中文入口

[English / 作者原始研究记录](README.md) · **简体中文入口**

本页是中文导航和下游工具更新摘要。原作者关于 A7CR 2.01 原生 FL2/FL3 的偏移、
处理路径、WBI 失效顺序及待验证事项，请阅读 [英文研究正文](README.md)。
这里没有重译或重新授权原作者的全部文档。

## 可复用的下游工具

[通用 agent A7CR 创意外观工具](https://github.com/littleben/a7cr-creative-look-tools)
基于本仓库的 A7CR 发现、injectiver 的研究源码和 QAQLeonard 的 DCP／原生 worker
基础，提供 Python/JSON 项目接口、通用 AGENTS.md，以及完整中英文 README。
固件输入、DCP、逐机备份／校准及照片都由使用者在本机保留。

下游报告的实测范围是一台 A7CR 2.01 / Apple Silicon Mac：原生 FL2/FL3、单槽
Classic Chrome，以及保留首槽后增量添加 PRO Neg. Std、ASTIA。完整 AP/CP、
68 份名称资源、新启动后的实际 AP 区段、保存／原生 ST 和新 JPEG 标记均已核验。
最近的人像外观试片是静物，尚未验证实际肤色、与富士原机的一致性或完整实机恢复。

项目接口分别执行检查点、本地拟合、工件检查、逐机准备、显式安装、只读核验和
精确逆向，不依赖某一家的 agent 客户端。它不代表本研究笔记成为所有机身通用的
刷机程序，也不替代作者原文中的待验证事项。

一次“全部写入并读回完成、等待普通 USB 返回超时”的案例，通过保留原始报告、
新冷启动和只读核验收尾，没有重复刷写。这类已证实完成的退出超时与未知写入结果
不同，不能把任何超时都当成成功，也不能直接重跑安装。

## 来源

- [本仓库原始 A7CR 研究](README.md)
- [QAQLeonard/A7C II 工具](https://github.com/QAQLeonard/a7c2-fl23-research)
- [injectiver/A7CR 研究工具](https://github.com/injectiver/a7cr-creative-look-tools)
- [下游项目署名与许可](https://github.com/littleben/a7cr-creative-look-tools/blob/main/NOTICE.md)
