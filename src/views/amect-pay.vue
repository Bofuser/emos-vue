<template>
	<el-dialog title="缴纳罚款" :close-on-click-modal="false" v-model="visible" width="305px" center>
		<img :src="qrCode" class="qrcode" v-if="!result" />
		<div v-if="result" class="pay-success">
			<el-result icon="success" title="付款成功" subTitle="请根据提示进行操作"></el-result>
		</div>
		<div class="dialog-footer-style">
			<el-button type="danger" size="medium" v-if="!result" @click="cancelHandle">取消支付</el-button>
			<el-button type="primary" size="medium" v-if="!result" @click="successHandle">支付成功</el-button>
			<el-button type="primary" size="medium" v-if="result" @click="closeHandle">关闭窗口</el-button>
		</div>
	</el-dialog>
</template>

<script>

export default {
	data: function() { 
		return {
			visible: false,		//true则使支付页面进行展示
			dataForm: {
				id: null
			},
			result: false,
			qrCode: null
		};
	},
	methods: {
		/**
		 * 支付初始化函数
		 * @param {} id 
		 */
		init: function(id){
			let that = this;
			that.visible = true;

			//传入要支付罚款的id参数 
			that.dataForm.id = id;
			that.result = false;

			that.$nextTick(() => {
				//从浏览器Cookie中获取Token令牌
				let token = that.$cookies.get('token');
				//向WebSocket服务类发送消息，让服务类缓存Session对象，可以推送消息给当前页面
				that.$socket.sendObj({ opt: 'pay_amect', token: token });
				//接收服务端推送的消息
				that.$options.sockets.onmessage = function(resp) {
					let data = resp.data;
					// console.log(data)
					if (data == '收款成功') {
						that.result = true;
					}
				};
				//获取二维码
				that.$http('amect/createNativeAmectPayOrder', 'POST', { amectId: id }, true, function(resp) {
					that.qrCode = resp.qrCodeBase64;
				});
			});

		},
		/**
		 * 取消支付页面，取消订单 
		 */
		cancelHandle: function(){
			//取消支付，关闭页面
			this.visible = false;
		},
		/**
		 * 支付成功页面
		 */
		 successHandle: function(){
			
			let that = this;
			//支付成功，关闭页面
			that.visible = false
			// 响应后端支付成功的结果
			that.$http('amect/searchNativeAmectPayResult', 'POST', { amectId: that.dataForm.id }, true, function (resp){
				//重新刷新列表
				that.$emit('refreshDataList');
			});
		},
		/**
		 * 关闭支付窗口
		 */
		closeHandle: function(){
			//关闭页面
			this.visible = false;
			this.$emit('refreshDataList');
		}
	}
};
</script>

<style scoped>
.qrCode {
	width: 255px;
	height: 255px;
}
.pay-success {
	width: 255px;
	height: 255px;
}
.dialog-footer-style {
	padding-bottom: 30px;
	padding-top: 10px;
	text-align: center;
}
</style>
