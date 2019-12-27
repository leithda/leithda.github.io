---
title: Java-网络编程-01基础
abbrlink: 2098644504
date: 2019-12-27 11:12:59
categories:
  - Java
  - Basic
tags:
  - Net
  - 网络编程
author: 长歌
---

{% cq %}
计算机网络是通过传输介质、通信设施和网络通信协议，把分散在不同地点的计算机设备互连起来的，实现资源共享和数据传输的系统。网络编程就是编写程序使互联网的两个（或多个）设备（如计算机）之间进行数据传输。Java语言对网络编程提供了良好的支持。
{% endcq %}
<!-- More -->

## IP
> Java中使用`InetAddress`对象来表示IP

```java
    /**
     * InetAddress 表示 IP
     */
    public void test() {
        InetAddress ip;
        try {
//            ip = InetAddress.getByName("www.baidu.com");  // 根据域名获取IP
//            ip = InetAddress.getLoopbackAddress();  // 本地回环IP   127.0.0.1
//            ip = InetAddress.getLocalHost();    // 本地 IP  127.0.1.1
            ip = InetAddress.getByAddress(new byte[]{39,-100,66,18}); // 根据byte IP【39,156,66,18】 获取IP
            System.out.println("addr:" + ip.getHostAddress());
            System.out.println("name:" + ip.getHostName());
            byte[] bytes = ip.getAddress();
            for (int i = bytes.length - 1; i >= 0; i--) { // 由于计算机表示IP时，地位表示高位,会导致阅读顺序同正常相反
                System.out.println(bytes[i]);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

## Socket

操作系统为网络服务提供的一组接口，通信的两端都有`Socket`,网络通信其实就是`Socket`间的通信，数据在两个`Socket`间通过`IO`传输


## UDP传输

> Java将UPP传输封装为对象`DatagreamSocket`,这个对象中封装了`upd`传输协议的`Socket`对象。由于数据包中包含的信息较多，将数据包封装为`DatagramPacket`，通过这个对象中的方法。就可以获取到数据包中的各种信息

```java
public class UdpTest {

    class UdpSender implements Runnable{

        @Override
        public void run() {
            try {
                DatagramSocket ds = new DatagramSocket(8888);
                String text = "leithda";
                byte[] buf = text.getBytes();
                DatagramPacket dp = new DatagramPacket(buf,buf.length, InetAddress.getLocalHost(),10001);
                ds.send(dp);
            }catch (Exception e){
                System.out.println("发送udp报文异常"+e);
            }
        }
    }

    class UdpReceiver implements Runnable{
        @Override
        public void run() {
            try{
                DatagramSocket ds = new DatagramSocket(10001);

                byte[] buf = new byte[1024];
                DatagramPacket dp = new DatagramPacket(buf, buf.length);
                ds.receive(dp);

                String ip = dp.getAddress().getHostAddress();
                int port = dp.getPort();
                String text = new String(dp.getData(),0,dp.getLength());
                System.out.println("收到IP["+ip+"],port["+port+"]的内容"+text);
                ds.close();
            }catch (Exception e){
                System.out.println("接收udp报文异常");
            }
        }
    }

    public static void main(String[] args) {
        UdpSender udpSender = new UdpTest().new UdpSender();
        UdpReceiver udpReceiver = new UdpTest().new UdpReceiver();
        new Thread(udpReceiver).start();
        new Thread(udpSender).start();
    }
}
```

## TCP传输

> 两个端点建立连接后会有一个传输数据的通道，这通道称为socket流。该流可读可写.

```java
public class TcpTest {


    class TcpSender implements Runnable{

        @Override
        public void run() {
            Socket socket;
            try{
                socket = new Socket("127.0.0.1",10002);
                OutputStream out = socket.getOutputStream();
                out.write("来自tcp的消息～".getBytes());
                socket.close();
            }catch (Exception e){
                System.out.println("Tcp发送数据失败");
            }

        }
    }

    class TcpReceiver implements Runnable{

        @Override
        public void run() {
            ServerSocket serverSocket;
            try{
                serverSocket = new ServerSocket(10002);
                Socket socket = serverSocket.accept();
                String ip = socket.getInetAddress().getHostAddress();
                System.out.println("ip : "+ip+" 已经连接");

                ExecutorService executorService = Executors.newCachedThreadPool();
                executorService.execute(new SingleServer(socket));

            }catch (Exception e){
                System.out.println("Tcp接收数据失败");
            }
        }
    }

    class SingleServer implements Runnable{

        private Socket socket;

        public SingleServer(Socket socket) {
            this.socket = socket;
        }

        @Override
        public void run() {
            if(socket != null) {
                try{
                    InputStream in = socket.getInputStream();
                    byte[] bytes = new byte[1024];
                    int len = in.read(bytes);
                    System.out.println("Tcp服务端收到客户端消息:["+new String(bytes,0,len)+"]");
                }catch (Exception e){
                    System.out.println("Tcp 接收线程处理失败");
                }finally {
                    System.out.println("Tcp服务端与客户端通讯结束");
                    try{
                        socket.close();
                    }catch (Exception ex){
                        ex.printStackTrace();
                    }
                }

            }
        }
    }

    public static void main(String[] args) {
        TcpReceiver tcpReceiver = new TcpTest().new TcpReceiver();
        TcpSender tcpSender = new TcpTest().new TcpSender();

        new Thread(tcpReceiver).start();
        new Thread(tcpSender).start();
    }
}
```

- 上述代码，服务端程序未关闭。