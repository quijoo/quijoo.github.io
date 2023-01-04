# AI 自瞄 | 分离计算-采集-控制


由于本机需要运行游戏进程，再额外运行神经网络，这样做非常卷，所以使用一台额外的计算机来推理（在网吧直接开两台电脑🤡）
现在有两种数据传输的方案：
### 使用采集卡
能采集高帧率画面的采集卡售价已经超过1000RMB了，所以很不划算
将游戏主机记为 A，推理主机记为 B，该方案，A 只需要运行一个控制鼠标移动的小程序（几乎没有消耗，可以直接纯 Python实现），主机 A 需要捕获采集卡的视频，并且进行推理，将目标列表数据通过 TCP/UDP 传输到主机A。两台主机需要通过**采集卡**和**RJ45**连接

### 进程间通信的方式
优点是成本低，只用连接一根网线，缺点是速度可能会慢）
该方法在主机A运行**屏幕捕获和鼠标更新**进程，在主机B运行推理进程

A->B : 剪裁后的 cv::Mat (320 x 320)
B->A : 鼠标控制数据
以 90 fps的推理速度来计算
$$
data = \frac{90  \times 320 \times 320 \times 24}{8} =  27,648,000(B/s)=27,000(KB/s) = 26.4(MB/s)
$$
所以进程间 以网线连接 使用UDP 通信，需要保证26.4 MB/s 的上传速度，不清楚这会不会导致过度占用网卡，导致游戏进程掉包

本地两台 PC 之间的传输时延低于 1ms， 对于本应用可以忽略不计
我们考虑每一帧的情况
$$
data = \frac{1  \times 320 \times 320 \times 24}{8} =  307,200(B/frame)=300(KB/frame) = 0.293(MB/frame)
$$
传输方法
```c++
// send
 Mat img = imread("img/1.jpg");
int imgSize = img.cols*img.rows*img.channels();
char *pos = (char*)img.data;
int total = 0;
while (total < imgSize)
{
    int sizelen = send(sockfd, pos+total, imgSize-total, 0);
    total = total + sizelen;
}
// recieve
 if ((connfd = accept(listenfd, (struct sockaddr*)NULL, NULL)) == -1)
{
    printf("accept socket error: %s(errno: %d)",strerror(errno),errno);
}
char buf[320 * 320 * 3]; // 图片宽高
memset(buf, 0, sizeof(buf));
int total = 0;
while(total < 320 * 320 * 3)
{
    long len = recv(connfd, buf+total, 320 * 320 * 3-total, 0); // 注意偏移量
    total = total + len;
}
cout << "接收长度为: " << total << endl;
Mat img(320 * 320, CV_8UC3, buf);
```

在开始开发之前需要测试**网络和性能瓶颈**
时延计算
$$
timeDelay = grabTime + cropTime + TransportTime_1 + InferenceTime + TransportTime_2
$$

使用一个ZeroMQ 的头文件only版本就行了（非常方便）
这样就能单机运行和多机运行了（设置启动脚本即可）
tips:
1. cppzmq 只是一个 c++ 的wrapper，其本质还是 基于c 的libzmq
2. 使用cppzmq 需要先编译 libzmq，并且将 libzmq/include/zmq.h 和 cppzmq/* 添加到包含目录
3. 将 libzmq/\<build_dir\>/lib/添加到库目录
4. 将以下文件作为链接器输入
    wsock32.lib
    ws2_32.lib
    Iphlpapi.lib
    libzmq-v143-mt-s-4_3_5.lib
    libzmq-v143-mt-4_3_5.exp
    testutil.lib
    testutil-static.lib
    unity.lib

经过测试，使用 cppzmq 在本地计算机能完成 1ms 内的数据传输，目标数据的传输需要使用 json 进行序列化！
