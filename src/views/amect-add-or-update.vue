<template>
	<!-- el-dialog 的title属性显示的是新增框中的命名 -->
	<!-- close-on-click-modal 属性表示单击显示框外面是否会关闭它 -->
	<!-- v-model表示该新增显示框是否可以展示 -->
	<el-dialog
		:title="!dataForm.id ? '新增罚款记录' : '修改罚款记录'"
		:close-on-click-modal="false"
		v-model="visible"
		width="692px"
	>
		<!-- 表单中添加信息 -->
		<el-form :model="dataForm" ref="dataForm" :rules="dataRule" label-width="60px">
			<!-- 这是处罚类型 -->
			<el-form-item label="类型" prop="typeId">
				<el-select v-model="dataForm.typeId" placeholder="罚款类型" size="medium" style="width:40%">
					<el-option v-for="one in amectTypeList" :key="one.id" :label="one.type" :value="one.id" />
				</el-select>
				<span class="desc">必须选择一个罚款类型，罚款金额可以自动生成</span>
			</el-form-item>
			<!-- 这是处罚金额 -->
			<el-form-item label="金额" prop="amount">
				<el-input v-model="dataForm.amount" size="medium" style="width:40%" placeholder="罚款金额" clearable />
				<span class="desc">元</span>
			</el-form-item>
			<!-- 这是处罚的原因 -->
			<el-form-item label="原因" prop="reason">
				<el-input
					type="textarea"
					:rows="2"
					placeholder="罚款原因"
					v-model="dataForm.reason"
					size="medium"
					resize="none"
					maxlength="200"
					show-word-limit
					clearable="clearable"
				/>
			</el-form-item>
			<!-- 这是添加处罚的成员名单 -->
			<el-form-item label="成员" prop="members" v-if="dataForm.id == 0">
				<el-transfer
					v-model="dataForm.members"
					:data="users"
					:titles="['员工', '当事人']"
					filterable
					filter-placeholder="请输入姓名"
				/>
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
				typeId: null,
				amount: null,
				reason: null,
				members: []
			},
			amectTypeList: [],
			users: [],
			dataRule: {	//设置添加信息中的正则表达式
				typeId: [{ required: true, message: '罚款类型为必填' }],
				amount: [
					{
						required: true,
						pattern: '(^[1-9]([0-9]+)?(\.[0-9]{1,2})?$)|(^(0){1}$)|(^[0-9]\.[0-9]([0-9])?$)',
						message: '罚款金额格式错误'
					}
				],
				reason: [{ required: true, message: '罚款原因为必填' }],
				members: [
					{ required: true, trigger: 'blur', message: '必须设置当事人' },
					{ required: false, trigger: 'change', message: '必须设置当事人' }
				]
			}
		};
	},
	methods: {
		/**
		 * 该函数用于初始化表单
		 * @param {} id 
		 */
		init: function(id) {
			let that = this;
			//将id值赋值给dataForm，没有则将0赋值给它
			that.dataForm.id = id || 0;
			//设置visible的值，使弹框显示
			that.visible = true; 
			//nextTick表示更新修改的数据，每次修改只执行一次
			that.$nextTick(() => {
				//它将表单的所有字段重置为其初始值
				that.$refs['dataForm'].resetFields();
				//这个方法获取罚款类型，将类型值初始化到表单中
				that.$http('amect_type/searchAllAmectType', 'GET', {}, true, function(resp) {
					that.amectTypeList = resp.list;
				});
				//将用户信息返回在成员表单中
				that.$http('user/searchAllUser', 'GET', null, true, function(resp) {
					let temp = [];
					for (let one of resp.list) {
						temp.push({ key: one.id, label: one.name });
					}
					that.users = temp;
				});
				//根据id的有无判断是“新增”还是“修改”，修改则将该值赋值到id中
				if (id) {
					that.$http('amect/searchById', 'POST', { id: id }, true, function(resp) {
						that.dataForm.typeId = resp.typeId;
						that.dataForm.amount = resp.amount+"";
						that.dataForm.reason = resp.reason;
					});
				}
			});
		},

		/**
		 * 表单提交的信息
		 */
		dataFormSubmit: function(){
			let that = this;
			//表单数据进行提交
			let data = {
				userId: that.dataForm.members,
				amount: that.dataForm.amount,
				typeId: that.dataForm.typeId,
				reason: that.dataForm.reason
			};

			//如果dataForm中有id，将其赋值给data为data中的id
			if(that.dataForm.id){
				data.id = that.dataForm.id;
			}

			//罚款人名单，将members 对象转换为JSON字符串
			//data.members = JSON.stringify(that.dataForm.members);
			//校验表单
			that.$refs["dataForm"].validate(valid => {

				//如果表单数据为true，执行下列语句
				if(valid){
					//发送 ajax
					that.$http(`amect/${!that.dataForm.id ? 'insert' : 'update'}`, "POST", data, true, function(resp){

						// 后端返回的是rows
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
								message: '操作失败',
								type: 'error',
								duration: 1200
							});
						}


					});
				}

			});

		},

		
	}
};
</script>

<style>
.desc {
	margin-left: 15px;
}
</style>
