<template>
	<el-dialog
		:title="!dataForm.id ? '新增' : '修改'"
		v-if="isAuth(['ROOT', 'MEETING_ROOM:INSERT', 'MEETING_ROOM:UPDATE'])"
		:close-on-click-modal="false"
		v-model="visible"
		width="430px"
	>
		<el-form :model="dataForm" ref="dataForm" :rules="dataRule" label-width="80px">
			<el-form-item label="房间名称" prop="name">
				<el-input v-model="dataForm.name" size="medium" style="width:100%" clearable />
			</el-form-item>
			<el-form-item label="人数上限" prop="max">
				<el-input v-model="dataForm.max" size="medium" style="width:100%" clearable />
			</el-form-item>
			<el-form-item label="备注" prop="desc">
				<el-input v-model="dataForm.desc" style="width:100%" size="medium" maxlength="20" clearable />
			</el-form-item>
			<el-form-item v-if="dataForm.id" label="状态">
				<el-select v-model="dataForm.status" class="input" placeholder="状态" size="medium">
					<el-option label="可使用" value="1" />
					<el-option label="已停用" value="0" />
				</el-select>
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
				name: null,
				max: null,
				desc: null,
				status: 1
			},
			dataRule: {
				name: [{ required: true, pattern: '^[a-zA-Z0-9\u4e00-\u9fa5]{2,20}$', message: '会议室名称格式错误' }],
				max: [{ required: true, pattern: '^[1-9]\\d{0,4}$', message: '数字格式错误' }]
			}
		};
	},

	methods: {
		
		/**
		 * 初始化表单，通过id来返回insert或者update
		 * @param {Object} id
		 */
		init: function(id) {
			let that = this;
			//判断是否传入id，进而选择是insert还是update
			that.dataForm.id = id || 0;
			//通过v-model的方式控制表单的隐藏和显示
			that.visible = true;
			//初始化表单的内容
			that.$nextTick(() => {
				that.$refs['dataForm'].resetFields();
				//当传入值有id时，即if条件成立，调用后端查询该id所绑定的数据
				if (id) {
					that.$http('meeting_room/searchById', 'POST', { id: id }, true,function(resp) {
						that.dataForm.name = resp.name;
						that.dataForm.max = resp.max+"";
						that.dataForm.desc = resp.desc;
						that.dataForm.status=resp.status+"";
					});
				}
			});		
		},
		
		
		/**
		 * 新增或修改的提交表单
		 */
		dataFormSubmit: function(){
			
			let that = this;
			
			//注意：这里可以直接调用dataForm表单，将值通过Ajax传入过去
			
			that.$refs["dataForm"].validate(valid => {
				
				//如果校验通过，则可直接发送表单
				if(valid){
					
					/**
					 * 将dataForm中的参数发送ajax过去，返回rows则表示添加成功
					 * @param {Object} resp
					 */
					that.$http(`meeting_room/${!that.dataForm.id ? 'insert' : 'update'}`, "POST", that.dataForm, true, function(resp){
						
						//后端发送Ajax为rows
						if(resp.rows == 1){
							
							that.$message({
								
								message: "操作成功",
								type: "success",
								duration: 1200
								
							});
							
							//隐藏添加表单
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

<style lang="less">
.el-form {
	margin-left: 0px;
	margin-right: 10px;
}
.el-dialog__footer {
	margin-right: 10px;
}
.note {
	margin-left: 20px;
	color: #999;
}
</style>
