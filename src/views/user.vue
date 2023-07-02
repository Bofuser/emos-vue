<template>
	<div v-if="isAuth(['ROOT', 'USER:SELECT'])">	<!-- 只允许权限为ROOT和SELECT的人查看次表单 -->
		<!-- form表单，用于条件查询时提交数据 -->
		<el-form :inline="true" :model="dataForm" :rules="dataRule" ref="dataForm">
			<el-form-item prop="name">
				<el-input
					v-model="dataForm.name"
					placeholder="姓名"
					size="medium"
					class="input"
					clearable="clearable"
				/>
			</el-form-item>
			<!-- 选择性别框 -->
			<el-form-item>
				<el-select v-model="dataForm.sex" class="input" placeholder="性别" size="medium" clearable="clearable">
					<el-option label="男" value="男" />
					<el-option label="女" value="女" />
				</el-select>
			</el-form-item>
			<!-- 角色数据通过调用后端数据库中的名字渲染到上面 -->
			<el-form-item>
				<el-select v-model="dataForm.role" class="input" placeholder="角色" size="medium" clearable="clearable">
					<el-option v-for="one in roleList" :key="one.roleId" :label="one.roleName" :value="one.roleName" />
				</el-select>
			</el-form-item>
			<!-- 部门数据通过调用后端数据库中的名字渲染到上面 -->
			<el-form-item>
				<el-select
					v-model="dataForm.deptId"
					class="input"
					placeholder="部门"
					size="medium"
					clearable="clearable"
				>
					<el-option v-for="one in deptList" :key="one.id" :label="one.deptName" :value="one.id" />
				</el-select>
			</el-form-item>
			<!--  -->
			<el-form-item>
				<el-select
					v-model="dataForm.status"
					class="input"
					placeholder="状态"
					size="medium"
					clearable="clearable"
				>
					<el-option label="在职" value="1" />
					<el-option label="离职" value="2" />
				</el-select>
			</el-form-item>
			<el-form-item>
				<el-button size="medium" type="primary" @click="searchHandle()">查询</el-button>
				<el-button
					size="medium"
					type="primary"
					:disabled="!isAuth(['ROOT', 'USER:INSERT'])"
					@click="addHandle()"
				>
					新增
				</el-button>
				<el-button
					size="medium"
					type="danger"
					:disabled="!isAuth(['ROOT', 'USER:DELETE'])"
					@click="deleteHandle()"
				>
					批量删除
				</el-button>
			</el-form-item>
		</el-form>
		<!-- dataList ， 通过loadDataList函数将数据渲染到这张表格中-->
		<!-- :data=dataList 为 v-bind:data=dataList 的缩写，表示表格的数据和dataList绑定-->
		<!-- @selection-change表示获取复选框的集合 -->
		<el-table
			:data="dataList"
			border
			v-loading="dataListLoading"
			@selection-change="selectionChangeHandle"
			cell-style="padding: 4px 0"
			style="width: 100%;"
			size="medium"
		>
			<el-table-column type="selection" header-align="center" align="center" width="50"/>
			<el-table-column type="index" header-align="center" align="center" width="100" label="序号">
				<template #default="scope">
					<span>{{ (pageIndex - 1) * pageSize + scope.$index + 1 }}</span>
				</template>
			</el-table-column>
			<el-table-column prop="name" header-align="center" align="center" min-width="100" label="姓名" />
			<el-table-column prop="sex" header-align="center" align="center" min-width="60" label="性别" />
			<el-table-column prop="tel" header-align="center" align="center" min-width="130" label="电话" />
			<el-table-column
				prop="email"
				header-align="center"
				align="center"
				min-width="240"
				label="邮箱"
				:show-overflow-tooltip="true"
			/>
			<el-table-column prop="hiredate" header-align="center" align="center" min-width="130" label="入职日期" />
			<el-table-column
				prop="roles"
				header-align="center"
				align="center"
				min-width="150"
				label="角色"
				:show-overflow-tooltip="true"
			/>
			<el-table-column prop="dept" header-align="center" align="center" min-width="120" label="部门" />
			<el-table-column prop="status" header-align="center" align="center" min-width="100" label="状态" />
			<el-table-column header-align="center" align="center" width="150" label="操作">
				<!-- scope表示当前这条记录，scope.row表示当前这条记录的ID -->
				<template #default="scope">
					<el-button
						type="text"
						size="medium"
						v-if="isAuth(['ROOT', 'USER:UPDATE'])"
						@click="updateHandle(scope.row.id)"
					>
						修改
					</el-button>
					<el-button
						type="text"
						size="medium"
						v-if="isAuth(['ROOT', 'USER:UPDATE'])"
						:disabled="scope.row.status == '离职' || scope.row.root"
						@click="dimissHandle(scope.row.id)"
					>
						离职
					</el-button>
					<!-- 删除一个则调通过scope.row.id调用这个id -->
					<el-button
						type="text"
						size="medium"
						:disabled="scope.row.root"
						v-if="isAuth(['ROOT', 'USER:DELETE'])"
						@click="deleteHandle(scope.row.id)"
					>
						删除
					</el-button>
				</template>
			</el-table-column>
		</el-table>
		<!-- 分页条 -->
		<!-- 其中 size-change事件是用户改变每页记录数量触发的；current-change事件是用户翻页触发的-->
		<el-pagination
			@size-change="sizeChangeHandle"			
			@current-change="currentChangeHandle"
			:current-page="pageIndex"
			:page-sizes="[10, 20, 50]"
			:page-size="pageSize"
			:total="totalCount"
			layout="total, sizes, prev, pager, next, jumper"
		></el-pagination>
		<!-- 新增或修改用户 引用这个页面作为修改表单-->
		<add-or-update v-if="addOrUpdateVisible" ref="addOrUpdate" @refreshDataList="loadDataList"></add-or-update>
		<dimiss v-if="dimissVisible" ref="dimiss" @refreshDataList="loadDataList"></dimiss>
	</div>
</template>

<script>
import AddOrUpdate from './user-add-or-update.vue';		//引用了这个user-add-or-update.vue这个弹窗页面
import Dimiss from './dimiss.vue';
export default {
	//通过上面import导入的两个组件，components属性是Vue组件中的一个选项，用于注册子组件
	components: {
		AddOrUpdate,
		Dimiss
	},
	data() {
		return {
			//这个表单用于封装添加数据的内容
			dataForm: {
				name: '',
				sex: '',
				role: '',
				deptId: '',
				status: ''
			},
			dataList: [],	//Ajax发送请求获得的用户数据会保存在这个数组中,然后由视图层进行调用数据遍历出数据
			roleList: [],	//Ajax发送请求的角色列表数据会保存在此数组中,然后由视图层进行调用数据遍历出数据
			deptList: [],	//Ajax发送请求的部门列表数据会保存在此数组中,然后由视图层进行调用数据遍历出数据
			pageIndex: 1,
			pageSize: 10,
			totalCount: 0,
			dataListLoading: false,
			dataListSelections: [],
			addOrUpdateVisible: false,
			dimissVisible: false,
			dataRule: {		// 用来验证输入的姓名是否符合规定，并提供了错误提示信息
				name: [{ required: false, pattern: '^[\u4e00-\u9fa5]{1,10}$', message: '姓名格式错误' }]
			}
		};
	},
	methods: {
		
		/**
		 * 新增函数调用
		 */
		addHandle: function() {
			//给add-or-update中的v-if条件赋值true，使该标签可用
		    this.addOrUpdateVisible = true;
			//调用add-or-update页面的初始化函数，使其初始化
		    this.$nextTick(() => {
		        this.$refs.addOrUpdate.init();
		    });
		},
		/**
		 * 修改函数调用
		 * @param {Object} id
		 */
		updateHandle: function(id) {
		    this.addOrUpdateVisible = true;
		    this.$nextTick(() => {
		        this.$refs.addOrUpdate.init(id);
		    });
		},
		/**
		 * 这个loadDataList函数很重要，通过这个函数向后端发送ajax数据，
		 * 数据包括 page、length、name、sex、role、deptId、status这几个数据
		 * 
		 * 
		 * 渲染后端传过来的用户数据
		 */
		loadDataList: function() {
			
			let that = this;
			
			//请求后端数据时，让表格出现循环等待的图标,这个功能是通过赋值v-loading为true实现的
			//这些数据发送到后端中，条件查询条件中没有赋值的则传null过去。
			that.dataListLoading = true;
			let data = {
				
				page: that.pageIndex,
				length: that.pageSize,
				name: that.dataForm.name,
				sex: that.dataForm.sex,
				role: that.dataForm.role,
				deptId: that.dataForm.deptId,
				status: that.dataForm.status
				
			};
			
			//发送ajax请求
			that.$http('user/searchUserByPage', 'POST', data, true, function(resp){
				
				/**
				 * 后端返回的是打包成json格式的page参数，其中page中包含有：
				 * totalCount, pageSize, totalPage, pageIndex, 和list（用户信息）。
				 * 
				 * list中的参数为：sex,root,roles,name,tel,id,dept,hiredate,email,status
				 * 
				 * @param {Object} let one of list
				 */
				
				let page = resp.page;
				let list = page.list;
				for(let one of list){
					//数据库中的status为1和2表示，则在这里用if判断以下将其赋值为“在职和离职”状态
					if(one.status == 1){
						one.status = '在职';
					} else if (one.status == 2){
						one.status = '离职';
					}
					
				}	
				//将请求的用户数据的值保存在模型层中的 dataList: []数组中，然后由视图层绑定调用该数组遍历
				that.dataList = list;
				//将值保存在模型层中的 totalCount中，然后由视图层绑定调用该值
				that.totalCount = page.totalCount;
				//把加载数据的滚动条清除掉
				that.dataListLoading = false;
								
			});
						
		},
		
		
		/**
		 * 返回所有加载角色和部分记录
		 */
		loadRoleList: function() {
			let that = this;
			that.$http('role/searchAllRole', 'GET', null, true, function(resp) {
				//将角色值保存在模型层中的 roleList: []数组中，然后由视图层绑定调用该数组遍历
				that.roleList = resp.list;
			});
		},
		/**
		 * 返回所有部门列表
		 */
		loadDeptList: function() {
			let that = this;
			that.$http('dept/searchAllDept', 'GET', null, true, function(resp) {
				//将部门值保存在模型层中的 deptList: []数组中，然后由视图层绑定调用该数组遍历
				that.deptList = resp.list;
			});
		},
		/**
		 * 用于勾选复选选框，由表格中的 @selection-change 属性绑定
		 * 复选框中选勾选内容不是选中记录的id，而是选中记录的完整数据，即val是所有用户信息
		 * @param {Object} val
		 */
		selectionChangeHandle: function(val) {
			
			this.dataListSelections = val;
		},
		/**
		 * 每页显示几条记录
		 * @param {Object} val 为条数，即上述自定义的[10,20,50]
		 */
		sizeChangeHandle(val) {
			//修改当前页显示条数,其中通过val传值进行修改。
		    this.pageSize = val;
		    //更改每页显示记录数量后，都从第一页开始查询
		    this.pageIndex = 1;
		    this.loadDataList();
		},
		/**
		 * 用户点击当前页
		 * @param {Object} val 表示当前的页数
		 */
		currentChangeHandle: function(val){
			
			this.pageIndex = val;
			//渲染数据
			this.loadDataList();
		},
		
		/**
		 * 条件查询表单，绑定了查询按钮“查询”
		 */
		searchHandle: function(){
			
			//先进行表单那验证
			this.$refs['dataForm'].validate( valid => {
				
				//表单校验通过时，valid为true
				if(valid){					
					//表单中不能传入空字符，但可以传入null
					if(this.dataForm.name == ''){
						this.dataForm.name = null;
					}
					
					//如果页面不是第一页，则跳转回第一页
					if(this.pageIndex != 1){
						this.pageIndex == 1;
					}
					
					//通过调用loadDataList，发起ajax,将查询条件传入后端进行数据查询，查询出来的数据渲染页面
					this.loadDataList();
					
				}else {
					return false;
				}
								
			});						
		},

		/**
		 * 删除用户操作
		 * @param {*} id 
		 */
		deleteHandle: function(id){
			
			let that = this;
			/**
			 * 通过三元运算符，对传入的id进行判断，当id为一个时，则判断为true，将其存储为数组[id];
			 * 当传入id为数组或为空时，则判断为false，将传入的id通过保存到数组dataListSelections一个个提取出来封装成数组。
			 * dataListSelections为模型端的存储数组，批量勾选时会将用户数据（不止是id，是所有用户数据）保存到那里中。
			 * 批量勾选时，传入的id值即为数组，故将调用下面的函数将Id保存为数组
			 */
			let ids = id ? [id] : that.dataListSelections.map(item => {
				
				return item.id;
			});
			
			//判断传入的id是否为空，空则返回提示，否则进行删除
			if(ids.length == 0){
				
				that.$message({
					
					message: '没有选中记录',
					type: 'warning',
					duration: 1200
					
				});
				
			}else{	//请求删除成功，发送弹框是提示是否删除
			
				that.$confirm('确定要删除选中的记录？', '提示', {
					
					confirmButtonText: '确定',
					cancelButtonText: '取消',
					type: 'warning'
					
				}).then(() => {	//确定删除后执行该函数,发送Ajax
					
					that.$http('user/deleteUserByIds', 'POST', {ids: ids}, true, function(resp){
						
						//当接收到后端返回的rows，则表示删除成功
						if(resp.rows > 0){
							
							that.$message({
								
								message: '操作成功',
								type: 'success',
								duration: 1200
								
							});
							//重新加载列表数据
							that.loadDataList();
							
						}else{//否则删除失败
							
							that.$message({
								
								message: '操作失败',
								type: 'error',
								duration: 1200
								
							});
							
						}
												
					});
					
				});	
			}
		}
		
	},
	//created是生命周期钩子，表示示例创建后调用
	created: function() {
		this.loadDataList();
		this.loadRoleList();
		this.loadDeptList();
	}
};
</script>

<style lang="less" scoped="scoped"></style>
