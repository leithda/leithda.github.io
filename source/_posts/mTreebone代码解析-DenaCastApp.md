---
title: mTreebone代码解析-DenaCastApp
categories:
  - C++
tags:
  - 仿真
  - P2P
  - 视频
author: 长歌
password: lmd19930317
abstract: 加密文章，非请勿入~
message: 请输入名字首字母加生日
abbrlink: 2077499524
date: 2019-10-26 17:00:00
---

密码文件
<!-- more -->
# DenaCastApp
## 头文件
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
	//double timeout;		// 跟踪从邻居收到的最后一个BM
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
    unsigned short int bufferSize; /**<缓冲区中的块数 */
	unsigned short int chunkSize; /**< 块中的帧数 */
	unsigned short int gopSize; /**< 一个GoP中可用的帧数（图片组）*/
    double bufferMapExchangePeriod; /**<我们将BufferMap推送到邻居的时间段 */
    bool isVideoServer; /**<store参数isVideoServer，如果为true则为视频服务器 */
    unsigned short int Fps; /**< 存储参数Fps，视频的每秒帧数*/
    int playbackPoint; /**< 存储最近发送给播放器模块的帧的帧号*/
    PlayingState playingState; /**< 当我们向播放器模块发送第一条消息时，它变为真实*/
    bool bufferMapExchangeStart; /**< 作为收到的第一个bufferMap，我们也开始交换我们的bufferMap，它变得真实*/
    double startUpBuffering; /** < 为了播放，应该缓冲多少视频（秒）*/
    bool rateControl; /**< 如果是真正的使用率控制机制*/
    double measuringTime; /**< 我们开始收集统计数据的SimTime*/
    unsigned short int receiverSideSchedulingNumber; /**< 选择接收方调度的号码*/
    unsigned short int senderSideSchedulingNumber; /**< 选择发送方调度的编号*/
    double averageChunkLength; /**< 平均一个长度*/
    bool schedulingSatisfaction; /**< 如果条件适用于开始计划，则为true*/

	bool meshPullActive;			// 如果要通过网格拉取请求块，则为true
	int lastSentChunk;				// 此节点最后发送给其子节点的块的块号
	int lastReceivedChunk;	 		// 节点最后从其父节点接收的块的块号


    std::vector <nodeBufferMap> neighborsBufferMaps;



	// self-messages
	cMessage* bufferMapTimer;/**< timer消息将buffermap发送给邻居 */
	cMessage* requestChunkTimer; /**< 从邻居请求块的时间消息 */
	cMessage* sendFrameTimer; /**< 用于呼叫发送方调度的定时器 */
	cMessage* playingTimer; /**< 定时器，以1 / fps发送自我消息，向播放器发送缓冲帧 */

	cMessage* meshPullTimer;	// 用于检查并拉回回放点和lastReceivedChunk之间丢失块的计时器

	BaseCamera* bc; /**< 指向基础相机的指针*/
	LocalVariables* LV; /** < 指向LocalVariables模块的指针*/

	//Scheduling
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

	bool firstChunkReceived; 	// 如果节点已收到其第一个块，则为true。
	double firstChunkTime; 		// 节点收到第一个块的时间.
	double stat_FirstFrameToPlayerTime; // 节点收到第一帧直到将其发送给播放器的时间.
	double sumPlaybackLag;		// 对所有帧的回放延迟求和.
	int frameCount;				// 计算播放的帧数.
	double stat_PlaybackLag;	// 计算所有帧的平均回放延迟.
};

```

## 源文件

### 初始化方法

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

### 处理定时器消息
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
        scheduleAt(simTime()+10/(double)Fps,meshPullTimer);
    }
    else
        delete msg;

}
```

### 处理子节点消息
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
     *  1.3 邻居离开消息
     *  1.4 当前节点离开消息
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
            if(!bufferMapExchangeStart) // 如果
            {
                scheduleAt(simTime(),requestChunkTimer);
                scheduleAt(simTime(),playingTimer);
                scheduleAt(simTime()+bufferMapExchangePeriod+uniform(0,0.25),bufferMapTimer);
                stat_startBufferMapExchange = simTime().dbl() + bufferMapExchangePeriod;
                stat_startBuffering = simTime().dbl();
                bufferMapExchangeStart=true;
                int shiftnum = 0;
                if(VideoMsg->getChunk().getChunkNumber() > 0)
                    shiftnum = VideoMsg->getChunk().getChunkNumber();
                shiftnum-=2;
                for(int i=0 ; i<shiftnum ; i++)
                    LV->videoBuffer->shiftChunkBuf();
                LV->updateLocalBufferMap();
            }

            bool redundantState = false;
            if(VideoMsg->getChunk().isComplete() && !VideoMsg->hasBitError())
            {
                int shiftnum = VideoMsg->getChunk().getChunkNumber() - LV->videoBuffer->chunkBuffer[bufferSize-1].getChunkNumber();
                for(int i=0 ; i<shiftnum ; i++)
                    LV->videoBuffer->shiftChunkBuf();
                LV->updateLocalBufferMap();

                if(LV->hostBufferMap->findChunk(VideoMsg->getChunk().getChunkNumber()))
                {
                    stat_RedundentSize += VideoMsg->getChunk().getChunkByteLength();
                    redundantState = true;
                }
                stat_TotalReceivedSize += VideoMsg->getChunk().getChunkByteLength();
                Chunk InputChunk = VideoMsg->getChunk();
                InputChunk.setHopCout(InputChunk.getHopCount()+1);
                LV->videoBuffer->setChunk(InputChunk);
                LV->updateLocalBufferMap();

                // 跟踪被推送的块，指示活动的父节点.
                if (VideoMsg->getIsPushed()) {
                    lastReceivedChunk = InputChunk.getChunkNumber();
                    meshPullActive = false;
                }
                else
                    deleteElement(VideoMsg->getChunk().getChunkNumber(),sendFrames);

                if(isMeasuring(VideoMsg->getChunk().getLastFrameNo()))
                {
                    globalStatistics->addStdDev("DenaCastApp: end to end delay", simTime().dbl() - VideoMsg->getChunk().getCreationTime());
                    globalStatistics->addStdDev("DenaCastApp: Hop Count", VideoMsg->getChunk().getHopCount());
                    if(VideoMsg->getChunk().getLastFrameNo() < playbackPoint)
                        LV->addToLateArivalLoss(VideoMsg->getChunk().getLateArrivalLossSize(playbackPoint));
                }

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
    else if(dynamic_cast <BufferMapMessage*> (msg) != NULL)
    {
        BufferMapMessage* BufferMap_Recieved=dynamic_cast<BufferMapMessage*>(msg);
        if(!bufferMapExchangeStart && !isVideoServer)
        {
                scheduleAt(simTime(),requestChunkTimer);
                scheduleAt(simTime(),playingTimer);
                scheduleAt(simTime()+bufferMapExchangePeriod,bufferMapTimer);
                scheduleAt(simTime(), meshPullTimer);
                stat_startBufferMapExchange = simTime().dbl() + bufferMapExchangePeriod;
                stat_startBuffering = simTime().dbl();
                bufferMapExchangeStart = true;
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
        notInNeighbors.clear();
        updateNeighborBMList(BufferMap_Recieved);

        if(schedulingSatisfaction && meshPullActive)
            selectRecieverSideScheduling();
        delete msg;
    }
    else
        delete msg;
}
```