<template>
	<el-dialog
		:title="!dataForm.id ? '新增' : '修改'"
		v-if="isAuth(['ROOT', 'USER:INSERT', 'USER:UPDATE'])"
		:close-on-click-modal="false"
		v-model="visible"
		width="450px"
	>
		<el-form :model="dataForm" ref="dataForm" :rules="dataRule" label-width="80px">
			<el-form-item label="用户名" prop="username">
				<el-input v-model="dataForm.username" size="medium" clearable />
			</el-form-item>
			<el-form-item label="密码" prop="password">
				<el-input type="password" v-model="dataForm.password" size="medium" clearable />
			</el-form-item>
			<el-form-item label="姓名" prop="name">
				<el-input v-model="dataForm.name" size="medium" clearable />
			</el-form-item>
			<el-form-item label="性别" prop="sex">
				<el-select v-model="dataForm.sex" size="medium" style="width: 100%;" clearable>
					<el-option label="男" value="男"></el-option>
					<el-option label="女" value="女"></el-option>
				</el-select>
			</el-form-item>
			<el-form-item label="电话" prop="tel">
				<el-input v-model="dataForm.tel" size="medium" clearable />
			</el-form-item>
			<el-form-item label="邮箱" prop="email">
				<el-input v-model="dataForm.email" size="medium" clearable />
			</el-form-item>
			<el-form-item label="入职日期" prop="hiredate">
				<el-date-picker
					v-model="dataForm.hiredate"
					type="date"
					placeholder="选择日期"
					size="medium"
					:editable="false"
					format="YYYY-MM-DD"
					value-format="YYYY-MM-DD"
					style="width: 100%;"
				/>
			</el-form-item>
			<el-form-item label="角色" prop="role">
				<el-select
					v-model="dataForm.role"
					size="medium"
					placeholder="选择角色"
					style="width: 100%;"
					multiple
					clearable
				>
					<el-option
						v-for="one in roleList"
						:key="one.id"
						:label="one.roleName"
						:value="one.id"
						:disabled="one.roleName == '超级管理员'"
					></el-option>
				</el-select>
			</el-form-item>
			<el-form-item label="部门" prop="deptId">
				<el-select
					v-model="dataForm.deptId"
					size="medium"
					placeholder="选择部门"
					style="width: 100%;"
					clearable
				>
					<el-option v-for="one in deptList" :key="one.id" :label="one.deptName" :value="one.id" />
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
import dayjs from 'dayjs';
export default {
	data: function() {
		return {
			visible: false,		//该页面用表单表示，并不是单独的页面，故当visible为false时不显示在用户管理页面中
			dataForm: {
				id: null,		//id是用于修改信息用的，新增信息用不到
				username: null,
				password: null,
				name: null,
				sex: null,
				tel: null,
				email: null,
				hiredate: new Date(),
				role: null,
				deptId: null,
				status: 1
			},
			roleList: [],	//Ajax发送请求的角色列表数据会保存在此数组中,然后由视图层进行调用数据遍历出数据
			deptList: [],	//Ajax发送请求的部门列表数据会保存在此数组中,然后由视图层进行调用数据遍历出数据
			dataRule: {		//弹窗中输入的验证规则, 通过prop属性绑定验证要求，待会测试一下。测试正确，绑定prop属性绑定该验证规则
				username: [{ required: true, pattern: '^[a-zA-Z0-9]{5,20}$', message: '用户名格式错误' }],
				password: [{ required: true, pattern: '^[a-zA-Z0-9]{6,20}$', message: '密码格式错误' }],
				name: [{ required: true, pattern: '^[\u4e00-\u9fa5]{2,10}$', message: '姓名格式错误' }],
				sex: [{ required: true, message: '性别不能为空' }],
				tel: [{ required: true, pattern: '^1\\d{10}$', message: '电话格式错误' }],
				email: [
					{
						required: true,
						pattern: '^([a-zA-Z]|[0-9])(\\w|\\-)+@[a-zA-Z0-9]+\\.([a-zA-Z]{2,4})$',
						message: '邮箱格式错误'
					}
				],
				hiredate: [{ required: true, trigger: 'blur', message: '入职日期不能为空' }],
				role: [{ required: true, message: '角色不能为空' }],
				deptId: [{ required: true, message: '部门不能为空' }],
				status: [{ required: true, message: '状态不能为空' }]
			}
		};
	},
	methods: {
		
		/**
		 * 初始化函数，和修改密码功能调用一样，但点击新增或修改时会调用该函数，具体调用在用户管理页面中实现
		 * @param {Object} id
		 */
		init: function(id) {
			let that = this;
			that.dataForm.id = id || 0;
			//visible1=true时，新增或修改表单可见
			that.visible = true;
			//加载列表数据的Ajax请求放在下次DOM更新来执行
			that.$nextTick(() => {
				//加载弹窗时将弹窗页面的填框中复位一下,清空信息
				that.$refs['dataForm'].resetFields();
				//新增填写信息中加载角色列表数据
				that.$http('role/searchAllRole', 'GET', null, true, function(resp) {
					that.roleList = resp.list;
				});
				//加载部门列表数据
				that.$http('dept/searchAllDept', 'GET', null, true, function(resp) {
					that.deptList = resp.list;
				});
				/**
				 * 这段代码用于修改用户，判断id是否为0，0则表示为新增用户，其他数则表示修改用户。
				 * 当id为0时，that.dataForm.id=null=false，当id为其他数字时，that.dataForm.id=true，执行下列判断条件
				 * 修改用户时需要用Ajax获取被修改的用户信息，将信息填写在表框中。
				 */			
				if (that.dataForm.id) {
					that.$http('user/searchById', 'POST', { userId: id }, true, function(resp) {
						that.dataForm.username = resp.username;
						that.dataForm.password = resp.password;
						that.dataForm.name = resp.name;
						that.dataForm.sex = resp.sex;
						that.dataForm.tel = resp.tel;
						that.dataForm.email = resp.email;
						that.dataForm.hiredate = resp.hiredate;
						that.dataForm.role = JSON.parse(resp.role);
						that.dataForm.deptId = resp.deptId;
						that.dataForm.status = resp.status;
					});
				}
			});
		},
		
		/**
		 * 提交表单时调用该函数发送数据给后端
		 */
		dataFormSubmit: function(){
			let that = this;
			//校验表单
			that.$refs['dataForm'].validate(function(valid){
				
				//当表单校验通过时，valid为true
				if(valid){
					
					//将表单中的信息存储到data中
					let data = {
						userId: that.dataForm.id,
						username: that.dataForm.username,
						password: that.dataForm.password,
						name: that.dataForm.name,
						sex: that.dataForm.sex,
						tel: that.dataForm.tel,
						email: that.dataForm.email,
						hiredate: dayjs(that.dataForm.hiredate).format('YYYY-MM-DD'),
						role: that.dataForm.role,
						deptId: that.dataForm.deptId,
						status: that.dataForm.status
												
						
					};
					/**
					 * 发送Ajax
					 * 发送路径表示当id取反表示为true时，则用insert路径，否则用update路径
					 */
					
					 that.$http(`user/${!that.dataForm.id ? 'insert' : 'update'}`, 'POST', data, true, resp => {
						if (resp.rows == 1) {
							that.$message({
								message: '操作成功',
								type: 'success',
								duration: 1200
							});
							//关闭弹窗显示
							that.visible = false;
							//重新刷新列表
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
</style>
