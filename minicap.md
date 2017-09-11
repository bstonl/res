# OpenStf远程控制之minicap原理简介
--
> by 邓将军

## 1.简介

[openstf](https://github.com/openstf)是一个可自动化测试+远程控制（仅针对android平台）的真正意义上的全栈框架。本文将分析其远程控制之重要组件minicap（屏幕投影的)实现原理。
该组件仅适用于android设备，能够实时截屏，通过内部的socket server接收外部请求，转发帧数据。该组件通过adb启动，故而不需要root。可以在SDK 26及以下的任何设备上运行（有些设备可能在开发者选项中有额外的选项如“是否允许adb截屏”、“是否允许adb模拟触摸事件”），截屏帧率10-50不等（根据设备、USB速率等影响）。
这个不同于screenrecord录屏工具，基于minicap可以实现实时屏幕投影。如果你想实现一款自定义的android远程控制软件，本文将带你入门。

> 屏幕投影使用方法可以参见[cap-android](https://github.com/bstonl/cap-android)。
忽略前面的所有步骤，手机端就干了一件事：

```
adb shell LD_LIBRARY_PATH=/data/local/tmp /data/local/tmp/minicap -P $size@$size/0
```
> 这句命令的意思是：通过adb执行shell，通过shell执行/data/local/tmp/目录下的可执行文件minicap，前缀LD_LIBRARY_PATH=/data/local/tmp意思是动态库链接目录（优先链接）。

## 2.几个概念
#### FrameBuffer
#### Vsync 
[参考链接](http://www.jianshu.com/p/59ad90bff2a7)
* 刷新率
* 帧速率
#### BufferQueue
[参考链接1](http://dragon.leanote.com/post/Android%E5%9B%BE%E5%BD%A2%E7%B3%BB%E7%BB%9F-II-%E6%9E%B6%E6%9E%84)
[参考链接2]()

**BufferQueue主要结构如下**：
> BufferQueue
>
> BufferQueueCore
>
> BufferQueueProducer
> 
> BufferQueueConsumer

![BufferQueue结构](http://img.blog.csdn.net/20151010171418943?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

手机图形系统是一个典型的生产-消费模型。app + 系统Ui作为生产者不停的产生新的帧缓冲数据，这些缓冲数据有一个专门的数据结构管理，即是BufferQueue。BufferQueue
又由SurfaceFlinger管理入队、出队，并与HAL关联，最终将Buffer反应在硬件上（显示
屏）。SDK21及以上对BufferQueue代码结构进行了优化，通过BufferQueue管理
BufferQueueProducer 与 BufferQueueConsumer，降低了Consumer与Producer之间的耦合。

* IGraphicBufferProducer

	Producer即为当前前台app与系统UI（状态栏、虚拟键等）, surfaceflinger会将这些显示数据混合。
* IGraphicBufferConsumer
 - SurfaceFlingerConsumer

## 3.原理
### 3.1.原理简介
#### 3.1.1.图形系统
系统在进行屏幕绘制的时候会开辟一个缓冲区，通常来说这个缓冲区是一个
循环队列（BufferQueue）。队列中的每一个元素（Buffer）持有屏幕每一帧的数据。
每一个app都是一个Buffer生产者。BuffferQueue与SurfaFlinger关联，不直
接与系统底层互动。当底层需要刷新屏幕显示时，
会触发信号Vsync(http://www.jianshu.com/p/59ad90bff2a7)。app生产
的Buffer会在Vsync到达之后移入到Display中管理。Dispaly将app与系统UI
（状态栏等）叠加，最终绘制在屏幕上，原理图如下所示。
![SurfaceFlinger+BufferQueue](https://leanote.com/api/file/getImage?fileId=578c7bd8ab644135ea0147cb)
	
#### 3.1.2.minicap基本原理
minicap初始化流程：通过注册可获取Buffer的回调（gWaiter），实时截取帧数据，这个格式通常为
RGBA（可配置，不过虽然可以配置为YUV格式，但是貌似并不会有效。
初始化成功后，minicap启动一个socket server，当监听到新连接socket client之后，会执行如下流程：
轮询gWaiter状态，当有新的Frame数据饿到达后，通过JpgEncoder将其编码为Jpg格式，然后将数据发送到socket client。

### 3.1.2.minicap源码分析
#### 3.1.2.1分析源码的入口
> android native源码编译底层基于gnumake，我们常用的[ndk-build](https://developer.android.com/ndk/guides/ndk-build.html?hl=zh-cn)不过是一个命令脚本。
官方minicap工程结构如下：
 
```
jni
    minicap
    minicap-shared
    vendor
        libjpeg-turbo
```
	
 通过名字大致可以猜测 minicap是主工程入口，minicap-share是共享库工程，
 vendor则是第三方库工程目录。libjpeg-turbo是一个跨平台的高速图片处理库。


#### 3.1.2.2.minicap目录

##### 3.1.2.2.1.目录结构

```
minicap
    util
        debug.h
        formatter.hpp
    Android.mk
    JpgEncoder.cpp/hpp
    Projection.cpp/hpp
    SimpleServer.cpp/hpp
    minicap.cpp
```
	
##### 3.1.2.2.2.Android.mk
[参考链接1](http://android.mk/)
[参考链接2](https://developer.android.com/ndk/guides/android_mk.html)

Android.mk是一个mini的GNU makefile片段。

常用格式：


**其实就是通过mk设置构建过程的一些环境变量**。

```
# my-dir是构建系统宏函数，返回当前目录。
LOCAL_PATH := $(call my-dir)
# CLEAR_VARS构建系统提供，用于清除LOCAL_XXX变量。但不会清除LOCAL_PATH
include $(CLEAR_VARS)
# 构建模块名称
LOCAL_MODULE := minicap-common
# 构建所需源文件
LOCAL_SRC_FILES := \
	JpgEncoder.cpp \
	SimpleServer.cpp \
	minicap.cpp \
# 构建所需静态库
LOCAL_STATIC_LIBRARIES := \
	libjpeg-turbo \
# 构建所需动态库
LOCAL_SHARED_LIBRARIES := \
	minicap-shared \
# 构建为静态库，其它为BUILD_SHARE_LIBRARY/BUILD_EXECUTABLE
include $(BUILD_STATIC_LIBRARY)
```

#### 3.1.2.2.3.SimpleServer

##### 类结构

```
class SimpleServer {
public:
  SimpleServer();
  ~SimpleServer();

  int
  start(const char* sockname);

  int 
  accept();

private:
  int mFd; 
};
```
这个类即是一个socket server封装类，主要有start、accept两个成员函数。
成员变量mFd是socket的文件描述符（一切皆文件）， 通过mFd我们可以对启动的socket进行
读写（发送、接收）。

- ##### start函数
```
	// 传入自定义的socket名称，初始化socket server
	int
	SimpleServer::start(const char* sockname) {
	  // AF_UNIX为协议族，SOCK_STREAM为socket类型，0代表该类型的默认协议。
	  // AF_UNIX为unix域的（根据名称，比如“minicap”），常用的还有AF_INET（根据ip：port的方式）。
	  // 1.新建socket
	  int sfd = socket(AF_UNIX, SOCK_STREAM, 0);
	
	  if (sfd < 0) {
	    return sfd;
	  }
	  // 2.socket地址初始化，sockaddr_un结构体
	  struct sockaddr_un addr;
	  // 清零addr这段内存，这个操作很有必要，声明一个变量，那不管有没有初始化，
	  // 都会占用内存，这段内存并不是干净的，因为c的机制是没有内存管理和自动清0的。
	  memset(&addr, 0, sizeof(addr));
	  addr.sun_family = AF_UNIX;
	  strncpy(&addr.sun_path[1], sockname, strlen(sockname));
	  // 3.根据新建的fd绑定socket地址
	  if (::bind(sfd, (struct sockaddr*) &addr,
	      sizeof(sa_family_t) + strlen(sockname) + 1) < 0) {
	    ::close(sfd);
	    return -1;
	  }
	  // 4.socket server监听。请求队列最大长度为1。
	  ::listen(sfd, 1);
	
	  mFd = sfd;
	
	  return mFd;
}
```

- ##### accept函数
```
	int
	SimpleServer::accept() {
	  struct sockaddr_un addr;
	  socklen_t addr_len = sizeof(addr);
	  return ::accept(mFd, (struct sockaddr *) &addr, &addr_len);
	}
```
socket server接收到新连接，建立连接，返回socekt client fd。

#### 3.1.2.2.4.JpgEncoder

rgba格式

|-----stride-------------------------|

|-------width-------|--padding---|

##### 类结构

```
class JpgEncoder {
public:
  JpgEncoder(unsigned int prePadding, unsigned int postPadding);
  ~JpgEncoder();
  bool encode(Minicap::Frame* frame, unsigned int quality);
  int getEncodedSize();
  unsigned char* getEncodedData();
  bool reserveData(uint32_t width, uint32_t height);
private:
  unsigned char* mEncodedData;
}
```

该类主要成员函数包括：编码encode，获取编码后的数据getEncodedData()。

- encode函数

	encode的实现主要就是调用了turbojpeg中的tjCompress2函数。

- 析构函数

 最后需要记得在析构函数
～JpgEncoder（）中调用tjFree(mEncodedData)释放内存以防内存泄漏（通常，如果不是
new或malloc的内存，会在该变量所在域的生命期结束后自动销毁并调用析构函数）。

##### 3.1.2.2.5.util/debug.h日志打印

- 日志打印

	可以打印行号，日志级别，错误码，开发中可以参照如下格式。
	
	```
	#include <stdio.h>
	#include <errno.h>
	#include <string.h>
		
	#ifdef NDEBUG
	#define MCDEBUG(M, ...)
	#else
	#define MCDEBUG(M, ...) fprintf(stderr, "DEBUG: %s:%d: " M "\n", __FILE__, __LINE__, ##__VA_ARGS__)
	#endif
		
	#define MCCLEAN_ERRNO() (errno == 0 ? "None" : strerror(errno))
		
	#define MCERROR(M, ...) fprintf(stderr, "ERROR: (%s:%d: errno: %s) " M "\n", __FILE__, __LINE__, MCCLEAN_ERRNO(), ##__VA_ARGS__)
		
	#define MCWARN(M, ...) fprintf(stderr, "WARN: (%s:%d: errno: %s) " M "\n", __FILE__, __LINE__, MCCLEAN_ERRNO(), ##__VA_ARGS__)
		
	#define MCINFO(M, ...) fprintf(stderr, "INFO: (%s:%d) " M "\n", __FILE__, __LINE__, ##__VA_ARGS__)
		
	#define MCCHECK(A, M, ...) if(!(A)) { MCERROR(M, ##__VA_ARGS__); errno=0; goto error; }
		
	#define MCSENTINEL(M, ...)  { MCERROR(M, ##__VA_ARGS__); errno=0; goto error; }
		
	#define MCCHECK_MEM(A) check((A), "Out of memory.")
		
	#define MCCHECK_DEBUG(A, M, ...) if(!(A)) { MCDEBUG(M, ##__VA_ARGS__); errno=0; goto error; }
	```
- 常见错误码

	[参见](https://android.googlesource.com/platform/frameworks/native/+/jb-dev/include/utils/Errors.h)

	系统对错误码都有相关的宏定义，在实际使用时可以完全忽略实际值，看定义名称就可以了。
	
	```
	android::NO_ERROR
	android::UNKONOW_ERROR
	android::NO_MEMORY:
	android::INVALID_OPERATION:
	android::BAD_VALUE:
	android::BAD_TYPE:
	android::NAME_NOT_FOUND:
	android::PERMISSION_DENIED:
	android::NO_INIT:
	android::ALREADY_EXISTS:
	android::DEAD_OBJECT:
	android::FAILED_TRANSACTION:
	android::BAD_INDEX:
	android::NOT_ENOUGH_DATA:
	android::WOULD_BLOCK:
	android::TIMED_OUT:
	android::UNKNOWN_TRANSACTION:
	android::FDS_NOT_ALLOWED:
	```


##### 3.1.2.2.6.主程序minicap.cpp
主程序的逻辑即是上述的 3.1.2 “minicap基本原理”。

#### 3.1.2.3.mincap-share
minicap-share包含了sdk14-26的版本以及sdk9。
本次分析基于sdk-23。
Minicap类主要结构：

- 内部结构体
	- Frame
	
		> 屏幕帧数据结构，相当于BufferItem的适配格式。
	- FrameAvailableListener
		> 屏幕帧数据可用回调。
- 成员函数
	- createVirtualDisplay
	- setFrameAvailableListener
	- consumePendingFrame

- 头文件 
	
	```
	class Minicap {
	// 关于C++中的结构体和类的区别，实在不好妄下结论，看具体的编译器吧。
	// 结构体中的变量默认为public。
	public:
		struct Frame {
		    void const* data;
		    Format format;
		    uint32_t width;
		    uint32_t height;
		    uint32_t stride;
		    uint32_t bpp;
		    size_t size;
		  };
		
		  struct FrameAvailableListener {
		    virtual
		    ~FrameAvailableListener() {}
		
		    virtual void
		    onFrameAvailable() = 0;
		  };
		  virtual int consumePendingFrame(Frame* frame) = 0;
  		  virtual void setFrameAvailableListener(FrameAvailableListener* listener) = 0;
	}
	```
主要有5个变量：

	```
	android::sp<android::BufferQueue> mBufferQueue;
	android::sp<android::CpuConsumer> mConsumer;
	android::sp<android::IBinder> mVirtualDisplay;
	android::sp<FrameProxy> mFrameProxy;
	Minicap::FrameAvailableListener* mUserFrameAvailableListener;
	```
通过继承FrameAvailableListener，然后注册回调，就可以拿到帧缓冲数据了。

## 4.项目地址
[android投影+控制github地址](https://github.com/bstonl/cap-android)



