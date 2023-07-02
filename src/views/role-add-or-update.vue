<template>
	<el-dialog
		:title="!dataForm.id ? '新增' : '修改'"
		v-if="isAuth(['ROOT', 'ROLE:INSERT', 'ROLE:UPDATE'])"
		:close-on-click-modal="false"
		v-model="visible"
		custom-class="dialog"
		width="750px"
		
	>
		<el-form :model="dataForm" ref="dataForm" :rules="dataRule" label-width="60px" >
			<el-form-item label="角色" prop="roleName">
				<el-input v-model="dataForm.roleName" size="medium" style="width:50%" :readonly="systemic" clearable />
				<span class="note">（ 角色名称必须是2~10个字符之间 ）</span>
			</el-form-item>
			<el-form-item label="备注" prop="desc">
				<el-input v-model="dataForm.desc" style="width:50%" size="medium" maxlength="20" clearable />
				<span class="note">（ 备注信息必须是20个字符以内 ）</span>
			</el-form-item>
			<el-form-item label="权限" prop="permissions">
				<el-transfer
					v-model="dataForm.permissions"
					:data="permisionList"
					size="medium"
					:titles="['权限列表', '具备权限']"
					filterable
					filter-placeholder="请输入权限"
					
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
			systemic: true,		//标记角色是否为系统内置，弹窗是否会用到该变量
			dataForm: {
				id: null,
				roleName: null,
				permissions: [],	//用户选中的权限
				desc: null,
				changed: false
			},
			permisionList: [],		//所有权限
			oldPermissions: [],		//该角色拥有的权限
			dataRule: {
				roleName: [
					{ required: true, pattern: '^[a-zA-Z0-9\u4e00-\u9fa5]{2,10}$', message: '角色名称格式错误' }
				],
				permissions: [
					{ required: true, trigger: 'blur', message: '角色没有关联权限' },
					{ required: false, trigger: 'change', message: '角色没有关联权限' }
				]
			}
		};
	},

	methods: {
		init: function(id, systemic) {
			let that = this;
			//当传入id值存在则为id,否则id值为0
			that.dataForm.id = id || 0;
			//判断登录用户的权限来确定readonly的值，readonly = systemic;
			that.systemic = systemic;
			that.visible = true;
			that.$nextTick(() => {
				//调用restFields函数会将表单所填写的数据重置为初始值并且移除表单校验的结果。
				that.$refs['dataForm'].resetFields();
				let defaultPermissions = [];
				//当从role.vue中传入id值时，判断其为修改用户，则发送Ajax从后端获取数据
				if (id) {
					//
					that.$http('role/searchById', 'POST', { id: id }, false, function(resp) {
						//后端响应的数据渲染在dataForm中，进而渲染在表格中，方便管理员修改
						that.dataForm.roleName = resp.roleName;
						that.dataForm.desc = resp.desc;
						that.dataForm.permissions = JSON.parse(resp.permissions);
						//保存原始权限数据
						that.oldPermissions = JSON.parse(resp.permissions);
						defaultPermissions = resp.defaultPermissions;
					});
				}
				//将权限值赋给permisionList，渲染到列表中。查询系统所有权限数据，用于生成左侧数组
				that.$http('permission/searchAllPermission', 'GET', null, true, function(resp) {
					let temp = [];
					//遍历所有系统权限
					for (let one of resp.list) {
						let disabled = false;
						//判断要修改的是否为系统内置角色
						if (that.systemic) {
							//如果默认权限数组中含有某个权限，那么disabled设置为true，禁止取消选中
							disabled = defaultPermissions.includes(one.id);
						}
						//生成左侧数组元素值
						temp.push({ key: one.id, label: `${one.moduleName}（${one.actionName}）`, disabled: disabled });
					}
					//更新左侧数组
					that.permisionList = temp;
				});
			});
		},
		
		dataFormSubmit: function() {
		    let that = this;
		    this.$refs['dataForm'].validate(valid => {
		        if (valid) {
		            //因为用户是随机选择权限，所以对选中的权限排序
		            that.dataForm.permissions.sort(function(a, b) {
		                return a - b;
		            });
		            //判断用户选择的权限和原来的权限是否一致？把数组转换成字符串，简化比较两个数组是否一致
		            if (that.dataForm.permissions.join() != that.oldPermissions.join()) {
		                that.dataForm.changed = true;
		            } else {
		                that.dataForm.changed = false;
		            }
					//将dataForm中的值发送到后端
		            that.$http(`role/${!that.dataForm.id ? 'insert' : 'update'}`, 'POST', that.dataForm, true, function(
		                resp
		            ) {
		                if (resp.rows == 1) {
		                    that.$message({
		                        message: '操作成功',
		                        type: 'success',
		                        duration: 1200
		                    });
							//发送完毕后隐藏表单
		                    that.visible = false;
							/**
							 * 重新加载表单, dataFormSubmit事件触发后，自动触发refreshDataList事件,由其父组件role调用，
							 * 即@refreshDataList=loadDataList,重新加载表单
							 */
		                    that.$emit('refreshDataList');
		                } else {
		                    that.$message({
		                        message: '操作失败',
		                        type: 'error',
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
.note {
	margin-left: 20px;
	color: #999;
}


</style>
