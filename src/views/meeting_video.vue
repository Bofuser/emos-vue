<template>
	<div>
		<el-row :gutter="15">
			<!-- 视频墙 -->
			<el-col :span="19">
				<div class="meeting-container">
					<el-scrollbar height="650px" id="videoListContainer">
						<!-- 当前用户列表排在视频墙第一位 -->
						<div class="video-list">
							<div class="video">
								<div class="user">
									<img class="photo" :src="mine.photo" />
									<span class="name">{{ mine.name }}{{ shareStatus ? '（共享中）' : '' }}</span>
								</div>
								<!-- 本地流表示将视频渲染到这里 -->
								<div id="localStream"></div>
							</div>
							<!-- 其他参会人 -->
							<div class="video" v-for="one in memberList">
								<div class="user">
									<img class="photo" :src="one.photo" />
									<span class="name">{{ one.name }}</span>
								</div>
								<div :id="one.id" class="remote-stream" @dblclick="bigVideoHandle(one.id)"></div>
							</div>
						</div>
					</el-scrollbar>
					<div id="videoBig" @dblclick="smallVideoHandle()"></div>
				</div>
				<!-- 会议下面的备注 -->
				<p class="desc">
					会议过程中，不需要发言的会场要主动将本地会场的MIC关闭，保证会场安静，当需要发言时要及时打开MIC。会议过程中，需要发言讨论时，先打开MIC向主会场提出请求，得到同意后再继续发言，否则请继续保持静音。发言时，要—个人一个人的发言，不要多人同时讲话，因为全向MIC会把所有人的声音混合，远端听到的声音会非常嘈杂，听不清具体说话内容。在会议进行过程中，尽量控制会场噪音，不要在会场中随意走动
				</p>
			</el-col>
			<!-- 用户列表区域 -->
			<el-col :span="5">
				<el-card shadow="never">
					<template #header>
						<div class="card-header"><span>用户列表</span></div>
					</template>
					<el-scrollbar height="555px">
						<ul class="user-list">
							<li v-for="one in userList">
								<!-- 话筒图标，用来显示说话人的话筒 -->
								<img class="mic" src="../assets/trtc/mic.png" />
								<div class="mic-container">
									<img
										class="mic-green"
										:id="'mic-' + one.userId"
										src="../assets/trtc/mic-green.png"
									/>
								</div>
								<!-- 用户列表显示的 “部门 - 人名” -->
								<span>{{ one.dept }} - {{ one.name }}</span>
							</li>
						</ul>
					</el-scrollbar>
				</el-card>
				<!-- 备注的旁边的点击功能事件 -->
				<div class="meeting-operate">
					<!-- 表示进入视频会议 -->
					<button :class="meetingStatus ? 'phone-btn-off' : 'phone-btn-on'" @click="phoneHandle"></button>
					<button :class="videoStatus ? 'video-btn-on' : 'video-btn-off'" @click="videoHandle"></button>
					<button :class="micStatus ? 'mic-btn-on' : 'mic-btn-off'" @click="micHandle"></button>
					<button :class="shareStatus ? 'share-btn-on' : 'share-btn-off'" @click="shareHandle"></button>
				</div>
			</el-col>
		</el-row>
	</div>
</template>

<script>
import TRTC from 'trtc-js-sdk';
import $ from 'jquery';
export default {
	data: function() {
		return {
			meetingId: null,
			uuid: null,
			appId: null,
			userSig: null,		//用户签名
			userId: null,		//用户Id
			roomId: null,		//房间id
			meetingStatus: false,	//判断用户是否进入会议室
			videoStatus: true,		//摄像头状态
			micStatus: true,		//麦克风状态
			shareStatus: false,
			userList: [], //进入会场的用户列表
			mine: {},
			memberList: [], //会议成员列表
			client: null,
			localStream: null,		//本地流
			shareStream: null,
			stream: {}, //所有的远端流
			bigVideoUserId: null //大屏显示远端流的用户ID，切换回小屏幕的时候使用
		};
	},
	methods: {
		
		//共享本地视频流
		shareHandle: function() {
		    let that = this;
		    //判断用户是否进入视频会议室
		    if (!that.meetingStatus) {
		        that.$alert('请先进入视频会议才能共享屏幕', '提示信息', {
		            confirmButtonText: '确定'
		        });
		        return;
		    }
		    //检查浏览器是否支持屏幕共享
		    if (!TRTC.isScreenShareSupported()) {
		        //提示当前浏览器不支持在线视频会议
		        this.$alert('当前浏览器不支持屏幕共享', '提示信息', {
		            confirmButtonText: '确定'
		        });
		        return;
		    }
		    that.shareStatus = !that.shareStatus;
		    //开启屏幕共享
		    if (that.shareStatus) {
		        //创建共享流
		        let shareStream = TRTC.createStream({
		            audio: that.micStatus,
		            screen: true,
		            userId: that.userId
		        });
		        shareStream.setScreenProfile('1080p');
		        that.shareStream = shareStream;
		        shareStream
		            .initialize()
		            .catch(error => {
		                console.error('初始共享流失败 ' + error);
		            })
		            .then(() => {
		                //取消推送本地视频流
		                that.client.unpublish(that.localStream).then(() => {
		                    that.localStream.close(); //关闭本地流
		                    that.localStream = null; //本地流设置为空
		                    //隐藏本地视频窗口
		                    $('#localStream').css({ 'z-index': -1 });
		                    that.client.publish(shareStream); //向远端推送共享流
		                });
		            });
		    }
		    //关闭屏幕共享
		    else {
		        //重建本地视频流
		        let localStream = TRTC.createStream({
		            userId: that.userId + '',
		            audio: that.micStatus,
		            video: that.videoStatus
		        });
		        that.localStream = localStream;
		        localStream.setVideoProfile('480p');
		        localStream
		            .initialize()
		            .catch(error => {
		                console.error('初始化本地流失败 ' + error);
		            })
		            .then(() => {
		                console.log('初始化本地流成功');
		                //取消共享流的推流
		                that.client.unpublish(that.shareStream).then(() => {
		                    that.shareStream.close(); //关闭共享流
		                    that.shareStream = null; //共享流设置为空
		                    //显示本地视频窗口
		                    $('#localStream').css({ 'z-index': 1 });
		                    localStream.play('localStream'); //播放本地流
		                    //向远端推送本地视频流
		                    that.client
		                        .publish(localStream)
		                        .catch(error => {
		                            console.error('本地流发布失败 ' + error);
		                        })
		                        .then(() => {
		                            console.log('本地流发布成功');
		                        });
		                });
		            });
		    }
		},
	
		//开关麦克风的回调函数
		micHandle: function() {
		    let that = this;
		    let stream = that.getStream();
		    if (that.micStatus) {
		        //关闭话筒
		        stream.muteAudio();
		    } else {
		        //开启话筒
		        stream.unmuteAudio();
		    }
		    that.micStatus = !that.micStatus;
		},

		
		//本地用户开关自己的摄像头和麦克风
		videoHandle: function() {
		    let that = this;
		    if (that.shareStatus) {
		        that.$alert('屏幕共享中无法开关摄像头', '提示信息', {
		            confirmButtonText: '确定'
		        });
		        return;
		    }
		    if (that.videoStatus) {
		        //关闭摄像头
		        that.localStream.muteVideo();
		    } else {
		        //开启摄像头
		        that.localStream.unmuteVideo();
		    }
		    that.videoStatus = !that.videoStatus;
		},

		
		//双击大屏视频，切换到小屏播放
		smallVideoHandle: function() {
		    let that = this;
		    //停止大屏播放远端视频
		    that.stream[that.bigVideoUserId].stop(); 
		    //在相应的小屏播放远端视频
		    that.stream[that.bigVideoUserId].play(that.bigVideoUserId); 
		    that.bigVideoUserId = null;
		    //隐藏大屏控件
		    $('#videoBig').hide();
		    //显示视频墙
		    $('#videoListContainer').show();
		},

		
		//双击某个远端小视频，切换到大屏显示
		bigVideoHandle: function(userId) {
		    let that = this;
		    //在模型层记录全屏显示的远端用户userId，切回小屏会用到
		    that.bigVideoUserId = userId; 
		    $('#videoListContainer').hide(); //隐藏视频墙
		    $('#videoBig').show(); //显示大屏控件
		    //停止该用户的远端视频在屏幕墙格子的播放
		    that.stream[userId].stop(); 
		    //远端视频在大屏播放
		    that.stream[userId].play('videoBig'); 
		},

		
		//用于判断当前本地使用的是本地流还是共享流，并返回相应的流对象
		getStream: function() {
		    let that = this;
		    let stream = null;
		    if (that.localStream != null) {
		        stream = that.localStream;
		    } else if (that.shareStream != null) {
		        stream = that.shareStream;
		    }
		    return stream;
		},

		
		//开启视频会议
		phoneHandle: function() {
			let that = this;
			
			//检查浏览器是否支持在线视频会议
			TRTC.checkSystemRequirements().then(checkResult => {
				if (!checkResult.result) {
					console.log('checkResult', checkResult.result, 'checkDetail', checkResult.detail);
					that.$alert('当前浏览器不支持在线视频会议', '提示信息', {
						confirmButtonText: '确定'
					});
				}else{
					
					//点击电话图标则进入会议，再点击一次则退出视频会议
					that.meetingStatus = !that.meetingStatus;
					//如果参会人员点击电话
					if(that.meetingStatus){
						//记录摄像头和话筒的状态
						that.videoStatus = true;
						that.micStatus = true;
						//TRTC日志输出级别为Error
						TRTC.Logger.setLogLevel(TRTC.Logger.LogLevel.ERROR);
						//生成用户签名
						that.$http("meeting/searchMyUserSig", "GET", {}, false, function(resp){
							//resp{userSig,userId,appId}
							that.userSig = resp.userSig;
							that.userId = resp.userId;
							that.appId = resp.appId;  
						});
						//1.创建TrtcClient对象
						let client = TRTC.createClient({ 
							mode: 'rtc', 
							sdkAppId: that.appId, 
							userId: that.userId + '', 
							userSig: that.userSig ,
						});
						that.client = client;
						
						//监听新增远端流事件(用户进入会议时出发)
						client.on('stream-added', event => {
							//获取远端流
							let remoteStream = event.stream;
							//订阅远端流
							client.subscribe(remoteStream);
							//从远端流获取远程用户userId(创建TrtcClient对象时候的参数)
							let userId = remoteStream.getUserId();
							
							//把新上线的用户添加到页面右侧的在线人员列表中
							that.$http("user/searchNameAndDept", "POST", { id: userId }, true, function(resp){
								
								//将后端查询的数据存储到userList中
								let name = resp.name;
								let dept = resp.dept;
								that.userList.put({ userId: userId, name: name, dept: dept });
								
							});
							
							
							//把远端流保存在模型层JSON中，将来大屏显示的时候要找到某个远端流停止小窗口播放，切换到大窗口播放
							that.stream[userId] = remoteStream;
						});
						
						//检测订阅事件是否订阅成功
						client.on('stream-subscribed', event => {
							//获取远端流和用户Id
							let remoteStream = event.stream;
							let userId = remoteStream.getuserId();
							//找到视频墙中某个远端用户的格子，把其中用于显示视频的DIV，置顶覆盖用户信息
							$('#' + userId).css({ 'z-index': 1 }); 
							//在这个置顶的DIV中播放远端音频讯号
							remoteStream.play(userId + '');
							
							
						});
						
						//订阅远端删除流事件（远端用户退出会议室）
						client.on('stream-removed', event => {
							
							let remoteStream = event.stream;
							//取消订阅该远端流
							client.unsubscribe(remoteStream);
							//获取远端流中的userId
							let userId = remoteStream.getUserId();
							
							//在页面右侧的用户列表中删除该用户
							//上线用户列表中删除该用户
							let i = that.userList.findIndex(function(one) {
								
								return one.userId == userId;
							});
							//删除userList数组中的某一个用户
							that.userList.splice(i,1);
							//停止播放远端流视频，并且关闭远端流
							remoteStream.stop();
							remoteStream.close();
							
							//删除模型层JSON中保存的远端流对象
							delete that.stream[userId];
							
							//把视频墙中该用户格子的视频DIV控件置底，显示用户基本信息
							$('#' + userId).css({ 'z-index': '-1' });
							$('#' + userId).html('');
							
						});
						
						//订阅语音事件（无论本地还是远端说话，都会触发这个事件）
						client.on('audio-volume', event => {
							
						    event.result.forEach(({ userId, audioVolume, stream }) => {
						        //说话声音超过5，就设置话筒音量动画
						        if (audioVolume > 5) {
						            $('#mic-' + userId).css('top', `${100 - audioVolume * 3}%`);
						        } else {
						            $('#mic-' + userId).css('top', `100%`);
						        }
						    });
						});
						// 开启音量回调函数，并设置每 30ms 触发一次事件
						client.enableAudioVolumeEvaluation(30);

						
						//2.进入音视频通话房间
						client.join({ roomId: that.roomId })
						.then(() => {
							
							//发起Ajax执行会议签到
							that.$http("meeting/updateMeetingPresent", "POST", {meetingId: that.meetingId}, true, function(resp){
								//resp{rows}
								let rows = resp.rows;
								if(rows == 1){
									
									that.$message({
										message: "签到成功",
										type: "success",
										duration: 1200
									});
									
								}
								
							});
							//成功进入会议室，然后创建本地流
							let localStream = TRTC.createStream({
								userId: that.userId + '',
								audio: true,
								video: true
							});
							that.localStream = localStream;
							localStream.setVideoProfile('480p'); //设置分辨率
	
							//把自己添加到上线用户列表中
							that.$http("user/searchNameAndDept", "POST", { id: that.userId }, true, function(resp){
								
								//将后端查询的数据存储到userList中
								let name = resp.name;
								let dept = resp.dept;
								that.userList.put({ userId: that.userId, name: name, dept: dept });
								
							});
	
	
							//初始化本地音视频流
							localStream
								.initialize()
								.catch(error => {
									console.error('初始化本地流失败 ' + error);
								})
								.then(() => {
									console.log('初始化本地流成功');
									//视频墙中第一个格子中的视频DIV置顶
									$('#localStream').css({ 'z-index': 1 });
									//播放本地音视频流
									localStream.play('localStream'); 
									
									
									//向远端用户推送本地流
									client
										.publish(localStream)
										.catch(error => {
											console.error('本地流发布失败 ' + error);
										})
										.then(() => {
											console.log('本地流发布成功');
										});
								});
						})
						.catch(error => {
							console.error('进入房间失败: ' + error);
						});
					}else{
						
						//关闭视频会议
						//获取当前本地使用的流，有可能是本地流或者共享流
						let stream = that.getStream(); 
						that.client.unpublish(stream).then(() => {
						    // 取消发布本地流成功
						    that.client
						        .leave()
						        .then(() => {
						            console.log('成功退出会议室');
						            //关闭本地流或者共享流
						            stream.stop();
						            stream.close();
									      //清空模型层的本地流
						            that.localStream = null;
						            that.shareStream = null;
									      //清空模型层的远端流
						            that.stream = {};
									      //销毁TrtcClient对象
						            that.client = null;
						            that.userList = []; //清空用户列表
						            that.videoStatus = true;
						            that.micStatus = true;
						            that.shareStatus = false;
									      //视频墙上本地流DIV区域置底
						            $('#localStream').css({ 'z-index': '-1' });
						            $('#localStream').html('');

									//如果是播放大屏视频的时候退出会议，退出会议后需要隐藏大屏，然后显示视频墙界面
									if (that.bigVideoUserId != null) {
									    $('#videoBig').hide();
									    $('#videoListContainer').show();
									    that.bigVideoUserId = null;
									}

									
									
						        })
						        .catch(error => {
						            console.error('成功退出会议室失败' + error);
						        });
						});

					}
					
				}
				
			})
			
		}
		
		
	},
	
	created: function() {
	    let that = this;
	    //路由传入的参数
	    let params = that.$route.params;
	    that.meetingId = params.meetingId;
	    that.uuid = params.uuid;
	    let data = { meetingId: that.meetingId };
	    //查询会议所有参会人，并且生成视频墙
	    that.$http('meeting/searchOnlineMeetingMembers', 'POST', data, true, function(resp) {
	        let list = resp.list;
	        if (list != null && list.length > 0) {
	            //取出当前用户的信息
	            that.mine = list[0];
	            //其他参会人信息保存在memberList里面
	            for (let i = 1; i < list.length; i++) {
	                that.memberList.push(list[i]);
	            }
	        }
	    });
	  
	    data = {
	        uuid: that.uuid
	    };
	    // 查询视频会议室的ID
	    that.$http('meeting/searchRoomIdByUUID', 'POST', data, true, function(resp) {
	        if (resp.roomId == null) {
	            that.$message({
	                message: '不存在视频会议室',
	                type: 'error',
	                duration: 1200
	            });
	        } else {
	            that.roomId = resp.roomId;
	        }
	    });
	}

}
</script>

<style lang="less" scoped="scoped">
@import url('meeting_video.less');
</style>
