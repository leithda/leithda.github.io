---
title: mTreebone代码解析-DenaCastApp
categories:
  - C++
tags:
  - 仿真
  - P2P
  - 视频
author: 长歌
abbrlink: 2077499524
date: 2019-10-26 17:00:00
---

关于DenaCastApp的解析
应用程序主类，程序的初始化及消息的处理都由这个类完成
<!-- more -->
# DenaCastApp
## 头文件 
### DenaCastApp.h
```c++
#include "BufferMap.h"
#include "VideoMessage_m.h"
#include <BaseApp.h>
#include <TransportAddress.h>
#include <BaseCamera.h>
#include <VideoBuffer.h>
#include "LocalVariables.h"

struct requesterNode
{
    TransportAddress tAddress; /**< 请求块的节点的传输地址*/
    short int chunkNo; /**< 请求节点的块号*/
};
struct nodeBufferMap
{
    TransportAddress tAddress; /**< 邻居的传输地址 */
    BufferMap buffermap; /**< 节点的BufferMap对象*/
    double totalBandwidth; /**< 节点的总带宽容量*/
    double freeBandwidth; /**< 定义时隙内邻居的空闲带宽*/
    int requestCounter; /**< 显示请求数量的数字 - 响应块数*/
    //double timeout;       // 跟踪从邻居收到的最后一个BM
};
struct chunkPopulation
{
    int chunkNum; /**< 块数 */
    std::vector <int>supplierIndex; /**< 保持neighborBufferMaps向量的供应商索引的向量*/
};
enum PlayingState
{
    PLAYING = 0,
    BUFFERING = 1,
    STOP = 2
};
class DenaCastApp : public BaseApp
{
public:
    /**
     * 初始化基类属性
     *
     * @param stage the init stage
     */
    virtual void initializeApp(int stage);
    virtual void finishApp();
    virtual void handleTimerEvent(cMessage* msg);
    virtual void handleLowerMessage(cMessage* msg);
    virtual void handleUpperMessage(cMessage* msg);
    /**
     * 从sendframes向量中删除帧编号（超时后）
     * @param framenum 要删除的帧号
     * @param sendFrames 已发送的帧矢量
     */
    virtual void deleteElement(int frameNum, std::vector<int>& sendFrames);
    /**
     * 广播缓冲区映射到所有节点邻居
     */
    virtual void bufferMapExchange();
    /**
     * 检查我们是否有足够的帧等于启动缓冲来播放
     */
    virtual void checkForPlaying();




protected:
    unsigned short int numOfBFrame; /**<“I”和“P”之间或两个“P”帧之间的B帧数 */
    /**
     * 缓冲区分为多少个块，块个数
     */
    unsigned short int bufferSize; /**<缓冲区中的块数 */
    /**
     * 每个缓冲区块保存多少帧视频数据，帧数/块
     */
    unsigned short int chunkSize; /**< 块中的帧数 */

    unsigned short int gopSize; /**< 一个GoP中可用的帧数（图片组）*/
    /**
     * 在何时将本地缓冲区数据及状态推送到其他节点
     */
    double bufferMapExchangePeriod;
    /**
     * 是否是视频服务器，true：是 false：不是
     */
    bool isVideoServer; 
    /**
     * 视频每秒需要多少帧视频数据
     */
    unsigned short int Fps; /**< 存储参数Fps，视频的每秒帧数*/

    /**
     * 存储最近发送给播放器模块的帧的帧号，用于判断是否需要后续拉取帧数据
     */
    int playbackPoint; 

    /**
     * 节点状态，感觉也可以称为服务器状态。当我们向播放器模块发送第一条消息时，它变为真实
     */
    PlayingState playingState;
    /**
     * 网络中的节点会定期向所有邻居节点发送BufferMap，用来将自己本地缓冲区当前的数据块状态通知给邻居节点。
     * 邻居节点在收到通知消息后，就可以根据需求周期性地请求自己所需的数据块。这个就是开始交换buffermap的标志
     */
    bool bufferMapExchangeStart;

    /**
     * 启动播放需要缓冲多少视频(单位:s，对应数据帧应为 startUpBuffering*Fps 帧)
     */
    double startUpBuffering;
    bool rateControl; /**< 如果是真正的使用率控制机制*/
    /**
     * 收集统计数据的时间
     */
    double measuringTime;

    /**
     * 选择接收方调度的编号，即调度算法编号
     */
    unsigned short int receiverSideSchedulingNumber; /**< 选择接收方调度的号码*/

    /**
     * 发送方调度的编号
     */
    unsigned short int senderSideSchedulingNumber; /**< 选择发送方调度的编号*/
    double averageChunkLength; /**< 平均一个长度*/
    bool schedulingSatisfaction; /**< 如果条件适用于开始计划，则为true*/

    bool meshPullActive;            // 如果要通过网格拉取请求块，则为true
    int lastSentChunk;              // 此节点最后发送给其子节点的块的块号
    int lastReceivedChunk;          // 节点最后从其父节点接收的块的块号


    /**
     * 用于保存邻居节点的视频缓冲数据及状态
     */
    std::vector <nodeBufferMap> neighborsBufferMaps;



    // 定义自身的消息(用于定时器处理)
    cMessage* bufferMapTimer; // timer消息将buffermap发送给邻居
    cMessage* requestChunkTimer; // 从邻居请求块的时间消息
    cMessage* sendFrameTimer; // 用于呼叫发送方调度的定时器
    cMessage* playingTimer; // 定时器，以1 / fps发送自我消息，向播放器发送缓冲帧

    cMessage* meshPullTimer;    // 用于检查并拉回回放点和lastReceivedChunk之间丢失块的计时器

    BaseCamera* bc; /**< 指向基础相机的指针*/
    LocalVariables* LV; /** < 指向LocalVariables模块的指针*/

    // 调度相关方法，由Scheduling.cc实现
    virtual void coolStreamingScheduling();
    virtual void recieverSideScheduling2();
    virtual void recieverSideScheduling3();
    virtual double getRequestFramePeriod();
    virtual int requestRateState();
    virtual void senderSideScheduling1();
    virtual void senderSideScheduling2();
    virtual void selectRecieverSideScheduling();
    virtual void selectSenderSideScheduling();
    virtual void handleChunkRequest(TransportAddress& SrcNode,int chunkNo, bool push);
    virtual void countRequest(TransportAddress& node);
    virtual void sendFrameToPlayer();
    virtual void deleteNeighbor(TransportAddress& node);
    virtual void sendframeCleanUp();
    virtual bool isMeasuring(int frameNo);
    virtual void resetFreeBandwidth();
    virtual void updateNeighborBMList(BufferMapMessage* BufferMap_Recieved);
    bool isInVector(TransportAddress& Node, std::vector <TransportAddress> &list);
    void checkAvailability_RateControlLoss();
    int getNextChunk();

    /**<
     *  矢量，其中保持发送帧以防止重复请求。
     */
    std::vector< int > sendFrames;
    /**<
     * 矢量，其中保持下一帧的帧用于请求但是它们不在邻居中（用于排除）。 收到BufferMap消息后，它将被清除
     */
    std::vector < int > notInNeighbors;
    /**<
     * 矢量用于保持邻居的bufferMap
     */
    std::vector <int> requestedChunks;
    std::vector <double> neighborHops;
    std::map <double,requesterNode> senderqueue;

    //statistics

    double stat_startupDelay;/**<
     * 在开始缓冲（选择流媒体频道）和开始播放之间经过的时间（随机变量）
     */
    double stat_startSendToPlayer; /**< 是时候开始向播放器模块发送帧了*/
    double stat_startBuffering; /**< 我们开始缓冲的时间（选择频道）*/
    double stat_startBufferMapExchange; /**< 节点开始交换bufferMap的时间*/
    double stat_TotalReceivedSize; /** < */
    double stat_RedundentSize; /** < */

    bool firstChunkReceived;    // 如果节点已收到其第一个块，则为true。
    double firstChunkTime;      // 节点收到第一个块的时间.
    double stat_FirstFrameToPlayerTime; // 节点收到第一帧直到将其发送给播放器的时间.
    double sumPlaybackLag;      // 对所有帧的回放延迟求和.
    int frameCount;             // 计算播放的帧数.
    double stat_PlaybackLag;    // 计算所有帧的平均回放延迟.
};

```

## 源文件
### DenaCastApp.cc
#### 初始化方法

```c
// 初始化基类属性
void DenaCastApp::initializeApp(int stage)
{
    if (stage != MIN_STAGE_APP)
        return;
    //初始化参数
    numOfBFrame = par("numOfBFrame");  // 2 作用未知 TODO
    bufferMapExchangePeriod = par("bufferMapExchangePeriod");  // 1s BufferMap交换时间间隔
    gopSize = par("gopSize");  // 12 作用未知 TODO
    /**
     * 如果节点的类型是2
     * 视频服务器节点标志置为true，否则为false
     */
    if(globalNodeList->getPeerInfo(thisNode.getAddress())->getTypeID() == 2)
        isVideoServer = true;
    else
        isVideoServer = false;

    startUpBuffering = par ("startUpBuffering");    // 10s 启动缓冲
    Fps = par("Fps");   // 25 作用未知 TODO frame percent second 每秒帧数
    double windowOfIntrest = par("windowOfIntrest");    // 40s 对等缓冲区中维护了多少视频
    chunkSize = par ("chunkSize");  // 2 分几块

    bufferSize = windowOfIntrest*Fps/chunkSize;     // (40s * 25帧/s) / 2块
    // 如果缓冲区大小(500)不能整除块个数
    if(bufferSize%chunkSize > 0)
            bufferSize += chunkSize - (bufferSize%chunkSize);
    numOfBFrame = par("numOfBFrame");  // 2 作用未知 TODO
    rateControl = par("rateControl");   // true 作用未知 TODO
    measuringTime = par("measuringTime");   // 20s 作用未知
    receiverSideSchedulingNumber = par("receiverSideSchedulingNumber"); // 接收方调度编号，算法2
    senderSideSchedulingNumber = par("senderSideSchedulingNumber"); // 发送方调度编号，算法1
    averageChunkLength = par("averageChunkLength"); // 130 平均块长度

    //初始化全局变量
    playbackPoint = -1;
    playingState = BUFFERING;
    bufferMapExchangeStart = false;
    schedulingSatisfaction = false;
    lastSentChunk = 0;
    lastReceivedChunk = 0;
    meshPullActive = true;

    //初始化自我消息 自带定时器
    bufferMapTimer = new cMessage("bufferMapTimer");
    requestChunkTimer = new cMessage("requestChunkTimer");
    playingTimer = new cMessage("playingTimer");
    sendFrameTimer = new cMessage("sendFrameTimer");
    meshPullTimer = new cMessage("meshPullTimer");

    if(!isVideoServer)
        bc = check_and_cast<BaseCamera*>(simulation.getModuleByPath("CDN-Server[1].tier2.mpeg4camera"));

    LV = check_and_cast<LocalVariables*>(getParentModule()->getSubmodule("localvariables"));

    //statistics   初始化监控变量
    stat_startupDelay = 0;
    stat_startSendToPlayer = 0;
    stat_startBuffering = 0;
    stat_startBufferMapExchange = 0;
    stat_TotalReceivedSize = 0;
    stat_RedundentSize = 0;

    firstChunkReceived = false;
    firstChunkTime = 0.0;
    stat_FirstFrameToPlayerTime = 0.0;

    sumPlaybackLag = 0.0;
    frameCount = 0;
    stat_PlaybackLag = 0.0;

    // 设置服务器和客户端显示字符串
    if(isVideoServer)
        getParentModule()->getParentModule()->setDisplayString("i=device/server;i2=block/circle_s");
    else
        getParentModule()->getParentModule()->setDisplayString("i=device/wifilaptop_vs;i2=block/circle_s");
}
```

#### 处理定时器消息
```c++
void DenaCastApp::handleTimerEvent(cMessage* msg)
{
    /**
     * 如果是发送视频消息
     * 选择调度算法，调用 Scheduling相关代码
     */
    if(msg == sendFrameTimer)
    {
        selectSenderSideScheduling();   
    }
    /**
     * 如果是请求视频块消息
     * 判断状态，如果没有停止，设置调度状态标志为true
     */
    else if(msg == requestChunkTimer)
    {
        if(playingState != STOP)
            schedulingSatisfaction = true;
    }
    /**
     * 如果是bufferMapT消息
     * 看不懂 TODO
     */
    else if(msg == bufferMapTimer)
    {
        bufferMapExchange();
    }
    /**
     * 如果是播放视频消息
     * 1. 播放状态如果是播放，发送视频帧到播放器
     * 2. 如果是缓冲状态，检查是否存在缓冲用于播放
     */
    else if(msg == playingTimer)
    {
        if(playingState == PLAYING){    // 1. 播放状态
            sendFrameToPlayer();    
        } else if (playingState == BUFFERING){    // 缓冲状态
            checkForPlaying();
        }
        // TODO
        scheduleAt(simTime()+1/(double)Fps,playingTimer);

    }
    /**
     * 如果是拉去视频帧消息(非播放，应该是视频分发)
     * 如果 1. meshPullActive(需要拉取视频流) || !LV->hasTreeboneParent(没有父节点) ||lastReceivedChunk*chunkSize < playbackPoint(从父节点接收的的最后的块数*几块 小于 最近发送给播放器的帧) 
     *      2. 如果当前不需要拉取视频，检查播放点和最后收到的视频块之间丢失的视频块
     */
    else if (msg == meshPullTimer) {
        // 定期检查是否需要将网块拉过网格。
        if (meshPullActive || !LV->hasTreeboneParent || lastReceivedChunk*chunkSize < playbackPoint) {
            meshPullActive = true;
            selectRecieverSideScheduling();
        }
        else {
            // 检查播放点和lastReceivedChunk之间的丢失
            int inCompleteChunks = 0;
            int playbackIndex = playbackPoint/chunkSize - LV->hostBufferMap->chunkNumbers[0];
            int lastReceivedChunkIndex = lastReceivedChunk - LV->hostBufferMap->chunkNumbers[0];
            if (playbackIndex >= 0 && playbackIndex < bufferSize && lastReceivedChunkIndex >= 0 && lastReceivedChunkIndex < bufferSize) {
                for (int i=playbackIndex; i<lastReceivedChunkIndex; i++)
                    if (!LV->hostBufferMap->buffermap[i])
                        inCompleteChunks++;
                if (inCompleteChunks > 10) // 缺失的块大于10
                    selectRecieverSideScheduling();
            }
        }
        scheduleAt(simTime()+10/(double)Fps,meshPullTimer); // 开启定时器【拉取视频定时器】
    }
    else
        delete msg;

}
```

#### 处理子节点消息
```c++
// 处理下层消息
void DenaCastApp::handleLowerMessage(cMessage* msg)
{
    /**
     * 转换消息
     * 1. 如果是视频相关消息
     *  1.1 如果是请求视频块消息
     *      获取请求方地址及请求块编号，将请求方请求块及地址插入发送队列中，删除当前消息
     *  1.2 如果是视频块响应并且当前节点不是服务器节点
     *      如果第一次收到此消息，记录响应时间
     *      如果不是更新缓冲区的响应(说明是播放时请求)，开启定时器，向播放器发送缓冲帧。
     *      如果数据传输完成且没有错误，更新拉取视频标志，更新监控变量，命中次数+1，推送数据到子节点
     *  1.3 邻居离开消息
     *      删除邻居并判断离开邻居是否是视频来源，如果是，将拉取视频标志置为true
     *  1.4 当前节点离开消息
     *      更新状态
     * 2. 如果是缓冲区相关消息
     */
    if (dynamic_cast<VideoMessage*>(msg) != NULL)
    {
        VideoMessage* VideoMsg=dynamic_cast<VideoMessage*>(msg);
        // 1. 如果是请求视频块
        if(VideoMsg->getCommand() == CHUNK_REQ) {
            requesterNode rN;
            rN.tAddress = VideoMsg->getSrcNode();   // 获取到消息来源IP
            rN.chunkNo = VideoMsg->getChunk().getChunkNumber(); // 获取请求视频块编号
            senderqueue.insert(std::make_pair<double,requesterNode>(VideoMsg->getDeadLine(),rN));   // 封装 VideoMsg。getDeadLine() 及IP 放入发送队列 TODO(猜测DeadLine是请求相关信息)
            selectSenderSideScheduling();   // 设置调度算法
            delete msg;
        }
        // 2. 如果是视频块响应并且当前节点不是视频服务器节点
        else if(VideoMsg->getCommand() == CHUNK_RSP && !isVideoServer)
        {
            if (!firstChunkReceived) {  // 判断是否第一次收到响应
                firstChunkReceived =true;
                firstChunkTime = simTime().dbl();   // 记录第一次收到响应时间
            }
            if(!bufferMapExchangeStart) // 如果不是开始更新缓冲区
            {
                scheduleAt(simTime(),requestChunkTimer);    // 定时器 从邻居节点请求数据库 开启
                scheduleAt(simTime(),playingTimer); // 定时器 以 1/fps 帧发送播放消息，向播放器发送缓冲帧
                scheduleAt(simTime()+bufferMapExchangePeriod+uniform(0,0.25),bufferMapTimer);   // 开启定时器，定时将bufferMap信息发送到邻居
                stat_startBufferMapExchange = simTime().dbl() + bufferMapExchangePeriod;
                stat_startBuffering = simTime().dbl();  // 记录开始缓冲的时间
                bufferMapExchangeStart=true;    // 节点交换缓冲区标志置为ture
                int shiftnum = 0;
                if(VideoMsg->getChunk().getChunkNumber() > 0)
                    shiftnum = VideoMsg->getChunk().getChunkNumber();
                shiftnum-=2;
                for(int i=0 ; i<shiftnum ; i++)
                    LV->videoBuffer->shiftChunkBuf();   // 向缓冲区加入一个块，新的空块会被加入
                LV->updateLocalBufferMap(); // TODO 
            }

            bool redundantState = false;    // 冗余状态，是否接收了多余的数据？ TODO
            /**
             * 如果块数据传输完成并且没有错误
             */
            if(VideoMsg->getChunk().isComplete() && !VideoMsg->hasBitError())
            {
                int shiftnum = VideoMsg->getChunk().getChunkNumber() - LV->videoBuffer->chunkBuffer[bufferSize-1].getChunkNumber();
                /**
                 * 加入空的缓冲块
                 */
                for(int i=0 ; i<shiftnum ; i++)
                    LV->videoBuffer->shiftChunkBuf();
                LV->updateLocalBufferMap();

                // 检查收到的块数据是否冗余
                if(LV->hostBufferMap->findChunk(VideoMsg->getChunk().getChunkNumber()))
                {
                    stat_RedundentSize += VideoMsg->getChunk().getChunkByteLength();
                    redundantState = true;
                }

                stat_TotalReceivedSize += VideoMsg->getChunk().getChunkByteLength();    // 监控变量，更新总的接收数据数目

                /**
                 * 设置数据块到缓冲区，并将数据块的命中次数HopCout + 1
                 */
                Chunk InputChunk = VideoMsg->getChunk();
                InputChunk.setHopCout(InputChunk.getHopCount()+1);
                LV->videoBuffer->setChunk(InputChunk);
                LV->updateLocalBufferMap();

                /**
                 * 判断视频数据是否推送完成
                 *  如果完成，记录收到的最后数据块的编号，将拉取数据标志置为fasle
                 *  否则，释放sendFrames中对应的数据？ TODO
                 */
                if (VideoMsg->getIsPushed()) {
                    lastReceivedChunk = InputChunk.getChunkNumber();
                    meshPullActive = false;
                }
                else
                    deleteElement(VideoMsg->getChunk().getChunkNumber(),sendFrames);

                /**
                 * 如果最后的视频数据是否释放(TODO 具体是什么意思？)
                 * 添加日志输出；计算由于数据迟到而造成的丢失的帧的大小
                 */
                if(isMeasuring(VideoMsg->getChunk().getLastFrameNo()))
                {
                    globalStatistics->addStdDev("DenaCastApp: end to end delay", simTime().dbl() - VideoMsg->getChunk().getCreationTime());
                    globalStatistics->addStdDev("DenaCastApp: Hop Count", VideoMsg->getChunk().getHopCount());
                    if(VideoMsg->getChunk().getLastFrameNo() < playbackPoint)
                        LV->addToLateArivalLoss(VideoMsg->getChunk().getLateArrivalLossSize(playbackPoint));
                }

                // 如果数据没有冗余(如果冗余，说明本地缓冲区已经有了相关数据(即：之前已经推送过这些数据)，避免接收重复数据)
                if (!redundantState) {
                    // 将收到的块推送给孩子。
                    for (unsigned int i=0; i < LV->treeboneChildren.size(); i++) {
                        handleChunkRequest(LV->treeboneChildren[i],LV->videoBuffer->lastSetChunk,true);
                        lastSentChunk = LV->videoBuffer->lastSetChunk;
                    }
                }
            }
            delete msg;
        }
        // 3. 邻居离开消息
        else if(VideoMsg->getCommand() == NEIGHBOR_LEAVE) {
            
            /* 删除当前邻居节点，判断当前节点的视频消息是否是离开的邻居节点
             * 如果是，需要重新拉取数据，将拉取视频标志置为true
             * */
            deleteNeighbor(VideoMsg->getSrcNode());
            if (LV->hasTreeboneParent && VideoMsg->getSrcNode() == LV->treeboneParent)
                meshPullActive = true;
            delete VideoMsg;
        }
        // 4. 当前节点离开
        else if(VideoMsg->getCommand() == LEAVING)
        {
            playingState = STOP;
            schedulingSatisfaction = false;
            delete VideoMsg;
        }
        else
            delete VideoMsg;
    }
    // 如果是缓冲区相关消息
    else if(dynamic_cast <BufferMapMessage*> (msg) != NULL)
    {
        BufferMapMessage* BufferMap_Recieved=dynamic_cast<BufferMapMessage*>(msg);

        // 如果缓冲区交换开关为false并且当前节点不是服务器节点
        if(!bufferMapExchangeStart && !isVideoServer)
        {
                /**
                 * 开启定时器
                 * 1. 从邻居拉取数据块 2. 播放(推送缓冲区帧到播放器)3. 缓冲区交换消息 4. 拉取视频
                 */
                scheduleAt(simTime(),requestChunkTimer);
                scheduleAt(simTime(),playingTimer);
                scheduleAt(simTime()+bufferMapExchangePeriod,bufferMapTimer);
                scheduleAt(simTime(), meshPullTimer);
                
                // 更新监控变量 缓冲区耗时
                stat_startBufferMapExchange = simTime().dbl() + bufferMapExchangePeriod;
                stat_startBuffering = simTime().dbl();
                bufferMapExchangeStart = true;
                /**
                 * 判断视频是否包含来源
                 * 1. 如果有的话，将拉取标志置为false
                 * 2. 否则，
                 */
                if (LV->hasTreeboneParent)
                    meshPullActive = false;
                if (meshPullActive) {
                    int shiftnum = 0;
                    if(BufferMap_Recieved->getBuffermap().getLastSetChunk() > 0)
                        shiftnum = BufferMap_Recieved->getBuffermap().getLastSetChunk();
                    if(shiftnum%gopSize > 0 && chunkSize%gopSize !=0)
                        shiftnum += gopSize - (shiftnum%gopSize);
                    else
                        shiftnum -= 2;
                    for(int i=0 ; i<shiftnum ; i++)
                        LV->videoBuffer->shiftChunkBuf();
                }
                LV->updateLocalBufferMap();
        }
        // 重新建立邻居网络
        notInNeighbors.clear();
        updateNeighborBMList(BufferMap_Recieved);

        // 如果调度状态标志为true并且拉取数据块标志为true,选择接收方调度算法
        if(schedulingSatisfaction && meshPullActive)
            selectRecieverSideScheduling();
        delete msg;
    }
    else
        delete msg;
}
```

#### 处理上层消息
```c++
void DenaCastApp::handleUpperMessage(cMessage* msg)
{
    // 如果是视频相关消息
    if (dynamic_cast<VideoMessage*>(msg) != NULL)
    {
        VideoMessage* VideoMsg=dynamic_cast<VideoMessage*>(msg);
        /**
         * 如果是来自摄像头消息
         */
        if (VideoMsg->getCommand() == CAMERA_MSG)
        {
            // 如果当前缓冲区没有交换，开启交换并设置标志
            if(!bufferMapExchangeStart)
            {
                scheduleAt(simTime()+bufferMapExchangePeriod,bufferMapTimer);
                bufferMapExchangeStart=true;
            }
            // 在缓冲区中设置帧 TODO
            LV->videoBuffer->setFrame(VideoMsg->getVFrame());
            LV->updateLocalBufferMap();
            delete msg;

            // 将服务器摄像头生成的块吐送给子节点，如果摄像头数据块ID不等于最后发送的块ID(有数据未推送)
            if (lastSentChunk != LV->videoBuffer->lastSetChunk) {
                for (unsigned int i=0; i < LV->treeboneChildren.size(); i++) {
                    handleChunkRequest(LV->treeboneChildren[i],LV->videoBuffer->lastSetChunk,true);
                }
                lastSentChunk = LV->videoBuffer->lastSetChunk;
            }
        }
    }
    else
        delete msg;
}
```

#### 更新邻居节点
```c++
// 更新邻居节点列表
void DenaCastApp::updateNeighborBMList(BufferMapMessage* BufferMap_Recieved)
{
    /**
     * 判断当前列表有没有
     * 如果没找到，加入到当前邻居列表的底部
     */
    bool find = false;
    for(unsigned int i=0 ; i<neighborsBufferMaps.size();i++)
        if(neighborsBufferMaps[i].tAddress == BufferMap_Recieved->getSrcNode())
        {
            neighborsBufferMaps[i].buffermap = BufferMap_Recieved->getBuffermap();
            neighborsBufferMaps[i].totalBandwidth = BufferMap_Recieved->getTotalBandwidth();
            find = true;
            break;
        }
    if(!find)
    {
        nodeBufferMap neighbourBM;
        neighbourBM.buffermap.setValues(chunkSize);
        neighbourBM.tAddress = BufferMap_Recieved->getSrcNode();
        neighbourBM.totalBandwidth =  BufferMap_Recieved->getTotalBandwidth();
        neighbourBM.requestCounter = 0;
        neighbourBM.buffermap = BufferMap_Recieved->getBuffermap();
        //neighbourBM.timeout = simTime().dbl();
        neighborsBufferMaps.push_back(neighbourBM);
    }
}
```

#### 删除发送帧元素
```c++
/**
 * 删除发送帧中元素
 * @param frameNum   [帧序号]
 * @param sendframes [发送帧参数]
 */
void DenaCastApp::deleteElement(int frameNum, std::vector<int>& sendframes)
{
    for (unsigned int i=0; i!=sendframes.size(); i++)
    {
        if (sendframes[i] == frameNum)
        {
            sendframes.erase(sendframes.begin()+i,sendframes.begin()+1+i);
            break;
        }
    }
}
```

#### 应用结束
```c++
// 应用结束
void DenaCastApp::finishApp()
{
    // 取消定时器
    cancelAndDelete(bufferMapTimer);
    cancelAndDelete(requestChunkTimer);
    cancelAndDelete(playingTimer);
    cancelAndDelete(sendFrameTimer);
    cancelAndDelete(meshPullTimer);
    
    // 设置统计数据
    if(stat_startupDelay != 0)
    {
        globalStatistics->addStdDev("DenaCastApp: Startup Delay", stat_startupDelay);
        globalStatistics->recordOutVector("DenaCastApp: Startup Delay_vec", stat_startupDelay);
    }
    if(stat_startSendToPlayer != 0)
        globalStatistics->addStdDev("DenaCastApp: start to send to player time", stat_startSendToPlayer);
    if(stat_startBuffering != 0)
        globalStatistics->addStdDev("DenaCastApp: start to buffering time", stat_startBuffering);
    if(stat_startBufferMapExchange != 0)
        globalStatistics->addStdDev("DenaCastApp: start exchanging bufferMap", stat_startBufferMapExchange);
    if(stat_TotalReceivedSize != 0)
        globalStatistics->addStdDev("DenaCastApp: Frame Redundancy",stat_RedundentSize/stat_TotalReceivedSize *100);

    if(stat_FirstFrameToPlayerTime != 0)
        globalStatistics->addStdDev("DenaCastApp: First Frame received to player time", stat_FirstFrameToPlayerTime);

    //播放延迟的统计收集：取所有帧的播放延迟的平均值。
    if (frameCount >0) {
        stat_PlaybackLag = sumPlaybackLag/frameCount;
        globalStatistics->addStdDev("DenaCastApp: Playback Lag", stat_PlaybackLag);
    }
}
```

### Scheduling.cc
#### 选择接受方调度算法
```c++
// Scheduling.cc

/**
 * [DenaCastApp::selectRecieverSideScheduling 选择接受方调度算法]
 */
void DenaCastApp::selectRecieverSideScheduling()
{ // Receive
    switch(receiverSideSchedulingNumber)  // 接收方调度号
    {
    case 1:
        coolStreamingScheduling();  //
        break;
    case 2:
        recieverSideScheduling2();
        break;
    case 3:
        recieverSideScheduling3();
        break;
    default:
        recieverSideScheduling2();
        break;
    }
}
```
