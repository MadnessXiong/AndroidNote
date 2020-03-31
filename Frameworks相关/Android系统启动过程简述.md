### init进程启动过程：

init进程是android系统中用户空间的第一个进程，进程号为1。它负责很多重要的重做职责，如创建Zygote和属性服务。init进程是由多个源文件共同组成，这些文件位于源码目录system/core/init。

init进程做了很多工作，总的来说主要做了以下3件事情：

- 创建和挂载启动所需的文件目录
- 初始化和启动属性服务
- 解析init.rc配置文件并启动Zygote进程

### Zygote进程启动过程：

在Android系统中，DVM和ART，应用程序进程以及运行系统的关键服务的SystemServer进程都是由Zygote进程来创建。它被称为孵化器。它通过fork(复制进程)的形式来创建应用程序进程和SystemServer进程，由于Zygote进程在启动过程中会创建DVM或ART虚拟机，因此通过fock创建的应用程序进程和SystemServer进程可以在内部获得一个DVM或ART的实例副本。

Zygote进程启动共做了以下几件事：

- 创建AppRuntime并调用其start方法，启用Zygote进程
- 创建java虚拟机并为Java虚拟机注册JNI方法
- 通过JNI调用ZygoteInit的Main函数进入Zygote的Java框架层。Zygoteinit主要做了4件事情：
- 通过registerZygoteSocket方法创建服务器端Socket，并通过runSelectLoop方法等待AMS的请求来创建新的应用程序进程
- 启动SystemServer进程

### SystemServer处理过程：

SystemServer进程主要用于创建系统服务，AMS，WMS和PMS都是由它创建的。

- ZygoteInit的handleSystemServerProcess()中创建了PathClassLoader，启动了Binder线程池，然后进入了SystemServer的main()
- 在main()中创建了SystemManager，用于对系统的服务进行创建，启动和生命周期进行管理。
- 在main()中启动了各种服务，如AMS，PMS等

### Launcher：

系统启动的最后一步是启动一个应用程序来显示系统中已经安装的应用程序，这个程序就是Launcher。Launcher在启动过程中会请求PackageManagerService返回系统中已经安装的应用程序信息，并将这些信息封装成一个快捷图标列表显示在系统屏幕上，这样用户可以通过点击这些快捷图标来启动相应的应用程序

Launcher就是Android系统的桌面，它的主要作用有2点，1是作为系统的启动器，用于启动应用程序；2是作为Android系统的桌面，用于显示和管理应用程序的快捷图标或者其他桌面组件。

**Launcher启动过程：**

SystemServer进程在启动过程中会启动PackageManagerService，PackageManagerService启动后会将系统中的应用程序安装完成。在此前已经启动的AMS会将Launcher启动起来。

启动Launcher的入口为AMS的systemReady(),最终会调用到ActivityStarter的startHomeActivityLocked()完成Luncher的启动。