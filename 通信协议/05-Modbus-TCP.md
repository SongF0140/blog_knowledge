modbus-tcp使用TCP/IP协议栈实现，所以通讯链路依赖以太网通信。由于tcp/ip协议栈在数据链路层已经存在着帧检验序列所以我们不必继续使用CRC校验码，而是删去并把从站报头改为MBAP报头

![Pasted image 20260901114501.png](./img/Pasted image 20260901114501.png)

如图是标准帧传递的服务端和客户端方案

![Pasted image 20260901115034.png](./img/Pasted image 20260901115034.png)


传递时belike:
![Pasted image 20260901115458.png](./img/Pasted image 20260901115458.png)

这是TCP和rtu的结合：
![Pasted image 20260901115609.png](./img/Pasted image 20260901115609.png)