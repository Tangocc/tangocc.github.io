---
layout:     post
title:      "分布式服务框架之原理实现"
subtitle:   ""
date:       2017-10-26 12:00:00
author:     "Tango"
header-img: "img/in-post/post-2017-10-11/post-bg-universe.jpg"
catalog: true
tags:   
    - 分布式
    - RPC
    - 分布式服务框架
---
> RPC的全称为**Remote Procedure Call**， 他是一种进程间通信放hi，允许向调用本地方法一样调用远程服务，对于上层应用来说透明化，屏蔽服务调用过程。目前业界由许多开源框架，例如
>> **Apache Thrift**(Facebook开源)　　
>> 
>> **Avro-RPC**(Hadoop子项目)　　
>> 
>> **Hessian**(caucho提供的基于binary-RPC)　　
>> 
>> **gRPC**(google开源)  

## 原理
 虽然各种开源框架实现细节不同，但其基本原理如下所示：
 <center>
![](/img/in-post/post-2017-10-11/post-rpc-structs.jpg)
</center>  

 1. 服务端服务发布：
 
  >  服务提供者根据配置自动连接服务注册中心地址，并通过xml配置文件将服务发布到注册中心，其中包括IP地址/端口号/以及服务名称/协议/版本号

 2. 消费端服务获取
  > 服务信息获取消费者根据配置自动连接服务注册中心地址，根据服务引用信息获取指定服务的地址等路由信息

 3. 服务注册中心推送

 > 服务注册中西根据服务订阅关系，动态向制定消费者推送服务地址信息

 4. 消费者调用服务

 >  消费者根据服务名称以及IP/端口等信息发起远程调用

 5. 序列化

 >   将请求参数以及请求方法等信息进行序列化

 6. 反序列化

  >  将序列化信息反序列化为对象

 7. 返回结果

 > 服务端根据消费端传来的参数调用本地服务，并将处理结果返回。


---
## 原理实现

### 注册中心
```java
package register;

import services.bean.ServiceInfo;

import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.Executor;
import java.util.concurrent.Executors;

/**
 * 服务注册中心
 * * 服务提供方注册服务消息
 * * 消费方拉取注册的服务信息
 */
public class RegisterCenter {


    private static final String REGISTER_IP = "127.0.0.1";
    //注册服务监听端口
    private static final int REGISTER_PORT = 4080;
    //拉取服务信息监听端口
    private static final int PULL_PORT = 4081;
    //注册服务处理线程
    private static Executor registerExecutor = Executors.newFixedThreadPool(10);
    //拉取服务信息处理线程
    private static Executor pullExecutor = Executors.newFixedThreadPool(10);
    //注册服务信息
    private static final Map<String, ServiceInfo> serviceMaps = new ConcurrentHashMap<>();

    public void init() {

        System.out.println("----------注册中心启动---------");

        //开启服务端注册服务监听线程
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("------------注册服务监听线程启动----------------");
                while (true) {
                    ServerSocket socket = null;
                    try {

                        socket = new ServerSocket();
                        socket.bind(new InetSocketAddress(REGISTER_IP, REGISTER_PORT));
                        while (true) {
                            registerExecutor.execute(new RegisterServiceTask(socket.accept()));
                        }
                    } catch (Exception e) {
                        System.out.println("RegisterCenter：" + e.getMessage());
                    }

                }

            }
        }).start();

        try {
            Thread.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        new Thread(new Runnable() {
            @Override
            public void run() {
                //主线程开启消费端拉取服务信息监听
                System.out.println("------------客户端服务请求监听线程启动----------------");
                ServerSocket socket = null;
                try {

                    socket = new ServerSocket();
                    socket.bind(new InetSocketAddress(REGISTER_IP, PULL_PORT));
                    while (true) {
                        pullExecutor.execute(new PullServiceTask(socket.accept()));
                    }
                } catch (Exception e) {
                    System.out.println("RegisterCenter：" + e.getMessage());
                } finally {
                    System.out.println("-------------注册中心停止-----------------");
                }
            }
        }).start();

    }


    //向消费端推送服务注册信息
    private static class PullServiceTask implements Runnable {

        private Socket socket;

        PullServiceTask(Socket socket) {
            this.socket = socket;
        }

        @Override
        public void run() {
            System.out.println("---------消费端服务获取线程启动------------");
            ObjectInputStream objectInputStream = null;
            ObjectOutputStream objectOutputStream = null;

            try {
                objectInputStream = new ObjectInputStream(socket.getInputStream());
                objectOutputStream = new ObjectOutputStream(socket.getOutputStream());

                String serviceName = (String) objectInputStream.readObject();
                if (!serviceMaps.containsKey(serviceName)) {
                    objectOutputStream.writeObject(null);
                    return;
                }
                ServiceInfo serviceInfo = serviceMaps.get(serviceName);
                objectOutputStream.writeObject(serviceInfo);
            } catch (Exception e) {
                System.out.println("PullServiceTask：" + e.getMessage());
            } finally {

                try {
                    if (socket != null) {
                        socket.close();
                        socket = null;
                    }
                    if (objectInputStream != null) {
                        objectInputStream.close();
                    }
                    if (objectOutputStream != null) {
                        objectOutputStream.close();
                    }
                } catch (Exception e) {
                    System.out.println("PullServiceTask：" + e.getMessage());
                }

            }
            System.out.println("---------消费端拉取服务信息线程停止------------");
        }
    }

    //服务提供方注册服务
    private static class RegisterServiceTask implements Runnable {

        private Socket socket;

        RegisterServiceTask(Socket socket) {
            this.socket = socket;
        }

        @Override
        public void run() {
            System.out.println("---------服务端注册服务线程启动------------");
            ObjectInputStream objectInputStream = null;
            ObjectOutputStream objectOutputStream = null;

            try {
                objectInputStream = new ObjectInputStream(socket.getInputStream());
                ServiceInfo serviceInfo = (ServiceInfo) objectInputStream.readObject();
                String serviceName = serviceInfo.getName();
                serviceMaps.put(serviceName, serviceInfo);
                System.out.println("注册服务：" + serviceInfo.toString());
            } catch (Exception e) {
                System.out.println("RegisterServiceTask：" + e.getMessage());
            } finally {

                try {
                    if (socket != null) {
                        socket.close();
                        socket = null;
                    }
                    if (objectInputStream != null) {
                        objectInputStream.close();
                    }
                    if (objectOutputStream != null) {
                        objectOutputStream.close();
                    }
                } catch (Exception e) {
                    System.out.println("RegisterServiceTask：" + e.getMessage());
                }

            }
            System.out.println("---------服务端注册服务信息线程停止------------");
        }
    }

}

```
### 服务端
```java
package server;

import services.bean.ServiceInfo;

import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Method;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;
import java.net.SocketAddress;
import java.util.concurrent.Executor;
import java.util.concurrent.Executors;

public class RpcServer {

    private static final String REGISTER_CENTER_IP = "127.0.0.1";

    private static final int REGISTER_CENTER_PORT = 4080;

    private static Executor serviceExecutor = Executors.newFixedThreadPool(10);

    //注册中心地址
    private static SocketAddress socketAddress = new InetSocketAddress(REGISTER_CENTER_IP, REGISTER_CENTER_PORT);

    //注册服务
    private void registerService(String ip, int port, String serviceName) {

        //连接注册中心
        Socket socket = new Socket();
        ObjectInputStream objectInputStream = null;
        ObjectOutputStream objectOutputStream = null;

        try {
            socket.connect(socketAddress);
            objectOutputStream = new ObjectOutputStream(socket.getOutputStream());
            ServiceInfo serviceInfo = new ServiceInfo();
            serviceInfo.setIp(ip);
            serviceInfo.setName(serviceName);
            serviceInfo.setPort(port);
            objectOutputStream.writeObject(serviceInfo);

        } catch (IOException e) {
            System.out.println(e.getMessage());
        } finally {

            try {
                if (socket != null) {
                    socket.close();
                    socket = null;
                }
                if (objectOutputStream != null) {
                    objectOutputStream.close();
                }
                if (objectInputStream != null) {
                    objectInputStream.close();
                }

            } catch (IOException e) {
                System.out.println("RpcServer: " + e.getMessage());
            }

        }

    }

    public void publishService(String ip, int port, String name) {

        //注册中心注册服务
        registerService(ip, port, name);

        //创建服务监听服务端口
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("----------服务端提供服务-----------");
                InetSocketAddress socketAddress = new InetSocketAddress("127.0.0.1", port);

                try {
                    ServerSocket socket = new ServerSocket();
                    socket.bind(socketAddress);

                    while (true) {
                        serviceExecutor.execute(new ServiceTask(socket.accept()));
                    }

                } catch (IOException e) {
                    System.out.println(e.getMessage());
                }
            }
        }).start();

    }

    //执行service 并将结果返回
    private class ServiceTask implements Runnable {
        private Socket socket;

        ServiceTask(Socket socket) {
            this.socket = socket;
        }

        @Override
        public void run() {
            System.out.println("-----------本地执行服务，并返回结果------------");
            try {
                ObjectInputStream objectInputStream = new ObjectInputStream(socket.getInputStream());
                String serviceName = (String) objectInputStream.readObject();
                String methodName = (String) objectInputStream.readObject();
                Class<?>[] paramType = (Class<?>[]) objectInputStream.readObject();
                Object[] args = (Object[]) objectInputStream.readObject();
                System.out.println(serviceName + " " + methodName + " " + paramType);
                Class service = Class.forName(serviceName);
                Object obj = service.newInstance();
                Method method = service.getDeclaredMethod(methodName, paramType);
                Object resObject = method.invoke(obj, args);
                ObjectOutputStream objectOutputStream = new ObjectOutputStream(socket.getOutputStream());
                System.out.println("执行结果：" + resObject);
                objectOutputStream.writeObject(resObject);

            } catch (Exception e) {
                System.out.println("ServiceTask:" + e.getMessage());
            } finally {
                try {
                    if (socket != null) {
                        socket.close();
                    }
                } catch (IOException e) {
                    System.out.println("ServiceTask:" + e.getMessage());
                }
            }

        }
    }
}

```
### 客户端
```java
package client;

import services.bean.ServiceInfo;

import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.net.InetSocketAddress;
import java.net.Socket;
import java.net.SocketAddress;
import java.util.HashMap;
import java.util.Map;

public class RpcClient<T> {

    private static final String REGISTER_IP = "127.0.0.1";
    private static final int REGISTER_PORT = 4081;
    private static final SocketAddress REGISTER_SOCKETADDRESS = new InetSocketAddress(REGISTER_IP, REGISTER_PORT);
    private static final Map<String, ServiceInfo> serviceMaps = new HashMap<>();

    //从注册中心拉去服务信息并缓存
    private void pullService(String serviceName) {

        System.out.println("----------从注册中心拉取服务信息----------");
        Socket socket = new Socket();
        ServiceInfo obj = null;
        ObjectInputStream objectInputStream = null;
        ObjectOutputStream objectOutputStream = null;
        try {
            socket.connect(REGISTER_SOCKETADDRESS);
            objectOutputStream = new ObjectOutputStream(socket.getOutputStream());
            objectOutputStream.writeObject(serviceName);
            Thread.sleep(10);
            objectInputStream = new ObjectInputStream(socket.getInputStream());
            obj = (ServiceInfo) objectInputStream.readObject();
            if (obj == null) {
                return;
            }
            serviceMaps.put(serviceName, obj);

            System.out.println("----------从注册中心拉取服务信息完成----------");
        } catch (Exception e) {
            System.out.println("RpcClient:" + e.getMessage());
        } finally {

            try {
                if (socket != null) {
                    socket.close();
                    socket = null;
                }
                if (objectInputStream != null) {
                    objectInputStream.close();
                }
            } catch (Exception e) {
                System.out.println("RpcClient:" + e.getMessage());
            }


        }


    }

    public Object importer(final Class<?> serviceClass, String service) {

        return Proxy.newProxyInstance(serviceClass.getClassLoader(), serviceClass.getInterfaces(), new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

                Socket socket = null;
                ObjectInputStream objectInputStream = null;
                ObjectOutputStream objectOutputStream = null;

                try {

                    if (!serviceMaps.containsKey(service)) {
                        pullService(service);
                    }

                    if (!serviceMaps.containsKey(service)) {
                        return null;
                    }
                    ServiceInfo serviceInfo = (ServiceInfo) serviceMaps.get(service);
                    socket = new Socket();
                    socket.connect(new InetSocketAddress(serviceInfo.getIp(), serviceInfo.getPort()));

                    if (!socket.isConnected()) {
                        //没有连接  则返回null
                        return null;
                    }

                    objectOutputStream = new ObjectOutputStream(socket.getOutputStream());
                    objectOutputStream.writeObject(serviceClass.getName());
                    objectOutputStream.writeObject(method.getName());
                    objectOutputStream.writeObject(method.getParameterTypes());
                    objectOutputStream.writeObject(args);

                    objectOutputStream.flush();
                    Thread.sleep(10);
                    objectInputStream = new ObjectInputStream(socket.getInputStream());
                    return objectInputStream.readObject();
                } catch (Exception e) {
                    System.out.println("RpcClient:"+e.getMessage());

                } finally {
                    if (socket != null) {
                        socket.close();
                    }
                    if (objectInputStream != null) {
                        objectInputStream.close();
                    }
                    if (objectInputStream != null) {
                        objectInputStream.close();
                    }

                }
                return null;
            }
        });
    }


}

```
### 服务
```java

public interface HelloWorldService {

    String say();
}


public class HelloWorldServiceImpl implements HelloWorldService {

    @Override
    public String say() {
        return "Hello World";
    }
}

```

### 测试用例
```java
package test;

import client.RpcClient;
import register.RegisterCenter;
import server.RpcServer;
import services.impl.HelloWorldServiceImpl;
import services.service.HelloWorldService;

public class HelloWorldTest {

    public static void main(String[] args) throws InterruptedException {

        //启动注册中心
        RegisterCenter registerCenter = new RegisterCenter();
        registerCenter.init();
        Thread.sleep(1000);

        //启动服务端，并发布服务
        RpcServer rpcServer = new RpcServer();
        rpcServer.publishService("127.0.0.1",5000,"helloworld.service");

        Thread.sleep(1000);
        RpcClient<HelloWorldService> rpcClient = new RpcClient<>();
        HelloWorldService helloWorldService = (HelloWorldService) rpcClient.importer(HelloWorldServiceImpl.class,"helloworld.service");
        String result =   helloWorldService.say();
        System.out.println(result);

        Thread.sleep(1000);
        System.out.println("---------Test Finished-----------");
    }
}

```