---
layout:     post
title:      基于Gstreamer和大疆OSDK4.0视频h264接口推流
subtitle:   
date:       2020-08-27
author:     Ganglin
header-img: img/post-bg-4.jpg
catalog: true
tags:
    - CV
    - GStreamer


---



# 基于Gstreamer和大疆OSDK4.0视频h264接口推流
## 背景

为了实现无人机视频实时推流和图像处理，首先要完成视频编解码，大疆的视频接口实在是坑太多了！参考了很多大神的文章，大多都是解码本地文件或者直接从服务器拉流，不能实现我想要的实时动态流解码，搞了半个月终于能实时解码了，希望我的研究结果能帮助更多人。

 1. 主流视频压缩格式是h264(IDR编码)，相关教程很多，而GDR编码相关的内容几乎没有，没法以字节流提取nalu的方式解码；
 2. 直接采样飞机视频保存为本地文件无法直接播放，转为mp4格式后播放会有花屏；
 3. 大疆的软解接口cpu负载太高。


大疆M300的H20主摄像头视频编码格式为h264(GDR)，OSDK-4.0提供了三个飞机摄像头视频接口sample。

 1. OSDK调用ffmpeg软解h264裸流，传出OpenCV可读的Mat数组，直接显示一帧图像；
 2. 以第一个接口为基础，增加了循环和睡眠，连续推出图像帧模拟视频流；
 3. 直接提供一个h264(GDR)裸流的函数，传出参数是一个数据包地址和数据包长度。

## 问题
有一个问题，大疆提供的接口初始化摄像头相关参数后会循环回调一个数据包推送的liveView函数，而GStreamer导入外部数据的插件appsrc也需要循环回调need-data函数，两个循环回调函数同时运行还要数据交互同步，我设计了一个循环队列的缓冲区来存放公共数据，使用互斥锁配合条件变量来保证两个线程同步。
## 实现
发送端：
```cpp
/***************************************
@Copyright(C) All rights reserved.
@Author DENG Ganglin
@Date   2020.08
***************************************/

#include "dji_vehicle.hpp"
#include "dji_linux_helpers.hpp"

#include <gst/gst.h>
#include <gst/gstminiobject.h>
#include <glib.h>

#include <stdlib.h>
#include <pthread.h>
#include <iostream>
#include <sys/stat.h>
#include <sys/types.h>
#include <queue>
#include <pthread.h>
#include <unistd.h>

#include "loopqueue.hpp"

using namespace DJI::OSDK;
using namespace std;

// 创建插件
GstElement *pipeline, *appsrc, *parse, *decoder, *scale, *filter1, *conv, *encoder, *filter2, *rtppay, *udpsink;
GstCaps *scale_caps;
GstCaps *omx_caps;
static GMainLoop *loop;
GstBuffer *buf;

// 定义结构体,保存生产者的数据
typedef struct Msg
{
	int len;
	uint8_t* data;
}msg;

// 创建循环队列
LoopQueue<msg*> loopqueue(256);

pthread_mutex_t myMutex = PTHREAD_MUTEX_INITIALIZER;  // 初始化互斥锁
pthread_cond_t  cond = PTHREAD_COND_INITIALIZER;  // 初始化条件变量

int count_t = 0; // 打印检查数据流动是否正常

static void cb_need_data(GstElement *appsrc, guint size, gpointer user_data)
{
	static GstClockTime timestamp = 0;
	GstFlowReturn ret;
	GstMapInfo map;

	count_t++;

	// 给互斥量加锁
	pthread_mutex_lock(&myMutex);

	// 判断条件变量是否满足 访问公共区条件
	while (0 == loopqueue.getSize())
	{
		pthread_cond_wait(&cond, &myMutex);
	}
	// 访问公共区, 取数据 
	// 往appsrc输入数据
	buf = gst_buffer_new_allocate(NULL, loopqueue.top()->len, NULL);
	gst_buffer_map(buf, &map, GST_MAP_WRITE);
	memcpy((guchar *)map.data, (guchar *)(loopqueue.top()->data), loopqueue.top()->len);

	printf("count_t = %d, bufLen = %d\n", count_t, loopqueue.top()->len);
	// 给互斥量解锁
	pthread_mutex_unlock(&myMutex);

	GST_BUFFER_PTS(buf) = timestamp;
	GST_BUFFER_DURATION(buf) = gst_util_uint64_scale_int(1, GST_SECOND, 1);
	timestamp += GST_BUFFER_DURATION(buf);
	g_signal_emit_by_name(appsrc, "push-buffer", buf, &ret);

	gst_buffer_unref(buf);
	if (ret != 0)
	{
		g_main_loop_quit(loop);
	}

	// 释放摘下的结点
	delete loopqueue.top()->data;
	delete loopqueue.top();
	loopqueue.pop();
}

void liveViewSampleCb(uint8_t* buf264, int bufLen, void* userData)
{
	// 生产数据
	msg *mp = new msg;
	mp->len = bufLen;
	mp->data = new uint8_t[bufLen];
	memcpy(mp->data, buf264, bufLen);
	// 加锁
	pthread_mutex_lock(&myMutex);
	// 访问公共区
	loopqueue.push(mp);
	// 解锁
	pthread_mutex_unlock(&myMutex);
	// 唤醒阻塞在条件变量上的消费者
	pthread_cond_signal(&cond);
	return;
}

// 创建管道，触发need-data信号
void *consumer(void *arg)
{
	// init GStreamer
	gst_init(NULL, NULL);
	loop = g_main_loop_new(NULL, FALSE);

	// setup pipeline
	pipeline = gst_pipeline_new("pipeline");
	appsrc = gst_element_factory_make("appsrc", "source");
	parse = gst_element_factory_make("h264parse", "parse");
	decoder = gst_element_factory_make("avdec_h264", "decoder");
	scale = gst_element_factory_make("videoscale", "scale");
	filter1 = gst_element_factory_make("capsfilter", "filter1");
	conv = gst_element_factory_make("videoconvert", "conv");
	encoder = gst_element_factory_make("omxh264enc", "encoder");
	filter2 = gst_element_factory_make("capsfilter", "filter2");
	rtppay = gst_element_factory_make("rtph264pay", "rtppay");
	udpsink = gst_element_factory_make("udpsink", "sink");

	scale_caps = gst_caps_new_simple("video/x-raw",
		"width", G_TYPE_INT, 1280,
		"height", G_TYPE_INT, 720,
		NULL);

	omx_caps = gst_caps_new_simple("video/x-h264",
		"stream-format", G_TYPE_STRING, "byte-stream",
		NULL);

	g_object_set(G_OBJECT(filter1), "caps", scale_caps, NULL);
	g_object_set(G_OBJECT(filter2), "caps", omx_caps, NULL);
	gst_caps_unref(scale_caps);
	gst_caps_unref(omx_caps);

	g_object_set(udpsink, "host", "127.0.0.1", NULL);
	g_object_set(udpsink, "port", 5555, NULL);
	g_object_set(udpsink, "sync", false, NULL);
	g_object_set(udpsink, "async", false, NULL);

	// setup	
	//g_object_set(G_OBJECT(appsrc), "caps",
	//	gst_caps_new_simple("video/x-raw",
	//		"stream-format", G_TYPE_STRING, "byte-stream",
	//		NULL), NULL);

	gst_bin_add_many(GST_BIN(pipeline), appsrc, parse, decoder, scale, filter1, conv, encoder, filter2, rtppay, udpsink, NULL);
	gst_element_link_many(appsrc, parse, decoder, scale, filter1, conv, encoder, filter2, rtppay, udpsink, NULL);

	// setup appsrc
	//g_object_set(G_OBJECT(appsrc),
	//	"stream-type", 0,
	//	"format", GST_FORMAT_TIME, NULL);

	g_signal_connect(appsrc, "need-data", G_CALLBACK(cb_need_data), NULL);

	/* play */
	std::cout << "start sender pipeline" << std::endl;
	gst_element_set_state(pipeline, GST_STATE_PLAYING);
	g_main_loop_run(loop);
	/* clean up */
	gst_element_set_state(pipeline, GST_STATE_NULL);
	gst_object_unref(GST_OBJECT(pipeline));
	g_main_loop_unref(loop);
	gst_buffer_unref(buf);

}

// 初始化飞机开始推送264码流数据包
void *producer(void *arg)
{
	while (true)
	{
		// 启动 OSDK.
		bool enableAdvancedSensing = true;
		LinuxSetup linuxEnvironment(1, NULL, enableAdvancedSensing);
		Vehicle *vehicle = linuxEnvironment.getVehicle();
		if (vehicle == nullptr)
		{
			std::cout << "Vehicle not initialized, exiting.\n";
			return NULL;
		}
		vehicle->advancedSensing->startH264Stream(LiveView::OSDK_CAMERA_POSITION_NO_1,
			liveViewSampleCb,
			NULL);
		sleep(100); //可根据需求定义回调函数一次调用的时间
	}
}

int main(int argc, char** argv)
{
	// 创建一个生产者线程和一个消费者线程
	int rc1, rc2;
	pthread_t pid, cid;

	if (rc1 = pthread_create(&pid, NULL, producer, NULL))
		cout << "producer thread creation failed: " << rc1 << endl;
	if (rc2 = pthread_create(&cid, NULL, consumer, NULL))
		cout << "consumer thread creation failed: " << rc2 << endl;

	pthread_join(pid, NULL);
	pthread_join(cid, NULL);

	pthread_mutex_destroy(&myMutex);
	pthread_cond_destroy(&cond);
	return 0;
}
```
接收端：

```cpp
/***************************************
@Copyright(C) All rights reserved.
@Author DENG Ganglin
@Date   2020.08
***************************************/

#include "opencv2/opencv.hpp"
#include <cstdio>

using namespace cv;
using namespace std;

int main(int argc, char** argv)
{
	VideoCapture cap("udpsrc port=5555 ! application/x-rtp,encoding-name=H264 ! rtpjitterbuffer latency=0 ! rtph264depay ! avdec_h264 ! videoconvert ! appsink");

	if( !cap.isOpened())
	{
		cout<<"Fail"<<endl;
		return -1;
	}

	namedWindow("frame",1);
	for(;;)
	{
		Mat frame(1280,720, CV_8UC3, Scalar(0,0,255));
		cap >> frame; // get a new frame from camera

		imshow("frame", frame);

		if(waitKey(30) >= 0) break;
	}
	return 0;
}
```

## 总结

 1. 没有放循环队列的hpp代码，非原创，网上有很多；
 2. 一个愚蠢的小错误，申请内存new uint8_t[bufLen]写成new uint8_t(bufLen），相当于只申请了一个uint8_t的内存，程序运行起来一直报非法内存，不应该；
 3. 实测发送端解析裸流后第一次编码只能用软编码，后面解码可以用硬解码，cpu负载约180%，比全软解250%稍低；
 4. 实际效果还行，延时约800ms，后续还可以优化。