<template>
	<div class="page">
		<el-row type="flex" justify="center" align="middle" class="container">
			<el-col :lg="14" :xl="10" class="hidden-md-and-down">
				<el-row class="panel">
					<el-col :span="12">
						<div class="left">
							<img src="../assets/login/logo.png" class="logo" />
							<img src="../assets/login/big-1.png" class="big" />
						</div>
					</el-col>
					<el-col :span="12">
						<div class="right">
							<div class="title-container">
								<h2>Emos在线办公平台</h2>
								<span>( Ver 1.0 )</span>
							</div>
							<div v-if="!qrCodeVisible">
								<div class="row">
									<el-input
										v-model="username"		
										placeholder="用户名"
										prefix-icon="el-icon-user"
										clearable="clearable"
									/>
								</div>
								<div class="row">
									<el-input
										type="password"
										v-model="password"
										placeholder="密码"
										prefix-icon="el-icon-lock"
										clearable="clearable"
										@keyup.enter.enter = 'login'
									/>
								</div>
								<div class="row">
									<el-button type="primary" class="btn" @click="login" >登陆系统</el-button>
								</div>
								<div class="row"><a class="link" @click="changeMode">二维码登陆</a></div>
							</div>
							<!-- 二维码登录页面 -->
							<div v-if="qrCodeVisible">
								<div class="qrCode-container">
									<img :src="qrCode" height="153" class="qrCode" />
									<img src="../assets/login/phone.png" height="148" />
								</div>
								<div class="row"><a class="link" @click="changeMode">用户名密码登陆</a></div>
							</div>
						</div>
					</el-col>
				</el-row>
			</el-col>
		</el-row>
	</div>
</template>

<script>
import { isUsername, isPassword } from '../utils/validate.js';
import 'element-plus/lib/theme-chalk/display.css';
import router from '../router/index.js';
export default {
	data: function() {
		return {
			username: null,
			password: null,
			//以下为跟二维码登录相关的页面
			qrCodeVisible: false,
			qrCode: '',
			uuid: null,
			qrCodeTimer: null,
			loginTimer: null
		};
	},

	methods: {
		
		
		
		
		/**
		 * 登录用户函数，用来验证用户名和密码的格式，然后判断登录是否正确并从后端返回ajax的值
		 */
		login: function(){
			
			let that = this;
			//首先判断用户名和密码的校验格式是否正确
			if(!isUsername(that.username)){		//username为前端表格用户名中v-model的双向数据绑定，可以直接将用户名框中填写的值传到isUsername中进行格式校验
				that.$message({
					
					message: '用户名输入格式不正确',
					type: 'error',
					duration: 1200  //响应时间为1.2秒
					
				});
							
			}else if(!isPassword(that.password)){
				
				that.$message({
					
					message: '密码输入格式不正确',
					type: 'error',
					duration: 1200
					
				});
				
			}else{	//如果用户名和密码格式都正确，则判断用户和密码是否正确
			
				let data = { username: that.username, password: that.password };	//将用户和密码保存到data中，作为数组形式。
				
				//发送Ajax登录请求,true为异步请求，false设置为同步请求
				that.$http('user/login', 'POST', data, true, function(resp){
				
					//返回回调函数，将后端传回的result和pemission信息返回给前端页面
					//其中result用于返回true，用于检验数据库中是否有该用户,判断用户名和密码是否正确
					if(resp.result){
						//在浏览器的storage中存储用户权限列表，这样其他页面也可使用storage中的数据，实现共享
						let permissions = resp.permissions;
						//取出Token令牌，保存到storage中
						let token = resp.token;
						//取出密码
						let password = resp.password;
						//将身份数据保存到浏览器中，便于访问系统其他页面时不用重新登录
						localStorage.setItem('permissions',permissions);
						localStorage.setItem('token',token);
						localStorage.setItem("password", password)
						//让路由跳转到home页面中
						router.push({ name: 'Home' });
						
					}else {	//登录失败返回信息
						
						//$符号表示vm层中自带的属性message，为了区分自定义的message而设定的
						that.$message({
							
							message: '登录失败',
							type: 'error',
							duration: 1200
							
						});
						
					}										
				});														
			}
						
		},
		/**
		 * 改变登录方式，选择用户名或者密码登录
		 */
		changeMode: function() {
			let that = this;
			//二维码登录原理就是通过让qrCodeVisible赋值为true时则为二维码登录页面，否则将变成账号登录
			that.qrCodeVisible = !that.qrCodeVisible;	
			//加载二维码图片
			if (that.qrCodeVisible) {
				//加载二维码
				that.loadQRCode();
				//创建刷新二维码定时器,默认为5分钟。5分钟后重新刷新二维码
				that.qrCodeTimer = setInterval(function() {
					that.loadQRCode();
				}, 5 * 60 * 1000);
				
				//微信小程序扫码登陆，创建轮询定时器，每隔5秒钟发起Ajax请求，检查Redis中UUID对应的值
				that.loginTimer = setInterval(function() {
					 //调用Web方法，检查Redis中UUID对应的值，判定用户是否扫码登陆。该Web方法下面有定义。
					that.$http('user/wechatLogin', 'POST', { uuid: that.uuid },true, function(resp) {
						if (resp.result) {
							//登录成功销毁定时器
							clearInterval(that.qrCodeTimer);
							clearInterval(that.loginTimer);
							//缓存用户权限
							let permissions = resp.permissions;
							localStorage.setItem('permissions', permissions);
							//跳转页面
							router.push({ name: 'Home' });
						}
					});
				}, 5000);
			} else {
				
				//切换回账户登录，销毁刷新二维码定时器
				clearInterval(that.qrCodeTimer);
				clearInterval(that.loginTimer);
			}
		},
		
		
		//加载二维码图片的封装方法
		loadQRCode: function() {
			//向后端发起Ajax，获取二维码和uuid
			this.$http('user/createQrCode', 'GET', null,true, resp => {
				this.qrCode = resp.pic;
				this.uuid = resp.uuid;
			});
		}
	}
};
</script>

<style lang="less" scoped="scoped">
@import url('login.less');
</style>
