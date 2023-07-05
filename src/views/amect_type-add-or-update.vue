<template>
	<el-dialog
		:title="!dataForm.id ? '新增' : '修改'"
		v-if="isAuth(['ROOT'])"
		:close-on-click-modal="false"
		v-model="visible"
		width="420px"
	>
		<!-- 需要新增（或修改）的内容 -->
		<el-form :model="dataForm" ref="dataForm" :rules="dataRule" label-width="80px">
			<el-form-item label="违纪类型" prop="type">
				<el-input v-model="dataForm.type" size="medium" style="width:100%" clearable />
			</el-form-item>
			<el-form-item label="罚款金额" prop="money">
				<el-input v-model="dataForm.money" size="medium" style="width:100%" clearable />
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
export default {
	data: function() {
		return {
			visible: false,
			dataForm: {
				id: null,
				type: null,
				money: null
			},
			dataRule: {
				type: [{ required: true, pattern: '^[a-zA-Z0-9\u4e00-\u9fa5]{2,10}$', message: '违纪类型格式错误' }],
				money: [
					{
						required: true,
						pattern: '(^[1-9]([0-9]+)?(\.[0-9]{1,2})?$)|(^(0){1}$)|(^[0-9]\.[0-9]([0-9])?$)',
						message: '罚款金额格式错误'
					}
				]
			}
		};
	},
	methods: {
		/**
		 * 该函数主要用于初始化表单，id 用来判断是添加函数还是修改函数
		 * @param {} id 
		 */
		init: function(id) {
			let that = this;
			that.dataForm.id = id || 0;
			that.visible = true;
			that.$nextTick(() => {
				that.$refs['dataForm'].resetFields();
				if (id) {
					that.$http('amect_type/searchById', 'POST', { id: id }, true, function(resp) {
						that.dataForm.type = resp.type;
						that.dataForm.money = resp.money+"";
					});
				}
			});
		},

		/**
		 * 提交表单函数
		 */
		dataFormSubmit: function(){

			let that = this;
			//表单提交数据 
			let data = {
				type: that.dataForm.type,
				money: that.dataForm.money
			};

			//如果dataForm中有id，将其赋值给data为data中的id
			if(that.dataForm.id){
				data.id = that.dataForm.id;
			}

			//校验表单
			that.$refs["dataForm"].validate(valid => {

				//表单校验成功
				if(valid){
					//发送Ajax
					that.$http(`amect_type/${!this.dataForm.id ? 'insert' : 'update'}`, "POST", data, true, function(resp){

						//后端返回的是rows
						if(resp.rows > 0){
							that.$message({
								message: '操作成功',
								type: 'success',
								duration: 1200
							})
							//关闭弹窗
							that.visible = false;
							//重新刷新列表，其中refreshDataList为自定义事件，$emit()用于触发自定义事件，其作用是将这个事件发送出去，让监听的事件做出相应的处理
							//该自定义事件在父组件中amect.vue中展示
							that.$emit('refreshDataList');

						}else{
							that.$message({
								message: '操作失败'
							})
						}


					})


				}


			});




		}
		
	}
};
</script>

<style lang="less" scoped="scoped"></style>
