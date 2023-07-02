<template>
	<el-dialog title="提示" v-model="visible" width="25%">
		<el-form :model="dataForm" :rules="dataRule" ref="dataForm" label-width="80px">
			<el-form-item label="原密码" prop="password">
				<el-input type="password" v-model="dataForm.password" size="medium" clearable />
			</el-form-item>
			<el-form-item label="新密码" prop="newPassword">
				<el-input type="password" v-model="dataForm.newPassword" size="medium" clearable />
			</el-form-item>
			<el-form-item label="确认密码" prop="confirmPassword">
				<el-input type="password" v-model="dataForm.confirmPassword" size="medium" clearable />
			</el-form-item>
		</el-form>
		<template #footer>
			<span class="dialog-footer">
				<el-button size="medium" @click="visible = false">取消</el-button>
				<el-button type="primary" size="medium" @click="dataFormSubmit">确定</el-button>
			</span>
		</template>
	</el-dialog>
</template>

<script>
	/**
	 * 这个update-password.vue不是页面，而是一个在main.vue中的弹框。通过visible = false将其隐藏起来，等待点击修改密码时visible = true
	 * 则将会显示到main.vue页面中间。
	 */
	
	
export default {
	data() {
		//判断输入的密码是否和原密码一致
		const validatePassword = (rule, value, callback) => {
			if (value != localStorage.getItem("password")) {
				callback(new Error('旧密码不正确！'));
			} else {
				callback();
			}
		};
				
		//输入的两次新密码密码是否一致
		const validateConfirmPassword = (rule, value, callback) => {
			if (value != this.dataForm.newPassword) {
				callback(new Error('两次输入的密码不一致'));
			} else {
				callback();
			}
		};

		return {
			visible: false,
			dataForm: {		//通过v-modle将数据保存到dataForm这里
				password: '',
				newPassword: '',
				confirmPassword: ''
			},
			dataRule: {		//验证规则，通过prop属性将输入的校验规则在上面中检验
				password: [
					{ required: true, pattern: '^[a-zA-Z0-9]{6,20}$', message: '密码格式错误' },
					{ validator: validatePassword, trigger: 'blur' }
					],
				newPassword: [
					{ required: true, pattern: '^[a-zA-Z0-9]{6,20}$', message: '密码格式错误' },
					
				],
				confirmPassword: [
					{ required: true, pattern: '^[a-zA-Z0-9]{6,20}$', message: '密码格式错误' },
					{ validator: validateConfirmPassword, trigger: 'blur' }		//validateConfirmPassword为上面自定义检验两个输入的密码是否保持一致。
				]
			}
		};
	},
	methods: {
		
		//init用于清空弹框中的数据，使其保持空白
		init: function() {
			this.visible = true;	//显示弹窗
			 //因为清空表单控件是异步的，所以把清空表单控件放在下次DOM更新循环中
			this.$nextTick(() => {
				this.$refs['dataForm'].resetFields();
			});
		},
		
		//用于提交修改密码的表单
		dataFormSubmit: function(){
			
			let that = this;
			
			//前端表单校验
			that.$refs['dataForm'].validate(function(valid) {
				
				if(valid){		//判断前端验证的参数的值为true则执行下面语句
					
					//获取最终确认的密码
					let data = { password : that.dataForm.confirmPassword };
					//发送ajax
					that.$http('user/updatePassword', 'POST', data, true, function(resp){						
						//从后端传过来的rows，表示修改的条数
						if (resp.rows == 1) {
							that.$message({
								message: '密码修改成功',
								type: 'success',
								duration: 1200,
							});
							that.visible = false;
						} else {
							that.$message({
								message: '密码修改失败',
								type: 'error',
								duration: 1200,
							});
						}												
					});										
				}								
			});			
		}		
	}
};
</script>

<style></style>
