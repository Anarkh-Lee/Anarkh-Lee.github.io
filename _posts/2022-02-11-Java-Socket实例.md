---
layout: post
title: 'Java Socket实例'
date: 2022-02-11
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: 网络传输





---

> 最近用到了Socket，遂总结一下例子。

### 1.实战过程的Socket客户端例子

```java
import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.OutputStreamWriter;
import java.net.Socket;

public class SocketUtils {
	
	public static final int port = 12999;   
    public static final String host = "127.0.0.1";

	
	public String socket(String str1){
		String retxml="";
        Socket socket = null;  
        try {
            //创建一个流套接字并将其连接到指定主机上的指定端口号  
        	socket = new Socket(host, port); 
        	//这里服务器端使用GBK编码，遂也要用GBK，否则乱码
			BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream(),"GBK"));
			System.out.println("strsocketXXX=["+str1+"]"); 
			bw.write(str1);
			bw.flush();
			BufferedReader br = new BufferedReader(new InputStreamReader(socket.getInputStream(),"GBK"));
			String s = "";  
			
			while((s = br.readLine()) != null){
				System.out.println(s); 
				retxml=retxml+s;
			}
			System.out.println("retxml="+retxml);
			br.close();  
			bw.close();
 
        } catch (Exception e) { 
            System.out.println("客户端异常:" + e.getMessage());   
        } finally {
            if (socket != null) {  
                try {  
                    socket.close();  
                    System.out.println("客户端 正常关闭啦");  
                } catch (IOException e) {  
                    socket = null;   
                    System.out.println("客户端 finally 异常:" + e.getMessage());   
                }  
            }  
        }  
		
        return retxml;
	}

}

```

### 2.完整的java socket服务器端和客户端示例

#### 2.1 服务器端

```java
package socket;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.ServerSocket;
import java.net.Socket;

public class SocketService {
    //搭建服务器端
    public static void main(String[] args) throws IOException{
        SocketService socketService = new SocketService();
        //1、a)创建一个服务器端Socket，即SocketService 
        socketService.oneServer();
    }
    public  void oneServer(){
        try{
            ServerSocket server=null;
            try{
                server=new ServerSocket(5209);
                //b)指定绑定的端口，并监听此端口。
                System.out.println("服务器启动成功");
                //创建一个ServerSocket在端口5209监听客户请求
            }catch(Exception e) {
                    System.out.println("没有启动监听："+e);
                    //出错，打印出错信息
            }
            Socket socket=null;
            try{
                socket=server.accept();
                //2、调用accept()方法开始监听，等待客户端的连接 
                //使用accept()阻塞等待客户请求，有客户
                //请求到来则产生一个Socket对象，并继续执行
            }catch(Exception e) {
                System.out.println("Error."+e);
                //出错，打印出错信息
            }
            //3、获取输入流，并读取客户端信息 
            String line;
            BufferedReader in=new BufferedReader(new InputStreamReader(socket.getInputStream()));
            //由Socket对象得到输入流，并构造相应的BufferedReader对象
            PrintWriter writer=new PrintWriter(socket.getOutputStream());
            //由Socket对象得到输出流，并构造PrintWriter对象
            BufferedReader br=new BufferedReader(new InputStreamReader(System.in));
            //由系统标准输入设备构造BufferedReader对象
            System.out.println("Client:"+in.readLine());
            //在标准输出上打印从客户端读入的字符串
            line=br.readLine();
            //从标准输入读入一字符串
            //4、获取输出流，响应客户端的请求 
            while(!line.equals("end")){
            //如果该字符串为 "bye"，则停止循环
                writer.println(line);
                //向客户端输出该字符串
                writer.flush();
                //刷新输出流，使Client马上收到该字符串
                System.out.println("Server:"+line);
                //在系统标准输出上打印读入的字符串
                System.out.println("Client:"+in.readLine());
                //从Client读入一字符串，并打印到标准输出上
                line=br.readLine();
                //从系统标准输入读入一字符串
            } //继续循环

            //5、关闭资源 
            writer.close(); //关闭Socket输出流
            in.close(); //关闭Socket输入流
            socket.close(); //关闭Socket
            server.close(); //关闭ServerSocket
        }catch(Exception e) {//出错，打印出错信息
            System.out.println("Error."+e);
        }
    }
}
```

#### 2.2 客户端

```java
package socket;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.InetAddress;
import java.net.Socket;
import java.net.URL;

public class SocketClient {
    // 搭建客户端
    public static void main(String[] args) throws IOException {
        try {
            // 1、创建客户端Socket，指定服务器地址和端口
            // Socket socket=new Socket("127.0.0.1",5200);
            Socket socket = new Socket("192.168.1.115", 5209);
            System.out.println("客户端启动成功");
            // 2、获取输出流，向服务器端发送信息
            // 向本机的52000端口发出客户请求
            BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
            // 由系统标准输入设备构造BufferedReader对象
            PrintWriter write = new PrintWriter(socket.getOutputStream());
            // 由Socket对象得到输出流，并构造PrintWriter对象
            //3、获取输入流，并读取服务器端的响应信息 
            BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            // 由Socket对象得到输入流，并构造相应的BufferedReader对象
            String readline;
            readline = br.readLine(); // 从系统标准输入读入一字符串
            while (!readline.equals("end")) {
                // 若从标准输入读入的字符串为 "end"则停止循环
                write.println(readline);
                // 将从系统标准输入读入的字符串输出到Server
                write.flush();
                // 刷新输出流，使Server马上收到该字符串
                System.out.println("Client:" + readline);
                // 在系统标准输出上打印读入的字符串
                System.out.println("Server:" + in.readLine());
                // 从Server读入一字符串，并打印到标准输出上
                readline = br.readLine(); // 从系统标准输入读入一字符串
            } // 继续循环
            //4、关闭资源 
            write.close(); // 关闭Socket输出流
            in.close(); // 关闭Socket输入流
            socket.close(); // 关闭Socket
        } catch (Exception e) {
            System.out.println("can not listen to:" + e);// 出错，打印出错信息
        }
    }

}
```

#### 2.3 测试结果

![](.\img\网络传输\1.png)

![](.\img\网络传输\2.png)



<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>