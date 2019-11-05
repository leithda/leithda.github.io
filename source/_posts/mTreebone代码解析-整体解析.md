---
title: mTreebone代码解析-整体解析
categories:
  - C++
tags:
  - 仿真
  - P2P
  - 视频
author: 长歌
abbrlink: 1449101074
date: 2019-11-05 19:25:00
---

对mTreebone设计上的整体把握
<!-- More -->

# 视频数据组成
## VedioFrame 类
- 视频帧类，视频数据的最小单位

### 属性
```c++
    char frameType;// 视频帧的类型
    int frameNumber;// 帧号（每个视频帧的唯一标识符）
    short int frameLength;// 视频帧的大小，以字节为单位
    float creationTime;// 创建此帧数据的SimTime
```
- 主要属性主要有四个，帧类型、帧ID、大小、创建时间

### 方法
1. 构造函数
```c++
VideoFrame::VideoFrame()
{
    setFrame(-1,-1, 'N',-1);
}

VideoFrame::VideoFrame(int FrameNumber, int FrameLength, char FrameType,double CreationTime)
{
    setFrame(FrameNumber, FrameLength,FrameType,CreationTime);
}
```

2. 赋值方法
```c++
void VideoFrame::setFrame(VideoFrame vFrame)
{
    frameNumber = vFrame.getFrameNumber();    // 视频帧号
    frameLength = vFrame.getFrameLength();    // 视频帧大小（字节）
    frameType = vFrame.getFrameType();        // 视频帧类型（I，P或B）
    creationTime = vFrame.getCreationTime();  // 创建时间
}
```

## Chunk 类
- 视频块类，由视频帧数组组成，网络中传输使用视频块

### 属性
```c++
    short int chunkNumber; // 视频块编号
    short int chunkSize; // 块对象中的帧数
    short int hopCount; // 遍历网络的块的跳数
    double creationTime; // 创建块的时间。 它用于计算端到端延迟

    VideoFrame* chunk; // 将视频帧保持为块的数组   
```

### 主要方法
1. 初始化方法
```c++
void Chunk::setValues(int ChunkSize)
{
    chunkSize = ChunkSize;                  // 块对象中的帧数
    chunk = new VideoFrame[chunkSize];      // 实例化视频帧数组
}
```

2. 设置帧数据
```c++
// 根据数字在块数组中设置一个帧
void Chunk::setFrame(VideoFrame vFrame)
{
    // 余数操作，举个栗子:每个块保存4帧视频帧，帧号为10的视频帧应该保存在第3个块的帧数组下标[2]中
    int ExtractedFrameNum = vFrame.getFrameNumber()%chunkSize;  
    chunk[ExtractedFrameNum].setFrame(vFrame);
}
```

3. 计算到达帧数
```c++
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

## VideoBuffer 类
- 视频缓冲类，由视频块组成的视频缓冲对象

### 属性
```c++
protected:
    int numOfBFrame; // 'I'和'P'之间或两个'P'帧之间的B帧数;
    int bufferSize; // 缓冲区中可用的块数
    int chunkSize; // 块中可用的帧数
public:
    int lastSetChunk; // 最近设置的最后一块
    int lastSetFrame; // 最近设置的最后一帧
    int gopSize; // 每组图片的帧数
    Chunk* chunkBuffer; // 保留一定数量块的视频缓冲区
```

### 主要方法
1. 构造函数
```c++
/**
 * [VideoBuffer::VideoBuffer 类构造函数]
 * @param NumOfBFrame [I和P之间或两个P帧之间的B帧数]
 * @param BufferSize  [缓冲区大小]
 * @param ChunkSize   [缓冲区中可用的块数]
 * @param GopSize     [每组图片的帧数]
 */
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
```

2. 移动块缓冲区
```c++
/**
 * [VideoBuffer::shiftChunkBuf 移动块缓冲区]
 */
void VideoBuffer::shiftChunkBuf()
{
    for(int j=0 ; j < 1 ; j++)
    {
        /**
         * 将所有块前移1位，块[0]删除
         */
        for(int i=0 ; i < bufferSize-1 ; i++)
        {
            chunkBuffer[i] = chunkBuffer[i+1];
        }
        // 设置最后一块
        Chunk ch;
        ch.setValues(chunkSize);    // 视频帧数量
        // 设置块编号为远最后视频块编号+1
        ch.setChunkNumber(chunkBuffer[bufferSize-2].getChunkNumber()+1);    
        chunkBuffer[bufferSize-1] = ch;
    }
}
```

3. 加入视频帧
```c++
/**
 * [VideoBuffer::setFrame 设置视频帧]
 * @param vFrame [视频帧]
 */
void VideoBuffer::setFrame(VideoFrame vFrame)
{
    /**
     * 举个栗子： 当前缓冲区状态[16,17,18,19,20] 每块可用帧数为5
     * 要设置的视频帧编号为 112 所属块编号应该为 112/5 = 22
     * 将缓冲区前移直到[19,20,21,22,23] 此时应加入的块下标为 22-19 = 3
     * chunkBuffer[3].setFrame(vFrame)
     */
    // 计算新加入的视频帧所属块编号
    int ExtractedChunkNum = vFrame.getFrameNumber()/(chunkSize);

    // 如果块编号大于当前倒数第二块的编号，往前移动缓冲区，直到等于
    while(ExtractedChunkNum > chunkBuffer[bufferSize-2].getChunkNumber())
            shiftChunkBuf();

    // 如果块编号大于等于第一块编号，并且小于等于最后一块编号
    if(ExtractedChunkNum >=  chunkBuffer[0].getChunkNumber() &&
            ExtractedChunkNum <= chunkBuffer[bufferSize-1].getChunkNumber())
    {
        // 块数组中相应的块加入这个视频帧
        chunkBuffer[ExtractedChunkNum - chunkBuffer[0].getChunkNumber()].setFrame(vFrame);
        // 如果最后设置的帧编号小于这个帧，赋值为它
        if(lastSetFrame < vFrame.getFrameNumber())
            lastSetFrame = vFrame.getFrameNumber();
        // 如果加入帧的这个块已经加载完成，设置最后修改的块编号为此块编号
        if(chunkBuffer[ExtractedChunkNum - chunkBuffer[0].getChunkNumber()].isComplete())
            lastSetChunk = ExtractedChunkNum;
    }
    /*else
        std::cout << "(ChunkBuffer::setFrame) ChunkBuffer
    Out of boundary!!!!!!!!" << std::endl;*/
}
```

## BufferMap 类
- VideoBuffer 视频缓冲区的补充类，保存视频缓冲区中视频块是否可用，最后设置的块编号，对外进行缓冲区操作时，使用此类

### 属性
```c++
protected:
    short int lastSetChunk; // 缓冲区中设置的最后一个块
    short int bufferSize; // 视频缓冲区中存在的块数
public:
    bool* buffermap;  // 布尔数组，显示视频缓冲区中块的可用性
    short int* chunkNumbers; // 视频缓冲区中的块的编号
```

### 主要方法
1. 初始化方法
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

2. 查找缓冲区中未设置的块
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
