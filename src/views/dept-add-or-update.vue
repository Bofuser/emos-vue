<template>
	<el-dialog
		:title="!dataForm.id ? '新增' : '修改'"
		v-if="isAuth(['ROOT', 'DEPT:INSERT', 'DEPT:UPDATE'])"
		:close-on-click-modal="false"
		v-model="visible"
		width="420px"
	>
		<el-form :model="dataForm" ref="dataForm" :rules="dataRule" label-width="60px">
			<el-form-item label="部门" prop="deptName">
				<el-input v-model="dataForm.deptName" size="medium" style="width:100%" clearable />
			</el-form-item>
			<el-form-item label="电话" prop="tel">
				<el-input v-model="dataForm.tel" size="medium" style="width:100%" clearable />
			</el-form-item>
			<el-form-item label="邮箱" prop="email">
				<el-input v-model="dataForm.email" size="medium" style="width:100%" clearable />
			</el-form-item>
			<el-form-item label="备注" prop="desc">
				<el-input v-model="dataForm.desc" style="width:100%" size="medium" maxlength="20" clearable />
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
				deptName: null,
				tel: null,
				email: null,
				desc: null
			},
			dataRule: {
				deptName: [
					{ required: true, pattern: '^[a-zA-Z0-9\u4e00-\u9fa5]{2,10}$', message: '部门名称格式错误' }
				],
				tel: [
					{
						required: false,
						pattern: '^1\\d{10}$|^(0\\d{2,3}\-){0,1}[1-9]\\d{6,7}$',
						message: '电话格式错误'
					}
				],
				email: [
					{
						required: false,
						pattern: '^([a-zA-Z]|[0-9])(\\w|\-)+@[a-zA-Z0-9]+\.([a-zA-Z]{2,4})$',
						message: '邮箱格式错误'
					}
				]
			}
		};
	},

	methods: {
		init: function(id) {
			let that = this;
			//当传入id值存在则为id,否则id值为0
			that.dataForm.id = id || 0;
			//通过v-model的方式控制表单的隐藏和显示
			that.visible = true;
			
			that.$nextTick(() => {
				//调用restFields函数会将表单所填写的数据重置为初始值并且移除表单校验的结果。
				that.$refs['dataForm'].resetFields();
				//通过id是否有值来判断是add还是update，有值则判断为update，进而调用下列值将数据渲染到表单中
				if (id) {
					that.$http('dept/searchById', 'POST', { id: id },true, function(resp) {
						that.dataForm.deptName = resp.deptName;
						that.dataForm.tel = resp.tel;
						that.dataForm.email = resp.email;
						that.dataForm.desc = resp.desc;
					});
				}
			});
		},
		
		dataFormSubmit: function(){
			
			let that = this;
			//注意：这里可以直接调用dataForm表单，将值通过Ajax传入过去
			
			//校验表单
			that.$refs["dataForm"].validate(valid => {
				
				if(valid){
					
					/**
					 * 将dataForm中的参数发送ajax过去，返回rows则表示添加成功
					 * @param {Object} resp
					 */
					that.$http(`dept/${!that.dataForm.id ? 'insert' : 'update'}`, "POST", that.dataForm, true, function(resp){
						//后端发送Ajax为rows
						if(resp.rows == 1){
							
							that.$message({
								
								message: "操作成功",
								type: "success",
								duration: 1200
								
							});
							
							//提交表单后隐藏表单
							that.visible = false;
							/**
							 * 重新加载表单, dataFormSubmit事件触发后，自动触发refreshDataList事件,由其父组件role调用，
							 * 即@refreshDataList=loadDataList,重新加载表单
							 */
							that.$emit('refreshDataList');
							
						}else{
							
							that.$message({
								
								message: "操作失败",
								type: "error",
								duration: 1200
								
							});
							
						}
						
						
					});
					
				}
				
				
			});
			
			
		}
		
	}
};
</script>

<style lang="less" scoped="scoped">
</style>
