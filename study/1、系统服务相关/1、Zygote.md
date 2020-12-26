# 谈谈对Zygote理解

### 一、Zygote的作用

###### 1、启动SystemServer（单独跑在独立的进程中）

###### 2、孵化应用进程



### 二、Zygote的启动流程

###### 1、安卓进程启动套路:

>1、进程启动
>
>2、准备工作
>
>3、Loop循环：不停接收消息，处理消息。
>
>> 消息可能来源：可能Socket发送过来的，可能MessageQueue中的，可能Binder驱动中的消息。等等。
>
>ps：启动三段式不止Zygote进程满足这个套路，只要是有独立进程的（如系统服务进程，自己的应用进程）都满足这个启动套路。

###### 2、Zygote的启动流程

> Zygote的启动可以分为两段,Zygote的进程是如何启动的? Zygote 进程启动后做了些什么事？

- Zygote的进程是如何启动的? 

  安卓系统启动会加载Linux kernel，Linux kernel加载完成后启用用户空间的第一个进程init进程。init进程中读取init.rc 配置文件，  这个配置文件中定义了要启动的一些系统进程（Zygote就是其中一个）。

  进程启动主要通过fork+handle 或者 fork +execve 系统调用，系统调用时把配置文件中进程相关的参数传递过来即可开启相应进程。如下代码zygote表示进程名，/system/bin/app_process可执行程序所在路径，-Xzygote /system/bin --zygote --start-system-server 表示参数。

  ```java
  //init.zygote32.rc
  service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
      class main
      priority -20
      user root
      group root readproc reserved_disk
      socket zygote stream 660 root system
      onrestart write /sys/android_power/request_state wake
      onrestart write /sys/power/state on
      onrestart restart audioserver
      onrestart restart cameraserver
      onrestart restart media
      onrestart restart netd
      onrestart restart wificond
      writepid /dev/cpuset/foreground/tasks
  ```

  fork 函数就是创建子进程的，fork函数会返回两次，子进程中返回子进程id，父进程中返回0。默认情况下子进程继承父进程所有资源，通过execve（path，argv，evn）执行系统调用加载新的二进制文件时继承的父类资源会被清掉。采用新的二进制的资源。

  信号处理（SIGCHLD）：如父进程-->fork-->子进程 。当子进程挂了父进程会收到SIGCHLD信号，然后可重建子进程。

- Zygote进程启动后做了些什么事？

  init 进程读取配置文件创建了Zygote进程，这时Zygote被创建后开始做准备工作，首先执行exeve系统调用打开Java世界，Zygote中主要做了三件事，即：执行系统调用启动安卓虚拟机-->注册安卓JNI函数-->进入Java 世界。

  Zygote java 世界主要做了三件事：预加载资源-->启动SystemSever-->进入Loop循环（等待Socket消息。）

  ​

  普通的c++ 调用java的代码示例如下：

  ![img]()

  ​

  我们自己的应用进程直接可以进行JNI调用，因为我们的进程是由Zygote进程孵化的，继承了Zygote进程创建时创建的java虚拟机。使用时直接重置下虚拟机的状态即可使用。



细节：

Zygote fork时要单进程

Zygote 的跨进程未采用Binder机制，而是采用的Socket。

### 三、Zygote的工作原理



四、问题思考

###### 1、孵化应用进程这种事为啥不交给SystemSever来做？而专门设计了Zygote

###### 2、Zygote的IPC机制为啥不采用Binder，如果采用Binder 会有什么问题？



