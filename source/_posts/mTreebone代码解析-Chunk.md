---
title: mTreebone代码解析-Chunk
categories:
  - C++
tags:
  - 仿真
  - P2P
  - 视频
author: 长歌
abbrlink: 290678603
date: 2019-11-05 18:38:00
---

视频块，用于缓存及播放使用
<!-- More -->

# Chunk
## 头文件
### Chunk.h
```c++
#include "VideoFrame.h"

class Chunk
{
protected:
    short int chunkNumber; // 创建的对象的块号
    short int chunkSize; // 块对象中的帧数
    short int hopCount; // 遍历网络的块的跳数
    double creationTime; // 创建块的时间。 它用于计算端到端延迟

public:
    VideoFrame* chunk; // 将视频帧保持为块的数组
    /**
     * 此函数用于初始化块类
     *
     * @param ChunkSize 此块中可用的帧数
     */
    void setValues(int ChunkSize);
    /**
     * 基类构造函数
     */
    Chunk();
    /**
     * 基类析构函数
     */
    ~Chunk();
    /**
     * 根据数字在块数组中设置一个帧
     *
     * @param VideoFrame 放置在这个块中的框架
     */
    void setFrame(VideoFrame vFrame);
    /**
     * 检查这个块是否填充了它的帧
     *
     * @return boolean true if it is contained all frames
     */
    bool isComplete();
    /**
     * 设置此块的数量
     *
     * @param ChunkNumber 要为块编号设置的编号
     */
    void setChunkNumber(int ChunkNumber);
    /**
     * 得到这个块的块号
     *
     * @return integer the chunk number
     */
    short int getChunkNumber(){return chunkNumber;}
    /**
     * 根据包含的帧和其他变量获取块的字节大小
     *
     * @return integer byte size of this chunk
     */
    int getChunkByteLength();
    /**
     * 获取块遍历的跃点的值
     *
     * @return integer hop count number
     */
    int getHopCount(){return hopCount;}
    /**
     * 获得创建这个块的时间
     * @return double creation time value
     */
    double getCreationTime();
    /**
     * 设置跳数计数
     *
     * @param HopCount to be set
     */
    void setHopCout(int HopCount);
    /**
     * Chunk的标准输出流，给出所有包含的帧
     *
     * @param os the ostream
     * @param c the Chunk
     * @return the output stream
     */
    friend std::ostream& operator<<(std::ostream& os, const Chunk& c);
    /**
     * 给出在此块中插入的最后一个帧编号
     *
     * @return integer las frame numbers
     */
    int getLastFrameNo();
    /**
     * 计算由于迟到而丢失的帧的大小
     *
     * @param playbackPoint 当前正在播放的帧编号
     * @return integer size of late arrival frames
     */
    int getLateArrivalLossSize(int playBackPoint);
};
```

## 源文件
### Chunk.cc
```c++
#include "Chunk.h"

// 基类构造函数
Chunk::Chunk()
{
    chunkNumber = -1;   // 创建的对象的块号
}

// 基类析构函数
Chunk::~Chunk()
{
}

// 初始化块类
void Chunk::setValues(int ChunkSize)
{
    chunkSize = ChunkSize;                  // 块对象中的帧数
    chunk = new VideoFrame[chunkSize];      // 将视频帧保持为块的数组
}

// 检查这个块是否填充了它的帧
bool Chunk::isComplete()
{
    for(int i = 0 ; i<chunkSize ; i++)
        if(!chunk[i].isSet())
            return false;
    return true;
}

// 设置此块的数量
void Chunk::setChunkNumber(int ChunkNumber)
{
    chunkNumber = ChunkNumber;
}

/**
 * [Chunk::setFrame 设置视频帧]
 * @param vFrame [视频帧]
 */
void Chunk::setFrame(VideoFrame vFrame)
{
    // 余数操作，举个栗子:每个块保存4帧视频帧，帧号为10的视频帧应该保存在第3个块的帧数组下标[2]中
    int ExtractedFrameNum = vFrame.getFrameNumber()%chunkSize;  
    chunk[ExtractedFrameNum].setFrame(vFrame);
}

/**
 * [Chunk::getChunkByteLength 获取块的字节长度]
 * @return [字节长度]
 */
int Chunk::getChunkByteLength()
{
    int totalSize = 0;
    if(isComplete())
    {
        for(int i = 0 ; i < chunkSize ; i++)
            totalSize += chunk[i].getFrameLength();
        return totalSize;
    }
    else
        return -1;
}

// Chunk的标准输出流，给出所有包含的帧
std::ostream& operator<<(std::ostream& os, const Chunk& c)
{
    os << c.chunkNumber<< ": ";
    for(int i = 0 ; i < c.chunkSize ; i++)
        os << c.chunk[i] << " ";
    return os;
}

// 设置跳数计数
void Chunk::setHopCout(int HopCount)
{
    hopCount = HopCount;
}

// 获得创建这个块的时间
double Chunk::getCreationTime()
{
    return chunk[chunkSize -1].getCreationTime();
}

// 给出在此块中插入的最后一个帧编号
int Chunk::getLastFrameNo()
{
    return chunk[chunkSize-1].getFrameNumber();
}

/**
 * [Chunk::getLateArrivalLossSize 计算到达了多少视频帧]
 * @param  playBackPoint [正在播放的帧编号]
 */
int Chunk::getLateArrivalLossSize(int playBackPoint)
{
    /**
     * 遍历块中帧数组，如果帧的编号>播放帧编号，
     * 表示后续帧已到达，统计大小，最后返回
     */
    int totalSize = 0;
    for(int i = 0 ; i < chunkSize ;i++)
        if(chunk[i].getFrameNumber() > playBackPoint)
            totalSize += chunk[i].getFrameLength();
    return totalSize;   // 到达帧的大小
}
```