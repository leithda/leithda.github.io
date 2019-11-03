---
title: mTreebone代码解析-VedioFrame
categories:
  - C++
tags:
  - 仿真
  - P2P
  - 视频
author: 长歌
abbrlink: 505931268
date: 2019-11-01 09:00:00
---

视频帧对应的对象
<!-- More -->

# VideoFrame
## 头文件
```c++
#include <iostream>

class VideoFrame
{
public:
    /**
     * 检查是否设置了VideoFrame对象的值。
     */
    bool isSet();

    /**
     * VideoFrame 构造函数
     */
    VideoFrame();

    /**
     * 设置VideoFrame对象的值。
     *
     * @param FrameNumber 视频帧号
     * @param FrameLength 视频帧大小（字节）
     * @param FrameType 视频帧类型（I，P或B）
     */
    VideoFrame(int FrameNumber, int FrameLength, char FrameType,double CreationTime);
    
    /**
     * 对视频帧进行赋值
     */
    void setFrame(VideoFrame vFrame);

    /**
     * 设置下一帧的编号
     *
     * @param FrameNumber 要设置的输入帧编号
     */
    void setFrameNumber(int FrameNumber);

    // 一系列get 方法
    char getFrameType(){return frameType;}

    int getFrameNumber(){return frameNumber;}

    int getFrameLength(){return frameLength;}

    double getCreationTime(){return creationTime;}

    VideoFrame getVFrame();

protected:
    char frameType; // 视频帧的类型
    int frameNumber;    // 帧号（每个视频帧的唯一标识符
    short int frameLength;  // 视频帧的大小，以字节为单位
    float creationTime; // 创建此框架的SimTime
    /**
     * 设置帧属性
     *
     * @param FrameNumber 帧号
     * @param FrameLength 帧大小（以字节为单位）
     * @param FrameType 帧的类型（可以是I，P或B）
     * @param CreationTime 创建帧的时间
     */
    void setFrame(int FrameNumber, int FrameLength, char FrameType,double CreationTime);
    /**
     * VideoFrame的标准输出流，提供视频帧中的所有信息
     *
     * @param os the ostream
     * @param v the VideoFrame
     * @return the output stream
     */
    friend std::ostream& operator<<(std::ostream& os, const VideoFrame& v);
};
```

## 源文件 
### VideoFrame.cc
```c++
#include "VideoFrame.h"

// VideoFrame基类构造函数
VideoFrame::VideoFrame()
{
    setFrame(-1,-1, 'N',-1);
}

// VideoFrame替代构造函数
VideoFrame::VideoFrame(int FrameNumber, int FrameLength, char FrameType,double CreationTime)
{
    setFrame(FrameNumber, FrameLength,FrameType,CreationTime);
}

// 检查是否设置了VideoFrame对象的值
bool VideoFrame::isSet()
{
    if(frameType == 'N' || frameLength == -1)
        return false;
    else
        return true;
}

// 设置VideoFrame对象的值
void VideoFrame::setFrame(VideoFrame vFrame)
{
    frameNumber = vFrame.getFrameNumber();    // 视频帧号
    frameLength = vFrame.getFrameLength();    // 视频帧大小（字节）
    frameType = vFrame.getFrameType();        // 视频帧类型（I，P或B）
    creationTime = vFrame.getCreationTime();  // 创建时间
}

// 返回一个等于当前对象的对象
VideoFrame VideoFrame::getVFrame()
{
    VideoFrame vFrame;
    vFrame.setFrame(frameNumber,frameLength,frameType,creationTime);
    return vFrame;
}

// 设置帧属性
void VideoFrame::setFrame(int FrameNumber, int FrameLength, char FrameType,double CreationTime)
{
    frameNumber = FrameNumber;      // 视频帧号
    frameLength = FrameLength;      // 视频帧大小（字节）
    frameType = FrameType;          // 视频帧类型（I，P或B）
    creationTime = CreationTime;    // 创建时间
}

// 设置下一帧的编号
void VideoFrame::setFrameNumber(int FrameNumber)
{
    frameNumber = FrameNumber;       // 视频帧号
}

// VideoFrame的标准输出流，提供视频帧中的所有信息
std::ostream& operator<<(std::ostream& os, const VideoFrame& v)
{
    os << v.frameNumber<<","
    << v.frameType << ","
    <<v.frameLength;
    return os;
}
```