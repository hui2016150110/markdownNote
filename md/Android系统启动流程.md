#### Android系统启动流程

init进程的启动总共包括三件主要的事：

1. 穿件和挂载所需的文件目录。

2. 初始化和启动属性服务
3. 解析init.rc配置文件并启动zygote进程（从启动zygote进程后，也就是调用了`zygoteInit`的main方法之后，就进入了java层的代码）

Zygote进程启动总结：

1. 创建`AppRuntime`并调用其start方法，启动zygote进程
2. 创建java虚拟机并为java虚拟机注册JNI方法
3. 通过JNI调用`zygoteInit`的main函数进入Zygote的Java框架层
4. 通过`registerZygoteSocket`方法创建服务端的Socket，并通过`runSelectLoop`方法等待AMS的请求来创建新的应用程序进程
5. 启动`SystemServer`进程

`SystemServer`进程总结：

1. 启动Binder线程池，这样就可以和其他线程通信了。
2. 创建`SystemServiceManager`，它的作用是管理系统的服务的创建，启动和生命周期的管理。
3. 启动各种系统服务

`Launcher`启动过程：
概念：`Launcher`是Android系统的桌面，主要用于启动应用程序，显示和管理应用程序的快捷图标或者其他桌面组件。`Launcher`是一个Activity，系统最终也是通过Intent的方式去启动它的。它显示桌面图标也是通过`recyclerView`的方式来显示的。

Android系统启动流程总结：

1. 启动电源以及系统启动
2. 引导程序`BootLoader`
3. Linux内核的启动
4. init进程启动
5. `zygote`进程的启动
6. `SystemServer`进程的启动
7. `Launcher`的启动

