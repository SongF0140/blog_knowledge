# 个人技术知识库

日常学习笔记的沉淀仓库，用 Obsidian 管理，随学随记。内容以「概念讲清楚 + 代码可复现」为目标，覆盖嵌入式、通信协议、Qt、数据库、Git、前端等方向。

## 目录结构

| 目录 | 内容 | 篇数 |
| --- | --- | --- |
| [通信协议](./通信协议) | 从物理层到总线协议：TCP/IP、串口、RS-232/485、Modbus、I2C、SPI、CAN | 8 |
| [嵌入式笔记](./嵌入式笔记) | ESP32 开发：GPIO、UART、I2C、SPI、ADC、定时器、FreeRTOS、WiFi/BLE、摄像头等 | 28 |
| [QT学习笔记](./QT学习笔记) | Qt 入门到进阶：CMake 工程、控件、信号槽、事件系统、多线程、绘图、SQLite | 22 |
| [MYSQL笔记](./MYSQL笔记) | 数据库与表、增删改查、联表查询、分组与子查询 | 9 |
| [GIT笔记](./GIT笔记) | 基础操作、分支管理、撤销回退、远程与标签、团队协作 | 5 |
| [前端碎碎念](./前端碎碎念) | HTML/CSS、JS、Vue2、Vue3、React | 5 |
| [CSDN博文备份](./CSDN博文备份) | 早期博客归档：数据结构、算法题解、神经网络、计算机组成原理 | 20 |

## 通信协议

- [01-从双绞线到TCPIP通信](./通信协议/01-从双绞线到TCPIP通信.md)
- [02-串口通信](./通信协议/02-串口通信.md)
- [03-RS232与RS485](./通信协议/03-RS232与RS485.md)
- [04-Modbus-RTU](./通信协议/04-Modbus-RTU.md)
- [05-Modbus-TCP](./通信协议/05-Modbus-TCP.md)
- [06-I2C](./通信协议/06-I2C.md)
- [07-SPI](./通信协议/07-SPI.md)
- [08-CAN](./通信协议/08-CAN.md)

## 快速入口

- [嵌入式知识地图](./嵌入式笔记/00-knowledge-map.md)
- [Qt 知识地图](./QT学习笔记/00-qt-knowledge-map.md)

## 使用方式

```bash
git clone https://github.com/SongF0140/blog_knowledge.git
```

笔记使用 Markdown 编写，直接用 Obsidian 打开仓库根目录即可获得双链与图谱视图；图片统一存放在各分类目录的 `img/` 下。

## 说明

笔记为个人学习过程中的理解与整理，部分内容参考了公开资料与课程，如发现错误欢迎提 Issue 指正。

## License

详见 [LICENSE](./LICENSE)。
