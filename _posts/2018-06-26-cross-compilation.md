---
layout: post
title: Embedded system & Cross Compilation
---
以下我们选取TI公司搭载DaVinci Video Processors的DM814x/AM387x EVM Baseboard为平台来讨论嵌入式系统、交叉编译环境的一些常识。

#  1、背景 #
##   1.1、嵌入式系统 #
这是维基百科词条，说明了嵌入式系统的特性、常见的试用领域：  
>An embedded system is a computer system with a dedicated function within a larger mechanical or electrical system, often with real-time computing constraints. It is embedded as part of a complete device often including hardware and mechanical parts. Embedded systems control many devices in   
common use today. Ninety-eight percent of all microprocessors are manufactured as components of embedded systems.
Examples of properties of typical embedded computers when compared with general-purpose counterparts are low power consumption, small size, rugged operating ranges, and low per-unit cost. This comes at the price of limited processing resources, which make them significantly more difficult to program and to interact with. However, by building intelligence mechanisms on top of the hardware, taking advantage of possible existing sensors and the existence of a network of embedded units, one can both optimally manage available resources at the unit and network levels as well as provide augmented functions, well beyond those available.For example, intelligent techniques can be designed to manage power consumption of embedded systems.

##    1.2、交叉编译 #
这是维基百科词条，说明了交叉编译的定义、存在的意义：  
>A cross compiler is a compiler capable of creating executable code for a platform other than the one on which the compiler is running. For example, a compiler that runs on a Windows 7 PC but generates code that runs on Android smartphone is a cross compiler. A cross compiler is necessary to compile code for multiple platforms from one development host. Direct compilation on the target platform might be infeasible, for example on a microcontroller of an embedded system, because those systems contain no operating system. In paravirtualization, one computer runs multiple operating systems and a cross compiler could generate an executable for each of them from one main source.

#  2、获取资料 #
通常芯片公司推出一款芯片都会给出说明特性、开发套件。  
见  [TMS320DM8148 DaVinci Digital Media Processor](http://www.ti.com/product/TMS320DM8148/description#descriptions)  。
+   说明特性  
通常是给二次开发的项目负责人看的。项目负责人的职能是面对具体的问题或者需求找到合适的解决方案，而说明特性就是告诉他们如果你碰到了以下领域的问题或者以下指标需求，请选择我。见 [TMS320DM8148 DaVinci Digital Media Processor descriptions](http://www.ti.com/product/TMS320DM8148/description#descriptions) 。
+   开发套件  
通常是给二次开发的项目工程师看的。评估板卡展示外围接口扩展、示例软件展示实际业务能力。见  [TMS320DM8148 DaVinci Digital Media Processor tools software](http://www.ti.com/product/TMS320DM8148/toolssoftware)  。  
以下我们将以项目工程师的角色进行实际业务二次开发。

#  3、环境搭建 #

##    3.1 设置环境变量包含交叉编译工具链路径 #
ruisu@ruisu:~/share$ sudo vi /etc/bash.bashrc  
在文件末尾添加:  
export PATH=$PATH:/home/ruisu/share/dvrrdk-rs8148/ti_tools/cgt_a8/arago/linux-devkit/bin  
注：如果提示  
/bin/sh: 1: /home/ruisu/share/dvrrdk-rs8148/dvr_rdk/../ti_tools/cgt_a8/arago/linux-devkit/bin/arm-arago-linux-gnueabi-gcc: not found  
64位系统需要32位库支持：  
ruisu@ruisu:~/share/dvrrdk-rs8148/dvr_rdk$ sudo apt-get install ia32-libs  
再编译应该就没有问题了。  

##    3.2 配置tftp服务 #
安装tftp：
ruisu@ruisu:~/share$ sudo apt-get install tftp-hpa tftpd-hpa xinetd  
这里tftpd为服务端tftp为客户端(可以用于测试服务端是否成功)。  
创建tftp目录：  
ruisu@ruisu:~/share$ mkdir tftproot  
ruisu@ruisu:~/share$ chmod 777 tftproot  
修改tftp配置文件，如果没有就创建：  
ruisu@ruisu:~/share$ sudo vi /etc/xinetd.d/tftp  
添加  
service tftp  
         {  
             disable         = no  
             socket_type     = dgram  
             protocol        = udp  
             wait            = yes  
             user            = ruisu  
             server          = /usr/sbin/in.tftpd  
             server_args     = -s /home/ruisu/share/tftproot  
             source          = 11  
             cps             = 100 2   
             flags =IPv4  
         }  
ruisu@ruisu:~/share$ sudo vi /etc/inetd.conf  
添加  
tftp	dgram	udp	wait	nobody		/usr/sbin/tcpd  
/usr/sbin/in.tftpd	/home/ruisu/share/tftproot  
ruisu@ruisu:~/share$ sudo vi /etc/default/tftpd-hpa  
改为  
#RUN_DAEMON="no"  
#OPTIONS="-s /home/ruisu/share/tftproot -c -p -U tftpd"  
TFTP_USERNAME="tftp"  
TFTP_DIRECTORY="/home/ruisu/share/tftproot"  
TFTP_ADDRESS="0.0.0.0:69"  
TFTP_OPTIONS="-l -c -s"  
重启tftp服务：  
ruisu@ruisu:~/share$ sudo service xinetd restart  
xinetd stop/waiting  
xinetd start/running, process 5159  
测试一下 tftp服务：  
ruisu@ruisu:~/share$ tftp 127.0.0.1 ，能在非tftp目录下下载文件即为成功。  

##    3.3 配置nfs服务 #
安装nfs服务器端：  
ruisu@ruisu:~/share/tftproot/rootfs-avcap$ sudo apt-get install nfs-kernel-server  
设置NFS-Server目录：  
ruisu@ruisu:~/share/tftproot/rootfs-avcap$ sudo vi /etc/exports  
 添加  
/home/ruisu/share/rootfs-avcap    \*(rw,sync,no_subtree_check,no_root_squash)  
重启portmap（如果有必要）和nfs-kernel-server服务：  
ruisu@ruisu:~/share/rootfs-avcap$ sudo service portmap restart  
ruisu@ruisu:~/share/rootfs-avcap$ sudo service nfs-kernel-server restart  
测试本机能否挂载成功：  
ruisu@ruisu:~/share/rootfs-avcap$ sudo mount -t nfs 192.168.1.119:/home/ruisu/share/rootfs-avcap /mnt/share  


#  4、二次开发 #
## 4.1、H264编码视频流写入文件，即录像功能 #  
入口main函数如下。
```c
int main ( int argc, char **argv )
{
    void *dfctx;
    df_ctx * ctx;
    Bool done;
    char ch[MAX_INPUT_STR_SIZE];
    printf("**************dframe_create*************\n");
    dfctx=dframe_create(1920, 1080, VSYS_STD_1080P_60,argc,argv);
    printf("**************dframe_start*************\n");
    dframe_start(dfctx);
    done = FALSE;

    while(!done)
    {
        fgets(ch, MAX_INPUT_STR_SIZE, stdin);
        if(ch[1] != '\n' || ch[0] == '\n')
        continue;
        switch(ch[0])
        {
            case 's':
                //ctx->getStart=1;
            break;
            case 'x':
                done = TRUE;
            break;
            case 'e':
                //ctx->getStart=0;
                break;
            default:
                //printf("This is a simple video capture/display test video source should be 1080P 60hz, please input x then ENTER for exit, otherwise vpss m4 will need reboot!!\r\n");
            break;
        }
    }
    dframe_stop(dfctx);
    dframe_delete(dfctx);
    return (0);
}
```
dframe_create函数，构建syslink可以参考TI官方资料。这里着重说明如何将编码后的视频流写入文件，主要关注回调函数ipcBitsInHostPrm.cbFxn = bitsincallback。
```c
void* dframe_create(int outwidth, int outheight, int videostd,int argc, char **argv)
{
    df_ctx *ctx = (df_ctx*)malloc(sizeof(df_ctx));
    if(ctx == NULL) return NULL;
    char name[20];
    if(argc>=2){
    if((ctx->fp=fopen(argv[argc-1],"w"))==NULL)
        return NULL;
    }else{
        fflush(stdout);
        OSA_printf("\n\nCHAINS:Enter file store name:");
        fflush(stdin);
        fscanf(stdin,"%s",name);
        if((ctx->fp=fopen(name,"w"))==NULL)
        return NULL;
    }

    DeiLink_CreateParams deiPrm;
    CaptureLink_CreateParams	capturePrm;
    CaptureLink_VipInstParams *pCaptureInstPrm;// only a vessel
    CaptureLink_OutParams	  *pCaptureOutPrm;
    SwMsLink_CreateParams		swMsPrm;
    DisplayLink_CreateParams	displayPrm;
    DupLink_CreateParams		dupPrm;
    EncLink_CreateParams	 encPrm;
    DecLink_CreateParams	 decPrm;
    IpcLink_CreateParams	 ipcOutVpssPrm;
    IpcLink_CreateParams	 ipcInVpssPrm;
    IpcLink_CreateParams	 ipcOutVideoPrm;
    IpcLink_CreateParams	 ipcInVideoPrm;
    IpcBitsOutLinkHLOS_CreateParams   ipcBitsOutHostPrm;
    IpcBitsOutLinkRTOS_CreateParams   ipcBitsOutVideoPrm;
    IpcBitsInLinkHLOS_CreateParams	  ipcBitsInHostPrm;
    IpcBitsInLinkRTOS_CreateParams	  ipcBitsInVideoPrm;
    Int i;
    Bool isProgressive;
    System_LinkInfo bitsProducerLinkInfo;

    UInt32 captureId, deiId, displayId;
    UInt32 encId, decId, snkId, dupId;
    UInt32 ipcOutVpssId, ipcInVpssId;
    UInt32 ipcOutVideoId, ipcInVideoId;
    UInt32 ipcBitsOutVideoId, ipcBitsInHostId;
    UInt32 ipcBitsInVideoId, ipcBitsOutHostId;
    char ch;
    UInt32 vipInstId;

    CaptureLink_CreateParams_Init(&capturePrm);
    DisplayLink_CreateParams_Init(&displayPrm);
    DeiLink_CreateParams_Init(&deiPrm);
    CHAINS_INIT_STRUCT(IpcLink_CreateParams,ipcOutVpssPrm);
    CHAINS_INIT_STRUCT(IpcLink_CreateParams,ipcInVpssPrm);
    CHAINS_INIT_STRUCT(IpcLink_CreateParams,ipcOutVideoPrm);
    CHAINS_INIT_STRUCT(IpcLink_CreateParams,ipcInVideoPrm);
    CHAINS_INIT_STRUCT(IpcBitsOutLinkHLOS_CreateParams,ipcBitsOutHostPrm);
    CHAINS_INIT_STRUCT(IpcBitsOutLinkRTOS_CreateParams,ipcBitsOutVideoPrm);
    CHAINS_INIT_STRUCT(IpcBitsInLinkHLOS_CreateParams,ipcBitsInHostPrm);
    CHAINS_INIT_STRUCT(IpcBitsInLinkRTOS_CreateParams,ipcBitsInVideoPrm);
    CHAINS_INIT_STRUCT(DecLink_CreateParams, decPrm);
    CHAINS_INIT_STRUCT(EncLink_CreateParams, encPrm);

    captureId	= SYSTEM_LINK_ID_CAPTURE;
    deiId = SYSTEM_LINK_ID_DEI_0;
    displayId	= SYSTEM_LINK_ID_DISPLAY_0;
    encId  = SYSTEM_LINK_ID_VENC_0;
    dupId = SYSTEM_VPSS_LINK_ID_DUP_0;
    ipcOutVpssId = SYSTEM_VPSS_LINK_ID_IPC_OUT_M3_0;
    ipcInVideoId = SYSTEM_VIDEO_LINK_ID_IPC_IN_M3_0;
    ipcOutVideoId= SYSTEM_VIDEO_LINK_ID_IPC_OUT_M3_0;
    ipcInVpssId  = SYSTEM_VPSS_LINK_ID_IPC_IN_M3_0;
    ipcBitsOutVideoId = SYSTEM_VIDEO_LINK_ID_IPC_BITS_OUT_0;
    ipcBitsInHostId   = SYSTEM_HOST_LINK_ID_IPC_BITS_IN_0;
    ipcBitsOutHostId  = SYSTEM_HOST_LINK_ID_IPC_BITS_OUT_0;
    ipcBitsInVideoId  = SYSTEM_VIDEO_LINK_ID_IPC_BITS_IN_0;

#ifdef A8_CONTROL_I2C
    Device_Sii9135Handle sii9135Handle;
#endif
#ifdef A8_CONTROL_I2C
    {
        Int32 status = 0;
        Device_VideoDecoderChipIdParams 	 vidDecChipIdArgs;
        Device_VideoDecoderChipIdStatus 	 vidDecChipIdStatus;
        VCAP_VIDEO_SOURCE_STATUS_PARAMS_S	 videoStatusArgs;
        VCAP_VIDEO_SOURCE_CH_STATUS_S		 videoStatus;

        Device_VideoDecoderCreateParams 	 createArgs;
        Device_VideoDecoderCreateStatus 	 createStatusArgs;

        Device_VideoDecoderVideoModeParams				   vidDecVideoModeArgs;

        printf(" set sii9135\r\n");
        /* Initialize and create video decoders */
        Device_sii9135Init();

        memset(&createArgs, 0, sizeof(Device_VideoDecoderCreateParams));

        createArgs.deviceI2cInstId	  = 0;
        createArgs.numDevicesAtPort   = 1;
        createArgs.deviceI2cAddr[0]   = Device_getVidDecI2cAddr(DEVICE_VID_DEC_SII9135_DRV,0);
        createArgs.deviceResetGpio[0] = DEVICE_VIDEO_DECODER_GPIO_NONE;

        sii9135Handle = Device_sii9135Create(		   DEVICE_VID_DEC_SII9135_DRV,
                               0, // instId - need to change
                               &(createArgs),
                               &(createStatusArgs));

        vidDecChipIdArgs.deviceNum = 0;

        status = Device_sii9135Control(sii9135Handle,
                         IOCTL_DEVICE_VIDEO_DECODER_GET_CHIP_ID,
                         &vidDecChipIdArgs,
                         &vidDecChipIdStatus);
        if (status >= 0)
        {
            videoStatusArgs.channelNum = 0;

            status = Device_sii9135Control(sii9135Handle,
                           IOCTL_DEVICE_VIDEO_DECODER_GET_VIDEO_STATUS,
                           &videoStatusArgs, &videoStatus);

            if (videoStatus.isVideoDetect)
            {
                if(videoStatus.frameInterval==0) videoStatus.frameInterval=1;

                printf(" VCAP: SII9135 (0x%02x): Detected video (%dx%d@%dHz, %d) !!!\n",
                    createArgs.deviceI2cAddr[0],
                    videoStatus.frameWidth,
                    videoStatus.frameHeight,
                    1000000 / videoStatus.frameInterval,
                    videoStatus.isInterlaced);

                if(videoStatus.isInterlaced)
                    videostd = DF_STD_1080I_60;
                else
                    videostd = DF_STD_1080P_60;
            }
            else
            {
                printf(" VCAP: SII9135 (0x%02x):  NO Video Detected !, try again\n", createArgs.deviceI2cAddr[0]);
#if 1
                usleep(100000);

                status = Device_sii9135Control(sii9135Handle,
                            IOCTL_DEVICE_VIDEO_DECODER_GET_VIDEO_STATUS,
                            &videoStatusArgs, &videoStatus);

                if (videoStatus.isVideoDetect)
                {
                    if(videoStatus.frameInterval==0) videoStatus.frameInterval=1;

                    printf(" VCAP: SII9135 (0x%02x): Detected video (%dx%d@%dHz, %d) !!!\n",
                        createArgs.deviceI2cAddr[0],
                        videoStatus.frameWidth,
                        videoStatus.frameHeight,
                        1000000 / videoStatus.frameInterval,
                        videoStatus.isInterlaced);

                    if(videoStatus.isInterlaced)
                        videostd = DF_STD_1080I_60;
                    else
                        videostd = DF_STD_1080P_60;
                }
                else
                {
                    printf(" VCAP: SII9135 (0x%02x):  NO Video Detected !!!\n", createArgs.deviceI2cAddr[0]);
                    Device_sii9135Delete(sii9135Handle, NULL);
                    Device_sii9135DeInit();
                    System_deInit();
                    return NULL;
                }
#endif
            }
        }
        else
        {
            printf(" VCAP: SII9135 (0x%02x): Device not found !!!\n", createArgs.deviceI2cAddr[0]);
        }

        /* Configure video decoder */
        memset(&vidDecVideoModeArgs,0, sizeof(Device_VideoDecoderVideoModeParams));
        vidDecVideoModeArgs.videoIfMode 	   = DEVICE_CAPT_VIDEO_IF_MODE_16BIT;
        vidDecVideoModeArgs.videoDataFormat    = SYSTEM_DF_YUV422P;
        vidDecVideoModeArgs.standard		   = videostd;
        vidDecVideoModeArgs.videoCaptureMode   = DEVICE_CAPT_VIDEO_CAPTURE_MODE_SINGLE_CH_NON_MUX_EMBEDDED_SYNC;
        vidDecVideoModeArgs.videoSystem 	   = DEVICE_VIDEO_DECODER_VIDEO_SYSTEM_NONE;
        vidDecVideoModeArgs.videoCropEnable    = FALSE;
        vidDecVideoModeArgs.videoAutoDetectTimeout = -1;

        status = Device_sii9135Control(sii9135Handle,IOCTL_DEVICE_VIDEO_DECODER_SET_VIDEO_MODE,&vidDecVideoModeArgs,NULL);
    }
    ctx->sii9135Handle=sii9135Handle;
#endif

    System_init();
    capturePrm.numVipInst = 1;
    capturePrm.outQueParams[0].nextLink = dupId;
    capturePrm.tilerEnable				= FALSE;
    capturePrm.enableSdCrop 			= FALSE;
    pCaptureInstPrm 					= &capturePrm.vipInst[0];
    pCaptureInstPrm->vipInstId			= SYSTEM_CAPTURE_INST_VIP1_PORTA;
#ifndef A8_CONTROL_I2C
    pCaptureInstPrm->videoDecoderId 	= SYSTEM_DEVICE_VID_DEC_SII9135_DRV;
#endif
    pCaptureInstPrm->inDataFormat		= SYSTEM_DF_YUV422P;
    pCaptureInstPrm->standard			= videostd;
    pCaptureInstPrm->numOutput			= 1;
    pCaptureOutPrm						= &pCaptureInstPrm->outParams[0];
    pCaptureOutPrm->dataFormat			= SYSTEM_DF_YUV420SP_UV;
    pCaptureOutPrm->scEnable			= FALSE;
    pCaptureOutPrm->scOutWidth			= outwidth;
    pCaptureOutPrm->scOutHeight 		= outheight;
    pCaptureOutPrm->outQueId			= 0;

    dupPrm.inQueParams.prevLinkId = captureId;
    dupPrm.inQueParams.prevLinkQueId= 0;
    dupPrm.outQueParams[0].nextLink = deiId;
    dupPrm.outQueParams[1].nextLink = ipcOutVpssId;
    dupPrm.numOutQue				   = 2;
    dupPrm.notifyNextLink			   = TRUE;

    ipcOutVpssPrm.inQueParams.prevLinkId    = dupId;
    ipcOutVpssPrm.inQueParams.prevLinkQueId = 1;
    ipcOutVpssPrm.numOutQue                 = 1;
    ipcOutVpssPrm.outQueParams[0].nextLink  = ipcInVideoId;
    ipcOutVpssPrm.notifyNextLink            = TRUE;
    ipcOutVpssPrm.notifyPrevLink            = TRUE;
    ipcOutVpssPrm.noNotifyMode              = FALSE;

    ipcInVideoPrm.inQueParams.prevLinkId    = ipcOutVpssId;
    ipcInVideoPrm.inQueParams.prevLinkQueId = 0;
    ipcInVideoPrm.numOutQue                 = 1;
    ipcInVideoPrm.outQueParams[0].nextLink  = encId;
    ipcInVideoPrm.notifyNextLink            = TRUE;
    ipcInVideoPrm.notifyPrevLink            = TRUE;
    ipcInVideoPrm.noNotifyMode              = FALSE;

    for (i=0; i<1; i++)
    {
        encPrm.chCreateParams[i].format 	= IVIDEO_H264HP;
        encPrm.chCreateParams[i].profile	= IH264_HIGH_PROFILE;
        encPrm.chCreateParams[i].dataLayout = IVIDEO_FIELD_SEPARATED;
        if (TRUE)
            encPrm.chCreateParams[i].fieldMergeEncodeEnable  = FALSE;
        else
            encPrm.chCreateParams[i].fieldMergeEncodeEnable  = TRUE;
        encPrm.chCreateParams[i].maxBitRate = -1;
        encPrm.chCreateParams[i].encodingPreset = 3;
        encPrm.chCreateParams[i].rateControlPreset = 0;
        encPrm.chCreateParams[i].enableHighSpeed = 0;
        encPrm.chCreateParams[i].defaultDynamicParams.intraFrameInterval = 30;
        encPrm.chCreateParams[i].encodingPreset = XDM_DEFAULT;
        encPrm.chCreateParams[i].enableAnalyticinfo = 0;
        encPrm.chCreateParams[i].enableWaterMarking = 0;
        encPrm.chCreateParams[i].rateControlPreset =
        IVIDEO_STORAGE;
        encPrm.chCreateParams[i].defaultDynamicParams.inputFrameRate = 30;
        encPrm.chCreateParams[i].defaultDynamicParams.targetBitRate = (2 * 1000 * 1000);
        encPrm.chCreateParams[i].defaultDynamicParams.interFrameInterval = 1;
        encPrm.chCreateParams[i].defaultDynamicParams.mvAccuracy =
        IVIDENC2_MOTIONVECTOR_QUARTERPEL;
        encPrm.chCreateParams[i].defaultDynamicParams.rcAlg = 0 ;
        encPrm.chCreateParams[i].defaultDynamicParams.qpMin = 10;
        encPrm.chCreateParams[i].defaultDynamicParams.qpMax = 40;
        encPrm.chCreateParams[i].defaultDynamicParams.qpInit = -1;
        encPrm.chCreateParams[i].defaultDynamicParams.vbrDuration = 8;
        encPrm.chCreateParams[i].defaultDynamicParams.vbrSensitivity = 0;
    }

    encPrm.inQueParams.prevLinkId    = ipcInVideoId;
    encPrm.inQueParams.prevLinkQueId = 0;
    encPrm.outQueParams.nextLink     = ipcBitsOutVideoId;

    ipcBitsOutVideoPrm.baseCreateParams.inQueParams.prevLinkId    = encId;
    ipcBitsOutVideoPrm.baseCreateParams.inQueParams.prevLinkQueId = 0;
    ipcBitsOutVideoPrm.baseCreateParams.numOutQue                 = 1;
    ipcBitsOutVideoPrm.baseCreateParams.outQueParams[0].nextLink   = ipcBitsInHostId;
    ipcBitsOutVideoPrm.baseCreateParams.notifyPrevLink = TRUE;
    ipcBitsOutVideoPrm.baseCreateParams.notifyNextLink = FALSE;
    ipcBitsOutVideoPrm.baseCreateParams.noNotifyMode = TRUE;

    ipcBitsInHostPrm.baseCreateParams.inQueParams.prevLinkId = ipcBitsOutVideoId;
    ipcBitsInHostPrm.baseCreateParams.inQueParams.prevLinkQueId = 0;
    ipcBitsInHostPrm.cbCtx = ctx;
    ipcBitsInHostPrm.cbFxn = bitsincallback;
    ipcBitsInHostPrm.baseCreateParams.notifyNextLink = FALSE;
    ipcBitsInHostPrm.baseCreateParams.notifyPrevLink = FALSE;

    deiPrm.inQueParams.prevLinkId                        = dupId;
    deiPrm.inQueParams.prevLinkQueId                     = 0;
    deiPrm.outQueParams[DEI_LINK_OUT_QUE_DEI_SC].nextLink               = displayId;
    deiPrm.enableOut[DEI_LINK_OUT_QUE_DEI_SC]               = TRUE;
    deiPrm.enableOut[DEI_LINK_OUT_QUE_VIP_SC_SECONDARY_OUT] = FALSE;
    deiPrm.enableOut[DEI_LINK_OUT_QUE_VIP_SC]               = FALSE;
    deiPrm.tilerEnable                                   = FALSE;
    deiPrm.comprEnable                                   = FALSE;
    deiPrm.setVipScYuv422Format                          = FALSE;
    if(videostd==DF_STD_1080I_60)
        deiPrm.enableDeiForceBypass  = FALSE;
    else
        deiPrm.enableDeiForceBypass                      = TRUE;
    deiPrm.enableLineSkipSc                              = FALSE;
    deiPrm.outScaleFactor[DEI_LINK_OUT_QUE_DEI_SC][0].scaleMode = DEI_SCALE_MODE_RATIO;
    deiPrm.outScaleFactor[DEI_LINK_OUT_QUE_DEI_SC][0].ratio.heightRatio.numerator   = 1;
    deiPrm.outScaleFactor[DEI_LINK_OUT_QUE_DEI_SC][0].ratio.heightRatio.denominator = 1;
    deiPrm.outScaleFactor[DEI_LINK_OUT_QUE_DEI_SC][0].ratio.widthRatio.numerator = 1;
    deiPrm.outScaleFactor[DEI_LINK_OUT_QUE_DEI_SC][0].ratio.widthRatio.denominator = 1;
    deiPrm.outScaleFactor[DEI_LINK_OUT_QUE_DEI_SC][0] = deiPrm.outScaleFactor[DEI_LINK_OUT_QUE_DEI_SC][0];
    /* FPS rates of DEI queue connected to Display */
    deiPrm.inputFrameRate[DEI_LINK_OUT_QUE_DEI_SC]  = 60;
    deiPrm.outputFrameRate[DEI_LINK_OUT_QUE_DEI_SC] = 30;

    displayPrm.inQueParams[0].prevLinkId	= deiId;
    displayPrm.inQueParams[0].prevLinkQueId = 0;
    displayPrm.displayRes				 = VSYS_STD_1080P_60;

    UInt32 displayRes[SYSTEM_DC_MAX_VENC] =
    {
        VSYS_STD_1080P_60,   //SYSTEM_DC_VENC_HDMI,
        VSYS_STD_1080P_60,    //SYSTEM_DC_VENC_HDCOMP,
        VSYS_STD_1080P_60,    //SYSTEM_DC_VENC_DVO2
        VSYS_STD_NTSC        //SYSTEM_DC_VENC_SD,
    };

    Chains_displayCtrlInit(displayRes);

    printf("*************captureId**************\n");
    System_linkCreate (captureId, &capturePrm, sizeof(capturePrm));
    printf("*************dupId**************\n");
    System_linkCreate(dupId, &dupPrm, sizeof(dupPrm));
    printf("*************deiId**************\n");
    System_linkCreate(deiId	 , &deiPrm, sizeof(deiPrm));
    printf("*************ipcOutVpssId**************\n");
    System_linkCreate(ipcOutVpssId , &ipcOutVpssPrm , sizeof(ipcOutVpssPrm) );
    printf("*************ipcInVideoId**************\n");
    System_linkCreate(ipcInVideoId , &ipcInVideoPrm , sizeof(ipcInVideoPrm) );
    printf("*************encId**************\n");
    System_linkCreate(encId, &encPrm, sizeof(encPrm));
    printf("**************ipcBitsOutVideoId*************\n");
    System_linkCreate(ipcBitsOutVideoId, &ipcBitsOutVideoPrm, sizeof(ipcBitsOutVideoPrm));
    printf("************ipcBitsInHostId***************\n");
    System_linkCreate(ipcBitsInHostId, &ipcBitsInHostPrm, sizeof(ipcBitsInHostPrm));
    printf("**************displayId*************\n");
    System_linkCreate(displayId, &displayPrm, sizeof(displayPrm));

    ctx->captureId	= captureId;
    ctx->displayId	= displayId;
    ctx->dupId = dupId;
    ctx->deiId = deiId;
    ctx->ipcOutVpssId = ipcOutVpssId;
    ctx->ipcInVideoId = ipcInVideoId;
    ctx->encId = encId;
    ctx->ipcBitsOutVideoId = ipcBitsOutVideoId;
    ctx->ipcBitsInHostId = ipcBitsInHostId;

    dframe_ipcBitsInitThrObj(ctx);
    return ctx;
}
```
bitsincallback发送信号量。  
```c
void bitsincallback(void *ctx)
{
    df_ctx *h = (df_ctx*)ctx;
    OSA_printf("*********bits in callback called************\n");
    OSA_semSignal(&h->bitsInSem);
}
```
信号量触发dframe_ipcBitsRecvFxn函数进行视频流文件写入：  
```c
static Void *dframe_ipcBitsRecvFxn(Void * prm)
{
    df_ctx *ctx = ( df_ctx *) prm;
    Int i,status;
    Bitstream_BufList fullBufList;
    Bitstream_Buf *pFullBuf;
    Bitstream_Buf *pEmptyBuf;
    Bitstream_BufList* pfullBufList=&fullBufList;

    while (FALSE == ctx->exitBitsInThread)
    {
        OSA_semWait(&ctx->bitsInSem,OSA_TIMEOUT_FOREVER);
        //if(ctx->getStart)
        if(1)
        {
            pfullBufList->numBufs = 0;
            IpcBitsInLink_getFullVideoBitStreamBufs(ctx->ipcBitsInHostId,pfullBufList);
            if (pfullBufList->numBufs)
            {
                for (i = 0; i < pfullBufList->numBufs; i++)
                {			
                    pFullBuf = (fullBufList.bufs[i]);
                    if(pFullBuf->fillLength)
                    {
                        fwrite(pFullBuf->addr,sizeof(char),pFullBuf->fillLength,ctx->fp);
                    }
                }
                OSA_waitMsecs(DFRAME_SENDRECVFXN_PERIOD_MS);
                IpcBitsInLink_putEmptyVideoBitStreamBufs(ctx->ipcBitsInHostId,pfullBufList);
            }

        }
        else
        {
            pfullBufList->numBufs = 0;
            IpcBitsInLink_getFullVideoBitStreamBufs(ctx->ipcBitsInHostId,pfullBufList);
            if (pfullBufList->numBufs)
            {
                status =IpcBitsInLink_putEmptyVideoBitStreamBufs(ctx->ipcBitsInHostId,pfullBufList);
            }
        }
        OSA_waitMsecs(DFRAME_SENDRECVFXN_PERIOD_MS);
        OSA_printf("DFRAME:%s:Leaving...",__func__);
    }
    return NULL;
}
```

## 4.2、H264文件解码输出，即播放功能 #
入口main函数如下。
```c
int main ( int argc, char **argv )
{
    void *dfctx;
    df_ctx * ctx;
    Bool done;
    char ch[MAX_INPUT_STR_SIZE];
    printf("**************dframe_create*************\n");
    dfctx=dframe_create(1920, 1080, VSYS_STD_1080P_60,argc,argv);
    printf("**************dframe_start*************\n");
    dframe_start(dfctx);
    done = FALSE;
    while(!done)
    {
        fgets(ch, MAX_INPUT_STR_SIZE, stdin);
        if(ch[1] != '\n' || ch[0] == '\n')
        continue;
        switch(ch[0])
        {
           case 'x':
               done = TRUE;
           break;
           default:
           break;
        }
    }
    dframe_stop(dfctx);
    dframe_delete(dfctx);
    return (0);
}
```
dframe_create 函数，构建syslink可以参考TI官方资料。这里着重说明如何将文件中的视频流导入，我们主要关注 dframe_ipcBitsInitThrObj(ctx)。
```c
void* dframe_create(int outwidth, int outheight, int videostd,int argc, char **argv)
{
    df_ctx *ctx = (df_ctx*)malloc(sizeof(df_ctx));

    if(ctx == NULL) return NULL;
    if(argc>=2){
        if((ctx->fp=fopen(argv[argc-1],"r"))==NULL)
            return NULL;
    }
    else{
        char name[20];
        fflush(stdout);
        OSA_printf("\n\nCHAINS:Enter file store name:");
        fflush(stdin);
        fscanf(stdin, "%s",name);
        if((ctx->fp=fopen(name,"r"))==NULL)
            return NULL;
    }
    SwMsLink_CreateParams		swMsPrm;
    DisplayLink_CreateParams	displayPrm;
    DupLink_CreateParams		dupPrm;
    EncLink_CreateParams	 encPrm;
    DecLink_CreateParams	 decPrm;
    IpcLink_CreateParams	 ipcOutVpssPrm;
    IpcLink_CreateParams	 ipcInVpssPrm;
    IpcLink_CreateParams	 ipcOutVideoPrm;
    IpcLink_CreateParams	 ipcInVideoPrm;
    IpcBitsOutLinkHLOS_CreateParams   ipcBitsOutHostPrm;
    IpcBitsOutLinkRTOS_CreateParams   ipcBitsOutVideoPrm;
    IpcBitsInLinkHLOS_CreateParams	  ipcBitsInHostPrm;
    IpcBitsInLinkRTOS_CreateParams	  ipcBitsInVideoPrm;
    MergeLink_CreateParams			   mergePrm;

    Int i;
    Bool isProgressive;
    System_LinkQueInfo bitsProducerLinkInfo;
    UInt32 captureId, swMsId, displayId;
    UInt32 encId, decId, mergeId, snkId, dupId;
    UInt32 ipcOutVpssId, ipcInVpssId;
    UInt32 ipcOutVideoId, ipcInVideoId;
    UInt32 ipcBitsOutVideoId, ipcBitsInHostId;
    UInt32 ipcBitsInVideoId, ipcBitsOutHostId;
    char ch;
    UInt32 vipInstId;

    SwMsLink_CreateParams_Init(&swMsPrm);
    DisplayLink_CreateParams_Init(&displayPrm);
    CHAINS_INIT_STRUCT(IpcLink_CreateParams,ipcInVpssPrm);
    CHAINS_INIT_STRUCT(IpcLink_CreateParams,ipcOutVideoPrm);
    CHAINS_INIT_STRUCT(IpcBitsOutLinkHLOS_CreateParams,ipcBitsOutHostPrm);
    CHAINS_INIT_STRUCT(IpcBitsInLinkRTOS_CreateParams,ipcBitsInVideoPrm);
    CHAINS_INIT_STRUCT(DecLink_CreateParams, decPrm);

    swMsId		= SYSTEM_LINK_ID_SW_MS_MULTI_INST_0;
    displayId	= SYSTEM_LINK_ID_DISPLAY_0;
    decId  = SYSTEM_LINK_ID_VDEC_0;
    ipcOutVideoId= SYSTEM_VIDEO_LINK_ID_IPC_OUT_M3_0;
    ipcInVpssId  = SYSTEM_VPSS_LINK_ID_IPC_IN_M3_0;
    ipcBitsOutHostId  = SYSTEM_HOST_LINK_ID_IPC_BITS_OUT_0;
    ipcBitsInVideoId  = SYSTEM_VIDEO_LINK_ID_IPC_BITS_IN_0;

#ifdef A8_CONTROL_I2C
    Device_Sii9135Handle sii9135Handle;
#endif
    System_init();
    ipcBitsOutHostPrm.inQueInfo.numCh = 1;
    ipcBitsOutHostPrm.inQueInfo.chInfo[0].bufType = SYSTEM_BUF_TYPE_VIDBITSTREAM;
    ipcBitsOutHostPrm.inQueInfo.chInfo[0].height = 1080;
    ipcBitsOutHostPrm.inQueInfo.chInfo[0].width = 1920;
    ipcBitsOutHostPrm.inQueInfo.chInfo[0].codingformat = IVIDEO_H264HP;
    ipcBitsOutHostPrm.inQueInfo.chInfo[0].memType = SYSTEM_MT_TILEDMEM;
    ipcBitsOutHostPrm.inQueInfo.chInfo[0].dataFormat = IVIDEO_H264HP;
    ipcBitsOutHostPrm.inQueInfo.chInfo[0].scanFormat = SYSTEM_SF_PROGRESSIVE;
    ipcBitsOutHostPrm.inQueInfo.chInfo[0].startX         = 0;
    ipcBitsOutHostPrm.inQueInfo.chInfo[0].startY         = 0;
    ipcBitsOutHostPrm.inQueInfo.chInfo[0].pitch[0]       = 0;
    ipcBitsOutHostPrm.inQueInfo.chInfo[0].pitch[1]       = 0;
    ipcBitsOutHostPrm.inQueInfo.chInfo[0].pitch[2]       = 0;
    ipcBitsOutHostPrm.baseCreateParams.numOutQue = 1;
    ipcBitsOutHostPrm.baseCreateParams.numChPerOutQue[0] = 1;
    ipcBitsOutHostPrm.baseCreateParams.outQueParams[0].nextLink = ipcBitsInVideoId;
    ipcBitsOutHostPrm.baseCreateParams.notifyNextLink = FALSE;
    ipcBitsOutHostPrm.baseCreateParams.notifyPrevLink = FALSE;
    ipcBitsOutHostPrm.baseCreateParams.noNotifyMode = TRUE;

    ipcBitsInVideoPrm.baseCreateParams.inQueParams.prevLinkId = ipcBitsOutHostId;
    ipcBitsInVideoPrm.baseCreateParams.inQueParams.prevLinkQueId = 0;
    ipcBitsInVideoPrm.baseCreateParams.numOutQue				 = 1;
    ipcBitsInVideoPrm.baseCreateParams.outQueParams[0].nextLink = decId;
    Chains_ipcBitsInitCreateParams_BitsInRTOS(&ipcBitsInVideoPrm, TRUE);

    for (i=0; i<1; i++) {
        decPrm.chCreateParams[i].format 		 = IVIDEO_H264HP;
        decPrm.chCreateParams[i].profile		 = IH264VDEC_PROFILE_ANY;
        decPrm.chCreateParams[i].targetMaxWidth  = 1920;
        decPrm.chCreateParams[i].targetMaxHeight = 1080;
        if (isProgressive)
            decPrm.chCreateParams[i].fieldMergeDecodeEnable  = FALSE;
        else
            decPrm.chCreateParams[i].fieldMergeDecodeEnable  = TRUE;
        decPrm.chCreateParams[i].numBufPerCh = 6;
        decPrm.chCreateParams[i].defaultDynamicParams.targetFrameRate = 30;
        decPrm.chCreateParams[i].defaultDynamicParams.targetBitRate = (2 * 1000 * 1000);
    }
    decPrm.inQueParams.prevLinkId = ipcBitsInVideoId;
    decPrm.inQueParams.prevLinkQueId = 0;
    decPrm.outQueParams.nextLink	 = ipcOutVideoId;

    ipcOutVideoPrm.inQueParams.prevLinkId	 = decId;
    ipcOutVideoPrm.inQueParams.prevLinkQueId = 0;
    ipcOutVideoPrm.numOutQue				   = 1;
    ipcOutVideoPrm.outQueParams[0].nextLink 	= ipcInVpssId;
    ipcOutVideoPrm.notifyNextLink			 = TRUE;
    ipcOutVideoPrm.notifyPrevLink			 = TRUE;

    ipcInVpssPrm.inQueParams.prevLinkId    = ipcOutVideoId;
    ipcInVpssPrm.inQueParams.prevLinkQueId = 0;
    ipcInVpssPrm.numOutQue					  = 1;
    ipcInVpssPrm.outQueParams[0].nextLink	= swMsId;
    ipcInVpssPrm.notifyNextLink 		   = TRUE;
    ipcInVpssPrm.notifyPrevLink 		   = TRUE;

    swMsPrm.inQueParams.prevLinkId = ipcInVpssId;
    swMsPrm.inQueParams.prevLinkQueId = 0;
    swMsPrm.outQueParams.nextLink	  = displayId;
    swMsPrm.numSwMsInst = 1;
    swMsPrm.swMsInstId[0] = SYSTEM_SW_MS_SC_INST_SC5;
    swMsPrm.maxInputQueLen			  = 4;
    swMsPrm.maxOutRes				  = VSYS_STD_1080P_60;
    swMsPrm.numOutBuf				  = 8;
    swMsPrm.lineSkipMode			  = TRUE;
    swMsPrm.layoutPrm.outputFPS 	  = 60;
    Chains_swMsGenerateLayoutParams(0, 2, &swMsPrm);

    displayPrm.inQueParams[0].prevLinkId	= swMsId;
    displayPrm.inQueParams[0].prevLinkQueId = 0;
    displayPrm.displayRes	= swMsPrm.maxOutRes;

    UInt32 displayRes[SYSTEM_DC_MAX_VENC] =
    {
        VSYS_STD_1080P_60,   //SYSTEM_DC_VENC_HDMI,
        VSYS_STD_1080P_60,    //SYSTEM_DC_VENC_HDCOMP,
        VSYS_STD_1080P_60,    //SYSTEM_DC_VENC_DVO2
        VSYS_STD_NTSC        //SYSTEM_DC_VENC_SD,
    };
    Chains_displayCtrlInit(displayRes);

    printf("**************ipcBitsOutHostId*************\n");
    System_linkCreate(ipcBitsOutHostId,&ipcBitsOutHostPrm,sizeof(ipcBitsOutHostPrm));
    printf("**************ipcBitsInVideoId*************\n");
    System_linkCreate(ipcBitsInVideoId,&ipcBitsInVideoPrm,sizeof(ipcBitsInVideoPrm));
    printf("**************decId*************\n");
    System_linkCreate(decId, &decPrm, sizeof(decPrm));
    printf("**************ipcOutVideoId*************\n");
    System_linkCreate(ipcOutVideoId, &ipcOutVideoPrm, sizeof(ipcOutVideoPrm));
    printf("**************ipcInVpssId*************\n");
    System_linkCreate(ipcInVpssId  , &ipcInVpssPrm	, sizeof(ipcInVpssPrm)	);
    printf("*************swMsId**************\n");
    System_linkCreate(swMsId  , &swMsPrm, sizeof(swMsPrm));
    printf("**************displayId*************\n");
    System_linkCreate(displayId, &displayPrm, sizeof(displayPrm));

    ctx->swMsId		= swMsId;
    ctx->displayId	= displayId;
    ctx->ipcBitsOutHostId	= ipcBitsOutHostId;
    ctx->ipcBitsInVideoId	= ipcBitsInVideoId;
    ctx->decId	= decId;
    ctx->ipcOutVideoId	= ipcOutVideoId;
    ctx->ipcInVpssId	= ipcInVpssId;
    dframe_ipcBitsInitThrObj(ctx);
    return ctx;
}
```
dframe_ipcBitsInitThrObj函数创建信号量、buff队列、并建立新的线程，我们主要关注dframe_ipcBitsSendFxn。  
```c
static Void dframe_ipcBitsInitThrObj(df_ctx *thrObj)
{
    OSA_semCreate(&thrObj->bitsInSem,CHAINS_IPCBITS_MAX_PENDING_RECV_SEM_COUNT,0);
    thrObj->exitBitsInThread = FALSE;
    thrObj->exitBitsOutThread = FALSE;
    OSA_queCreate(&thrObj->bufQFreeBufs,CHAINS_IPCBITS_FREE_QUE_MAX_LEN);
    OSA_queCreate(&thrObj->bufQFullBufs,CHAINS_IPCBITS_FULL_QUE_MAX_LEN);
    OSA_thrCreate(&thrObj->thrHandleBitsIn,
                            dframe_ipcBitsSendFxn,
                            CHAINS_IPCBITS_RECVFXN_TSK_PRI,
                            CHAINS_IPCBITS_RECVFXN_TSK_STACK_SIZE,
                            thrObj);
}
```
dframe_ipcBitsSendFxn这个新的线程循环体中IpcBitsOutLink_getEmptyVideoBitStreamBufs 与 IpcBitsOutLink_putFullVideoBitStreamBufs 构成一对，
将 dframe_read_frame_h264 从文件中读入的视频流输入到 ipcBitsOutHostId 对应的syslink中，现在视频流的来龙去脉一目了然。
```c
static Void *dframe_ipcBitsSendFxn(Void * prm)
{
    df_ctx *thrObj = ( df_ctx *) prm;
    Int i,status;
    Bitstream_BufList emptyBufList;
    Bitstream_Buf *pBuf;
    UInt32 bitBufSize,framesize;
    IpcBitsOutLinkHLOS_BitstreamBufReqInfo reqInfo;

    char *framebuf=(char *)malloc(FRAME_BUFFER_LEN*sizeof(char));
    if(framebuf==NULL)return NULL;
    memset(framebuf,0,FRAME_BUFFER_LEN*sizeof(char));
    while (FALSE == thrObj->exitBitsOutThread)
    {
        OSA_waitMsecs(DFRAME_SENDRECVFXN_PERIOD_MS);
        bitBufSize = CHAINS_IPCBITS_GET_BITBUF_SIZE(CHAINS_IPCBITS_DEFAULT_WIDTH, CHAINS_IPCBITS_DEFAULT_HEIGHT);
        emptyBufList.numBufs = 0;
        reqInfo.numBufs = VIDBITSTREAM_MAX_BITSTREAM_BUFS;
        reqInfo.reqType = IPC_BITSOUTHLOS_BITBUFREQTYPE_BUFSIZE;
        for (i = 0; i < VIDBITSTREAM_MAX_BITSTREAM_BUFS; i++)
        {
            reqInfo.u[i].minBufSize = bitBufSize;
        }
        IpcBitsOutLink_getEmptyVideoBitStreamBufs(thrObj->ipcBitsOutHostId,&emptyBufList,&reqInfo);
        if (emptyBufList.numBufs)
        {
            OSA_printf("DFRAME:%s:***********getbufs******************",__func__);
            for (i = 0; i < emptyBufList.numBufs;)
            {
                pBuf = emptyBufList.bufs[i];
                pBuf->channelNum = 0;
                pBuf->codingType = 0;
                pBuf->fillLength = 0;
                pBuf->isKeyFrame = 0;
                pBuf->timeStamp  = 0;
                pBuf->mvDataFilledSize = 0;
                pBuf->bottomFieldBitBufSize = 0;
                pBuf->inputFileChanged = 0;
                framesize=0;
                framesize =dframe_read_frame_h264(thrObj->fp,framebuf);
                if(framesize==-1)break;
                if(framesize!=-1 && framesize){
                    memcpy(pBuf->addr, framebuf, framesize);
                    pBuf->fillLength=framesize;
                    printf("framesize:***********%d******************",framesize);
                    i++;
                }
            }
            IpcBitsOutLink_putFullVideoBitStreamBufs(thrObj->ipcBitsOutHostId,&emptyBufList);
        }
    }
    free(framebuf);
    OSA_printf("DFRAME:%s:Leaving...",__func__);
    return NULL;
}
```
视频文件存储是对应一帧帧的图像的，唯一不同的是采用了各式各样的压缩技术。 从 dframe_read_frame_h264 函数你可以领略到其中的意味。下面我们来分析关键的 dframe_read_frame_h264 函数。
```c
#define DFRAME_FRAMEPOOL_TBL_SIZE                            (128)
#define DFRAME_FRAMEPOOL_INVALIDID                           (~0u)
#define BUFFER_READ_LEN         (512)
#define FRAME_BUFFER_LEN         (1920*1280/2)
enum H264NALTYPE{
    H264NT_NAL = 0,
    H264NT_SLICE,       /*1, p帧*/
    H264NT_SLICE_DPA,   /*2*/
    H264NT_SLICE_DPB,   /*3*/
    H264NT_SLICE_DPC,   /*4*/
    H264NT_SLICE_IDR,   /*5, I帧*/
    H264NT_SEI,         /*6, SEI*/
    H264NT_SPS,         /*7, SPS, sequence parameter set*/
    H264NT_PPS,         /*8, PPS, Piture parameter set*/
    H264NT_SINGLE_PKT = 100,
};

static int dframe_read_frame_h264(FILE* fp,char * dest)
{
    int rlen = 0, offset = 0, frm_step = 0;
    int i;
    char rcvbuf[BUFFER_READ_LEN];
    rlen = fread (rcvbuf,1,BUFFER_READ_LEN,fp);
    if(rlen < 0)return -1;
    while(rlen > 0)
    {
        for(i=0; i<(rlen-5); i++)
        {
            if((rcvbuf[i]==0) && (rcvbuf[i+1]==0) && (rcvbuf[i+2]==0) && (rcvbuf[i+3]==1)){
                if((rcvbuf[i+4]&0x1f) == H264NT_SLICE){
                    frm_step++;
                    if(frm_step == 2)break;
                }
                else if((rcvbuf[i+4]&0x1f) == H264NT_SLICE_IDR){
                    ;
                }
                else if((rcvbuf[i+4]&0x1f) == H264NT_SPS){
                    frm_step++;
                    if(frm_step == 2)break;
                }
                else if((rcvbuf[i+4]&0x1f) == H264NT_PPS){
                    ;
                }
            }
        }//the end of the file
        if(i == 0)
        {
            memcpy(dest+offset, rcvbuf, rlen);
            offset += rlen;
        }
        else
        {
            memcpy(dest+offset, rcvbuf, i);
            offset += i;
            fseek(fp, i-rlen, SEEK_CUR);
        }
        if(frm_step == 2)
        {
            break;
        }
        /*for next loop*/
        rlen = fread (rcvbuf,1,BUFFER_READ_LEN,fp);
        if(rlen < 0)return -1;
    }
    if((frm_step == 1 && rlen == 0) || (frm_step == 2))
    {
        return offset;
    }
    return -1;        
}
```
