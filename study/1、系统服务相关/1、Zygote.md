# 谈谈对Zygote理解

### 一、Zygote的作用

###### 1、启动SystemServer进程
     是zygote fork的第一个进程,其中WindowManagerService，ActivityManagerService等重要的可以binder通信的服务都运行在这个SystemServer进程。而像WindowManagerService，ActivityManagerService这样重要，繁忙的服务，是运行在单独线程中，而有些没有繁重的服务，并没有单独开一个线程，有些服务会注册Receiver。
​     SystemServer需要用到一些Zygote准备好的一些系统资源，如常用类、JNI函数、主题资源、共享库。这些资源直接从Zygote继承过来就不用自己重新加载了，提升性能。

###### 2、孵化应用进程（apk1进程、apk2进程、等等）

######      应用相关的进程都是从这个进程中孵化出来的。



### 二、Zygote的启动流程

###### 1、安卓进程启动常用套路总结

>1、进程启动
>
>2、准备工作
>
>3、Loop循环：不停接收消息，处理消息。
>
>> 消息可能来源：可能Socket发送过来的，可能MessageQueue中的，可能Binder驱动中的消息。等等。
>
>ps：启动三段式不止Zygote进程满足这个套路，只要是有独立进程的，如系统服务进程，自己的应用进程都满足这个启动套路。

###### 2、Zygote的启动流程

> Zygote的启动可以分为两块,Zygote的进程是如何启动的? Zygote 进程启动后做了些什么事？

- Zygote的进程是如何启动的? 

  安卓系统启动会加载Linux kernel，Linux kernel加载完成后会启用用户空间的第一个进程init进程。init进程中读取init.rc 配置文件，  这个配置文件中定义了要启动的一些系统服务，Zygote、ServiceManager等就是其中之一（系统服务我们可以粗略地认为系统进程。）。

  进程启动主要通过fork+handle 或者 fork +execve 系统调用。系统调用时把配置文件中进程相关的参数传递过来即可开启相应进程。如下第一行代码，zygote 表示进程名，/system/bin/app_process可执行程序所在路径，-Xzygote /system/bin --zygote --start-system-server 表示相关参数。

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

  fork 函数就是创建子进程的，fork函数会返回两次，子进程中返回pid为0，父进程中返回pid为子进程pid。默认情况下子进程继承父进程所有资源，通过execve（path，argv，evn）执行系统调用加载新的二进制程序时，继承的父类资源会被清掉，采用新的二进制的资源。

  信号处理（SIGCHLD）：这个信号比较常见，一般fork子进程时父进程比较关注这个信号。如父进程-->fork-->子进程 。当init进程fork出zygote进程，zygote挂了，init进程会收到SIGCHLD信号，然后可重建zygote进程。

- Zygote进程启动后做了些什么事？

  init 进程读取init.rc文件创建了Zygote进程，Zygote被创建后执行exeve系统调用执行二进制可执行程序，打开Java世界。

  Zygote 主要做了三件事，即：执行系统调用启动安卓虚拟机-->注册安卓JNI函数-->进入Java 世界（开启JVM，执行目标Java Main 方法。普通的c++程序调用java的代码示例如下代码）。

  ​

  ```java
   int main(int argc, char *argv[]) {
          JavaVM *jvm;
          JNIEnv *env;
          //创建java虚拟机
          JNI_ CreateJavaVM(&jvm, (void **) &env, &vm_ args);
          //找到ZygoteInit java 类
          jclass clazz = env-> FindClass("Zygotelnit");

          //找ZygoteInit 类 main 静态函数
          jmethodID method = env- > GetStaticMethodlD(clazz,
                  "Main", "([Ljava/lang/String;)V");
          // 执行mian 方法
          env-> CallStaticVoidMethod(clazz, method, args);
          
          jvm-> DestroyJavaVM(); // 销毁jVM
          
      }
  ```

  ​

  Zygote java 世界主要做了三件事，即：预加载系统资源（为了将来孵化子进程时继承给他们）、启动SystemSever、进入Loop循环（等待Socket消息。Loop循环中处理Socket消息的核心代码如下）

  ```java
    boolean runOne(){
          //读取参数列表
          String []args = readArgumentList();
          // 根据参数启动子进程
          int pid = Zygote.forkAndSpecialize();

          if (pid==0){
              // 子进程中干活，执行java main方法：
              //1、方法名ActivityThread的main方法
              //2、这个方法类名、参数名是AMS通过Socket 跨进程发送来的。
              handleChildProc(args,...);
              return true;
          }
      }
  ```

  ​

  我们自己的应用进程直接可以进行JNI调用，不需要创建Java VM，因为我们的进程是由Zygote进程孵化的，继承了Zygote进程创建时创建的java VM。直接重置下虚拟机的状态即可使用。


###### 3、Zygote的启动流程细节注意：

Zygote fork时要单进程。不管父进程中有多少线程，子进程被fork出时子进程中只有一个进程。对于子进程来说多余的线程就不见了，还可能导致死锁、状态不一致问题。问了避免这个问题，在fork子进程时就把主线程之外的线程全停了。等fork完成之后再把这些线程重启即可。其实Zygote就是这么做的。Zygote不是单线程的，其内部还跑了很多虚拟机相关的守护线程。

Zygote的跨进程未采用Binder机制，而是采用本地的Socket。binder机制是在应用程序进程启动之后才开始启动。

### 三、问题思考

###### 1、孵化应用进程这种事为啥不交给SystemSever来做？而专门设计了Zygote。

​    应用在启动的时候需要做很多准备工作，包括启动虚拟机，加载各类系统资源等等，这些都是非常耗时的，如果能在zygote里就给这些必要的初始化工作做好，子进程在fork的时候就能直接共享，那么这样的话效率就会非常高。

   SystemServer里跑了一堆系统服务，这些是不能继承到应用进程的。而且我们应用进程在启动的时候，内存空间除了必要的资源外，最好是干干净净的，不要继承一堆乱七八糟的东西。所以呢，不如给SystemServer和应用进程里都要用到的资源抽出来单独放在一个进程里，也就是这的zygote进程，然后zygote进程再分别孵化出SystemServer进程和应用进程。孵化出来之后，SystemServer进程和应用进程就可以各干各的事了。

###### 2、Zygote的IPC机制为啥不采用Binder，如果采用Binder 会有什么问题？

   采用Binder机制，首先zygote要启用binder机制，需要打开binder驱动，获得一个描述符，再通过mmap进行内存映射，还要注册binder线程，这还不够，还要创建一个binder对象注册到serviceManager，另外AMS要向zygote发起创建应用进程请求的话，要先从serviceManager查询zygote的binder对象，然后再发起binder调用，这来来回回好几趟非常繁琐。相比之下，zygote和SystemServer进程本来就是父子关系，对于简单的消息通信，用管道或者socket非常方便省事。

 

### 四、小结

谈谈对Zygote理解如何回答？这点可从3w回答：

what：干嘛的，即作用。

how：怎么干的，即启动流程（启动关键步骤）

why：工作原理，即（如何孵化进程，如何与别人通信）