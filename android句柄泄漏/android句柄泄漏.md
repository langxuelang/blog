#android句柄泄漏

###前言
---
在android开发过程中，跑一些单元测试，很容易暴露出文件句柄泄漏的问题。前段时间就有这么一个bug，最后确定是文件句柄泄漏的问题。下面我记录下当时一步步如何查找定位句柄泄漏。

---

### 正文
首先让我们看一眼抛错的log日志。

```
10-27 00:35:32.141  7437  7437 E AndroidRuntime: FATAL EXCEPTION: main

10-27 00:35:32.141  7437  7437 E AndroidRuntime: Process: com.Android56, PID: 7437

10-27 00:35:32.141  7437  7437 E AndroidRuntime: java.lang.RuntimeException: Could not read input channel file descriptors from parcel.

10-27 00:35:32.141  7437  7437 E AndroidRuntime: 	at android.view.InputChannel.nativeReadFromParcel(Native Method)

10-27 00:35:32.141  7437  7437 E AndroidRuntime: 	at android.view.InputChannel.readFromParcel(InputChannel.java:148)

10-27 00:35:32.141  7437  7437 E AndroidRuntime: 	at android.view.InputChannel$1.createFromParcel(InputChannel.java:39)
```
这里有一句Could not read input channel file descriptors from parcel，然后我们在这句话的上面又发现一个有价值的信息。

```
Caused by: java.io.IOException: Too many open files
```
通过网上搜索，基本判断这是一个文件句柄泄漏的问题。

那么我们该如何查找文件句柄泄漏的地方呢。

首先我们需要做到监控文件句柄数，由于android是linux的内核，所以，系统为每一个进程都有一个文件句柄的目录。
我们先通过ps命令，获取到我们app的进程id。
然后找到一个root过的手机，或者使用andorid模拟器，然后用adb连接到手机，通过shell命令进入到/proc/进程id/fd这个目录。由于linux关于系统的管理都是用文件方式，所以这个文件夹下面就是所有被打开的句柄。

我们可以在app运行的过程中，不断的进入到这个目录中，然后用ls -l 命令列出所有的文件句柄，这样就能看到文件句柄有哪些是增长的。然后再根据不同类型的文件句柄，初步定位是什么在泄漏。

在排查的过程中，为了方便获取某个进程的句柄数，我写了一个简单的shell脚本。有需要的同学可以拿去使用。

```
echo '查询进程占用文件句柄数'
set `adb shell ps |grep com.Android56 |grep -v channel |grep -v Daemon`
pidnum=$2
index=0
while true
do
index=$[index+1]
echo '##################'
echo '第'$index'次查询'
echo '总句柄'
adb shell ls -l /proc/$pidnum/fd |grep "" -c
echo 'anon句柄'
adb shell ls -l /proc/$pidnum/fd |grep anon -c
echo '##################'
sleep 2
done
```

---
###总结
通过艰难的排查过程，终于定位到是播放器打开so动态链接库的时候，一个文件没有释放。
具体而言就是，调用了dlopen、dlsym，但是没有调用dlclose这个方法。以后在编程的过程中，一定得多注意这种涉及资源方面的东西。有打开必须有关闭。同时我建议在测试的过程中，可以加入对文件句柄的监控，这样可以更好的发现问题，避免线上出bug。








