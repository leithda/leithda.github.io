---
title: mTreebone代码解析-BufferMap
categories:
  - C++
tags:
  - 仿真
  - P2P
  - 视频
author: 长歌
abbrlink: 3152553048
date: 2019-10-29 21:00:00
---

关于 BufferMap 的解析
保存视频缓冲区信息
<!-- more -->

# BufferMap
## 头文件
```c++
#include <vector>
#include <iostream>

class BufferMap
{
protected:
    short int lastSetChunk; // 缓冲区中设置的最后一个块
    short int bufferSize; // 视频缓冲区中存在的块数
public:
    bool* buffermap;  // 布尔数组，显示视频缓冲区中块的可用性
    short int* chunkNumbers; //  视频缓冲区中的块的编号
    /*
     * 根据其编号查找块
     *
     * @param ChunkNumber 要检查的块号
     * @return 如果存在返回true
     */
    virtual bool findChunk(int ChunkNumber);
    /**
     * BufferMap的标准输出流，打印缓冲区映射及其关联的块号
     *
     * @param os the ostream
     * @param b the BufferMap
     * @return the output stream
     */
    friend std::ostream& operator<<(std::ostream& os, const BufferMap& b);
    /**
     * 初始化变量的默认值
     *
     * @param BufferSize 视频缓冲区中可用的块数
     */
    void setValues(int BufferSize);
    /**
     * 检索未在播放顺序中设置的下一个块
     *
     * @param sendFrames 保持请求的块但不在缓冲区中的向量（应排除它们）
     * @param notInNeighbors 之前返回的块，但它们在邻居中不可用
     * @param playbackPoint 正在播放的帧
     * @return integer 要检查请求的块号
     */
    int getNextUnsetChunk(std::vector<int>& sendFrames ,
                std::vector<int>& notInNeighbors, int playbackPoint);
    /**
     * 获取视频缓冲区中设置的最后一个块
     *
     * @return integer 在视频缓冲区中设置的最后一个块
     */
    int getLastSetChunk(){return lastSetChunk;}
    /**
     * 设置最后一个块的值
     *
     * @param LastSetChunk 为最后设置的块设置的值
     */
    void setLastSetChunk(int LastSetChunk);
    /**
     * 计算缓冲区映射对象的大小以便在数据包中进行设置
     *
     * @return integer 比特大小
     */
    int getBitLength();
};
```

## 源文件

### 设置最后一个块的值
```c++
// 设置最后一个块的值
void BufferMap::setLastSetChunk(int LastSetChunk)
{
    lastSetChunk = LastSetChunk;  // 为最后设置的块设置的值
}
```

### 初始化变量的默认值
```c++
/**
 * [BufferMap::setValues 初始化方法]
 * @param BufferSize [缓冲区可用块数]
 */
void BufferMap::setValues(int BufferSize)
{
    bufferSize = BufferSize;
    lastSetChunk = 0;
    buffermap = new bool[bufferSize];
    chunkNumbers = new short int[bufferSize];
    for(int i = 0 ; i < bufferSize ; i++)
    {
        buffermap[i] = false;
        chunkNumbers[i]=i;
    }
}
```

### 根据编号查找对应块是否存在
```c++
/**
 * 根据编号查找视频数据块
 * @param  ChunkNumber [要检查的块号]
 * @return             [是否存在]
 */
bool BufferMap::findChunk(int ChunkNumber)
{
    if(ChunkNumber - chunkNumbers[0] >= bufferSize || ChunkNumber - chunkNumbers[0] < 0)
        return false;
    if(buffermap[ChunkNumber - chunkNumbers[0]])
        return true;
    else
        return false;
}
```

### 标准输出流实现
```c++
std::ostream& operator<<(std::ostream& os, const BufferMap& b)
{
    for(int i = 0 ; i < b.bufferSize ; i++)
    {
        os << b.chunkNumbers[i] << ": " << b.buffermap[i] << "  ";
        if((i+1)%12 == 0)   // 每输出12个换行
            os << std::endl;    
    }
    return os;
}
```

### 获取下一个未设置的块
```c++
/**
 * [BufferMap::getNextUnsetChunk 查找缓冲区中下一个未设置的帧编号]
 * @param  sendFrames     [请求中的块]
 * @param  notInNeighbors [有返回但不在邻居节点中的块]
 * @param  playbackPoint  [正在播放的帧编号]
 * @return                [帧编号]
 */
int BufferMap::getNextUnsetChunk(std::vector<int>& sendFrames ,
                std::vector<int>& notInNeighbors, int playbackPoint)
{
    int unSetChunk = -1;
    bool c1 = true;
    bool c2 = true;
    // 遍历缓冲区
    for(int i=0 ; i<bufferSize ; i++)
    {
        unSetChunk = chunkNumbers[i];
        // 如果正在播放的块比当前块编号大
        if(playbackPoint > unSetChunk)
            continue;
        c1=true;
        c2=true;
        // 如果当前块不可用
        if(!buffermap[i])
        {
          
            for (unsigned int k=0; k!=sendFrames.size(); k++)
            {
                // 判断是否请求过  
                if (sendFrames[k] == unSetChunk)
                {
                    c1=false;
                    break;
                }
            }
            if(c1)
                for (unsigned int k=0; k!=notInNeighbors.size(); k++)
                {  
                    // 判断是否收到过此块的响应
                    if (notInNeighbors[k] == unSetChunk)
                    {
                        c2=false;
                        break;
                    }
                }
            if(c1 && c2)    // 未请求过，并且也没有返回过响应数据。表明真的未设置
                return unSetChunk;
        }
    }
    return -1;
}
```


### 计算缓冲区映射对象的大小以便在数据包中进行设置
```c++
int BufferMap::getBitLength()
{
    return bufferSize+16+16+16;   // 视频缓冲区中存在的块数
}

```