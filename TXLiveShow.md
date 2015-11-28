##AVContext
对象代表了一个SDK运行的实例，可以理解为一个上下文。
AVContext对内负责管理SDK的工作线程，控制各种内部对象的生命周期；
 对外提供了一系列的接口方法，App可以通过AVContext的成员函数进一步访问SDK的其他对象。
##AVMultiRoomDelegate
 多人房间回调代理，`AVMultiRoomDelegate(controller)`controller接受这些事件来通知应用层.
##AVEndpoint
表示位于房间中的一个成员
##AVDeviceMgr
设备管理类
##AVDevice
 + 可以调用AVContext::GetAudioCtrl()来获得音频控制器对象。
    音频控制器对象提供了一组接口用来调节会话中的音量：GetVolume()、SetVolume()、GetDynamicVolume()
 + 可以调用AVContext::GetVideoDeviceMgr()获得视频设备管理器对象的引用
 
 
 ------

#流程
##观众:
 + 创建MultiRoomManager()实例，里面会配置Config参数  -> CreateContext(config) ，创建AVContext对象 -> avStartContext(),开启上下文
 + -OnContextStartComplete(result)回调 
 	+ joinGroup()
 	+ avCreateRoom(),内部调用EnterRoom()，进入房间
 	+ OnRoomCreateComplete（result）回调 
 		+ if result == true 
 		    +  get innerRoomId(旁路推流？) 
 		    +  AVAudioCtrl(控制麦克风和扬声器)	
 		+ false -> avExitRoom() 

 			
 		+ 通知服务器，我进入房间
 			+  true -> avRequestViewPhone(liveUserPhone)
 			+ VideoframeDataCallback(frameData)回调，_imageRender.render.displayVideoFrame()渲染帧
 			+ 请求当前主播画面
 			<pre><code>AVRoomMulti* multiRoom = self.avContext->GetRoom();
 			AVEndpoint* endpoint = multiRoom->GetEndpointById(phone);
 			if endPoint && endPoint->RequestView() == OK
 				+ true 调用请求画面成功 ，OnRequestViewComplete（）回调
 					+ sendAddUserMessage()
 					+ _imageRender.changeToRender()渲染画面
 				+ false 调用请求画面失败, avExitRoom()
 			
 </code></pre>
 			+  false -> avExitRoom()
 		
##主播:
 + 向服务器请求直播房间号 -> 创建AVContext,开启上下文
 + -OnContextStartComplete(result)回调 
 	1. createGroup()
 	+ avCreateRoom(),内部调用EnterRoom()，默认第一个进入房间的就是创建者.
 	+ OnRoomCreateComplete（result）回调 
 		+ if result == true 
 		    +  get innerRoomId(旁路推流？) 
 		    +  AVAudioCtrl(控制麦克风和扬声器)	
 		+ false -> avExitRoom() 
 		+ insertLivingData() -> avEnableCamera(),启用摄像头
 			+OnEnableCameraComplete()
 		+ 向服务器发送心跳.
 + 房间用户更新，-OnRoomEndpointsUpdate(),dosomething


##说明

avExitRoom() ：quitGroup()/deleteGroup , avStopContext()，停止上下文
 	 