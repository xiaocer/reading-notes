## 基于TCP的服务器端和客户端
##### 1.TCP原理
1.TCP套接字的I/O缓冲
![image.png](https://upload-images.jianshu.io/upload_images/17728742-f6efcdd55b236a57.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
IO缓冲特性：
```
1. IO缓冲在每个TCP套接字中单独存在
2. IO缓冲在创建套接字时自动生成
3. 即使关闭套接字也会继续传输输出缓冲区中遗留的数据
4. 关闭套接字将丢失输入缓冲区中的数据
```
==套接字位于内核中，所以套接字的IO缓冲也是==
2. TCP内部工作原理1：与对方套接字的连接（三次握手）
3. TCP内部工作原理2：与对方主机的数据交换
4. TCP内部工作原理3：断开与套接字的连接（四次挥手）