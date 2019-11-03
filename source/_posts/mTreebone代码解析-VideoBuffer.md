---
title: mTreebone代码解析-VedioBuffer
categories:
  - C++
tags:
  - 仿真
  - P2P
  - 视频
author: 长歌
abbrlink: 3544798524
date: 2019-11-01 09:00:00
---

视频缓存类
<!-- More -->

# VideoBuffer
## 头文件
### VideoBuffer.h
```c++
#include "Chunk.h"
#include "BufferMap.h"

class VideoBuffer
{
protected:
    int numOfBFrame; // 'I'和'P'之间或两个'P'帧之间的B帧数; 
    int bufferSize; // 缓冲区中可用的块数
    int chunkSize; // 块中可用的帧数
public:
    int lastSetChunk; // 最近设置的最后一块
    int lastSetFrame; // 最近设置的最后一帧
    int gopSize; // 每组图片的帧数 
    Chunk* chunkBuffer; // 保留一定数量块的视频缓冲区

    /**
     * 类构造函数
     * @param NumOfBFrame I和P之间的B帧数或两个P帧
     * @param BufferSize 缓冲区中可用的块数
     * @param ChunkSize 块中可用的帧数
     */
    VideoBuffer(int NumOfBFrame,int BufferSize, int ChunkSize, int GopSize);
    /**
     * 类析构函数
     */
    ~VideoBuffer();
    /**
     * 根据缓冲区中可用的内容设置BufferMap数组
     * @param BMap 要设置BufferMap
     */
    void updateBufferMap(BufferMap* BMap);
    /**
     * 将缓冲区移动一个块（一个块将被删除，新的空块将添加）
     */
    void shiftChunkBuf();
    /**
     *  根据数字给出帧的类型
     *  @param FrameNumber 要检查的帧号
     *  @return 帧的字符类型
     */
    char getFrameType(int FrameNumber);
    /**
     * 根据帧号在缓冲区中设置帧
     * @param vFrame 视频帧设置
     */
    void setFrame(VideoFrame vFrame);
    /**
     * 根据给定的帧编号给出视频帧
     * @param FrameNumber 要检索的帧编号
     * @return VideoFrame video frame
     */
    VideoFrame getFrame(int FrameNumber);
    /**
     * 根据给定的块数给出一个块
     * @param ChunkNumber 要查找的块号
     * @return Chunk video chunk to retreive
     */
    Chunk getChunk(int ChunkNumber);
    /**
     * 设置缓冲区中此节点接收的块
     * @param InputChunk 应该在缓冲区中设置的块
     */
    void setChunk(Chunk InputChunk);
    /**
     * VideoBuffer的标准输出流，给出所有包含的块和帧
     *
     * @param os the ostream
     * @param v the VideoBuffer
     * @return the output stream
     */
    friend std::ostream& operator<<(std::ostream& os, const VideoBuffer& v);
};
```

## 源文件
### VideoBuffer.cc
```c++
#include "VideoBuffer.h"

// 类构造函数
// NumOfBFrame：I和P之间或两个P帧之间的B帧数
 // BufferSize：缓冲区中可用的块数
 // GopSize：每组图片的帧数
VideoBuffer::VideoBuffer(int NumOfBFrame,int BufferSize, int ChunkSize, int GopSize)
{
    gopSize = GopSize;
    numOfBFrame = NumOfBFrame;
    bufferSize = BufferSize;
    chunkSize = ChunkSize;
    lastSetChunk = 0;
    lastSetFrame = 0;
    chunkBuffer = new Chunk[bufferSize];
    for(int i = 0 ; i < bufferSize ; i++)
    {
        chunkBuffer[i].setValues(chunkSize);
        chunkBuffer[i].setChunkNumber(i);
    }
}

// 类析构函数
VideoBuffer::~VideoBuffer()
{
    delete[] chunkBuffer;   // 保留一定数量块的视频缓冲区
}

// 根据缓冲区中可用的内容设置BufferMap数组
// BMap：要设置的BufferMap
void VideoBuffer::updateBufferMap(BufferMap* BMap)
{
    for(int i=0 ; i < bufferSize ; i++)
    {
        if(chunkBuffer[i].isComplete())  // 检查这个块是否填充了它的帧
            BMap->buffermap[i] = true;
        else
            BMap->buffermap[i] = false;
        BMap->chunkNumbers[i] = chunkBuffer[i].getChunkNumber();
    }
    BMap->setLastSetChunk(lastSetChunk);
}

// VideoBuffer的标准输出流，给出所有包含的块和帧
std::ostream& operator<<(std::ostream& os, const VideoBuffer& v)
{
    for (int i=0 ; i < v.bufferSize ; i++)
    {
        os << v.chunkBuffer[i];
        if((i+1)%v.gopSize == 0)
            os << std::endl;
    }
    return os;
}

// 将缓冲区移动一个块（一个块将被删除，新的空块将添加）
void VideoBuffer::shiftChunkBuf()
{
    for(int j=0 ; j < 1 ; j++)
    {
        for(int i=0 ; i < bufferSize-1 ; i++)
        {
            chunkBuffer[i] = chunkBuffer[i+1];
        }
        Chunk ch;
        ch.setValues(chunkSize);
        ch.setChunkNumber(chunkBuffer[bufferSize-2].getChunkNumber()+1);
        chunkBuffer[bufferSize-1] = ch;
    }
}

// 根据帧号在缓冲区中设置帧
// vFrame：视频帧设置
void VideoBuffer::setFrame(VideoFrame vFrame)
{
    int ExtractedChunkNum = vFrame.getFrameNumber()/(chunkSize);
    while(ExtractedChunkNum > chunkBuffer[bufferSize-2].getChunkNumber())
            shiftChunkBuf();

    if(ExtractedChunkNum >=  chunkBuffer[0].getChunkNumber() &&
            ExtractedChunkNum <= chunkBuffer[bufferSize-1].getChunkNumber())
    {
        chunkBuffer[ExtractedChunkNum - chunkBuffer[0].getChunkNumber()].setFrame(vFrame);
        if(lastSetFrame < vFrame.getFrameNumber())
            lastSetFrame = vFrame.getFrameNumber();
        if(chunkBuffer[ExtractedChunkNum - chunkBuffer[0].getChunkNumber()].isComplete())
            lastSetChunk = ExtractedChunkNum;
    }
    /*else
        std::cout << "(ChunkBuffer::setFrame) ChunkBuffer Out of boundary!!!!!!!!" << std::endl;*/
}

// 根据给定的帧编号给出视频帧
VideoFrame VideoBuffer::getFrame(int FrameNumber)
{
    int ExtractedChunkNum = FrameNumber/(chunkSize);
    if(ExtractedChunkNum >=  chunkBuffer[0].getChunkNumber() &&
            ExtractedChunkNum <= chunkBuffer[bufferSize-1].getChunkNumber())
        return chunkBuffer[ExtractedChunkNum - chunkBuffer[0].getChunkNumber()]
                           .chunk[FrameNumber%chunkSize].getVFrame();
    /*else
    {
        std::cout << chunkBuffer[0].getChunkNumber() << "  to  "<< chunkBuffer[bufferSize-1].getChunkNumber()<< std::endl;
        std::cout << "ExtractedChunkNum : "<<ExtractedChunkNum <<std::endl;
        std::cout << "(VideoBuffer::getFrame) ChunkBuffer Out of boundary!!!!!!!!" << std::endl;
    }*/
}

// 根据给定的块数给出一个块
// ChunkNumber：要查找的块号
Chunk VideoBuffer::getChunk(int ChunkNumber)
{
    if(ChunkNumber >=  chunkBuffer[0].getChunkNumber() &&
            ChunkNumber <= chunkBuffer[bufferSize-1].getChunkNumber())
    {
        return chunkBuffer[ChunkNumber - chunkBuffer[0].getChunkNumber()];
    }
    /*else
    {
        std::cout << chunkBuffer[0].getChunkNumber() << "  to  "<< chunkBuffer[bufferSize-1].getChunkNumber()<< std::endl;
        std::cout << "ChunkNum : "<<ChunkNumber <<std::endl;
        std::cout << "(ChunkBuffer:getChunk) Chunk Not Found !!!!!"<< std::endl;
    }*/
}

// 设置缓冲区中此节点接收的块
void VideoBuffer::setChunk(Chunk InputChunk)
{
    int index = InputChunk.getChunkNumber() - chunkBuffer[0].getChunkNumber();
    if(InputChunk.getChunkNumber() >=  chunkBuffer[0].getChunkNumber() &&
                InputChunk.getChunkNumber() <= chunkBuffer[bufferSize-1].getChunkNumber())
    {
        chunkBuffer[index] = InputChunk;
        lastSetChunk = chunkBuffer[index].getChunkNumber();
    }
    /*else
    {
        std::cout << chunkBuffer[0].getChunkNumber() << "  to  "<< chunkBuffer[bufferSize-1].getChunkNumber()<< std::endl;
        std::cout << "ChunkNum : "<<InputChunk.getChunkNumber() <<std::endl;
        std::cout << "(ChunkBuffer:setChunk) Chunk Not Found !!!!!"<< std::endl;
    }*/
}
```