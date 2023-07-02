<template>
	<div v-if="isAuth(['ROOT', 'DEPT:SELECT'])">
		<el-form :inline="true" :model="dataForm" :rules="dataRule" ref="dataForm">
			<el-form-item prop="deptName">
				<el-input
					v-model="dataForm.deptName"
					placeholder="部门名称"
					size="medium"
					class="input"
					clearable="clearable"
				/>
			</el-form-item>
			<el-form-item>
				<el-button size="medium" type="primary" @click="searchHandle()">查询</el-button>
				<el-button
					size="medium"
					type="primary"
					:disabled="!isAuth(['ROOT', 'DEPT:INSERT'])"
					@click="addHandle()"
				>
					新增
				</el-button>
				<el-button
					size="medium"
					type="danger"
					:disabled="!isAuth(['ROOT', 'DEPT:DELETE'])"
					@click="deleteHandle()"
				>
					批量删除
				</el-button>
			</el-form-item>
		</el-form>
		<el-table
			:data="dataList"
			border
			v-loading="dataListLoading"
			@selection-change="selectionChangeHandle"
			cell-style="padding: 4px 0"
			size="medium"
			style="width: 100%;"
		>
			<el-table-column type="selection" :selectable="selectable" header-align="center" align="center" width="50"/>
			<el-table-column type="index" header-align="center" align="center" width="100" label="序号">
				<template #default="scope">
					<span>{{ (pageIndex - 1) * pageSize + scope.$index + 1 }}</span>
				</template>
			</el-table-column>
			<el-table-column prop="deptName" header-align="center" align="center" label="部门名称" min-width="170"/>
			<el-table-column prop="tel" header-align="center" align="center" label="联系电话" min-width="170"/>
			<el-table-column prop="email" header-align="center" align="center" label="邮箱" min-width="270"/>
			<el-table-column header-align="center" align="center" label="员工数量" min-width="140" >
				<template #default="scope">
					<span>{{ scope.row.emps }}人</span>
				</template>
			</el-table-column>
			<el-table-column prop="desc" header-align="center" align="center" label="备注" min-width="400"/>
			<el-table-column header-align="center" align="center" width="150" label="操作">
				<template #default="scope">
					<el-button
						type="text"
						size="medium"
						:disabled="!isAuth(['ROOT', 'DEPT:UPDATE'])"
						@click="updateHandle(scope.row.id)"
					>
						修改
					</el-button>
					<el-button
						type="text"
						size="medium"
						:disabled="!isAuth(['ROOT', 'DEPT:DELETE']) || scope.row.emps > 0"
						@click="deleteHandle(scope.row.id)"
					>
						删除
					</el-button>
				</template>
			</el-table-column>
		</el-table>
		<!-- 分页索引 -->
		<el-pagination
			@size-change="sizeChangeHandle"
			@current-change="currentChangeHandle"
			:current-page="pageIndex"
			:page-sizes="[10, 20, 50]"
			:page-size="pageSize"
			:total="totalCount"
			layout="total, sizes, prev, pager, next, jumper"
		></el-pagination>
		<!-- 添加表单，用addOrUpdateVisible 让其显示出来 -->
		<add-or-update v-if="addOrUpdateVisible" ref="addOrUpdate" @refreshDataList="loadDataList"></add-or-update>
	</div>
</template>

<script>
import AddOrUpdate from './dept-add-or-update.vue';
export default {
	components: {
		AddOrUpdate
	},
	data: function() {
		return {
			dataForm: {
				deptName: null
			},
			dataList: [],
			pageIndex: 1,
			pageSize: 10,
			totalCount: 0,
			dataListLoading: false,
			dataListSelections: [],
			addOrUpdateVisible: false,
			dataRule: {
				deptName: [
					{ required: false, pattern: '^[a-zA-Z0-9\u4e00-\u9fa5]{1,10}$', message: '部门名称格式错误' }
				]
			}
		};
	},
	methods: {
		
		/**
		 * 勾选小框中的部门
		 * @param {Object} row
		 * @param {Object} index
		 */
		selectable:function(row,index){
			if(row.emps>0){
				return false
			}
			return true
		},
		/**
		 * 绑定@selection-change，勾选时将会使该行所有的值通过val存储到dataListSeletions中
		 * @param {Object} val
		 */
		selectionChangeHandle: function(val) {
			this.dataListSelections = val;
		},
		
		
		/**
		 * 将从后端通过Ajax获取数据值,并渲染到table表单中
		 * 
		 */
		loadDataList: function(){
			
			let that = this;
			that.dataListLoading = true; 	//数据未出现前一个小圆圈正在加载中
			
			//发送Ajax的data数据
			let data = {
				
				deptName: that.dataForm.deptName,
				page: that.pageIndex,		//当前页
				length: that.pageSize		//展示行数
				
			};
			/**
			 * 发送Ajax
			 * resp{
			 *     page{totalCount(count), pageSize(length), totalPage, pageIndex(start), list(部门信息)}
			 * 	}
			 */
			that.$http("dept/searchDeptByPage", "POST", data, true, function(resp){
				
				//后端返回page对象，需要先将page提取出来先
				let page = resp.page;
				//将list表中的数据渲染到table中
				that.dataList = page.list;
				//将总页数渲染到分页索引
				that.totalCount = page.totalCount;
				//关掉加载小圆圈
				that.dataListLoading = false;
				
			});
			
		},
		
		/**
		 * 每页显示几条记录
		 * @param {Object} val 为条数，即上述自定义的[10,20,50]
		 */
		sizeChangeHandle: function(val){
			
			//为查询展示的页面条数10，20，50
			this.pageSize = val;
			//变换展示条数后页面返回第一页
			this.pageIndex = 1;
			//重新加载数据
			this.loadDataList();
			
		},
		
		/**
		 * 用户点击当前页
		 * @param {Object} val 表示当前的页数
		 */
		currentChangeHandle: function(val){
			
			//将用户点击的页面传值到索引中
			this.pageIndex = val;
			//重新渲染页面 
			this.loadDataList();
						
			
		},
		
		/**
		 * 条件查询表单，绑定了查询按钮“查询”
		 */
		searchHandle: function(){
			
			//校验输入的表单
			this.$refs["dataForm"].validate(valid => {
				
				//校验成功后，判断是否输入数据
				if(valid){
					
					this.$refs["dataForm"].clearValidate();
					
					if(this.dataForm.deptName == ''){
						
						this.dataForm.deptName = null;
						
					}
					if(this.pageIndex != 1){
						
						this.pageIndex = 1;
						
					}
					//重新渲染表格
					this.loadDataList();
					
				}else{
					
					return false;
					
				}
				
				
			});
			
			
		},
		
		//添加部门函数
		addHandle: function(){
			//当点击新增按钮时，则显示添加和修改表单
			this.addOrUpdateVisible = true;
			//调用dept-add-or-update中的初始化函数init()，初始化该表单
			this.$nextTick(() => {
				this.$refs.addOrUpdate.init();
			});
			
			
		},
		
		
		/**
		 * 修改部门信息，通过传入的id值来判断是update，进而渲染整个数据
		 * @param {Object} ids
		 */
		updateHandle: function(id){
			
			//当点击新增按钮时，则显示添加和修改表单
			this.addOrUpdateVisible = true;
			//调用dept-add-or-update中的初始化函数init()，初始化该表单
			this.$nextTick(() => {
				this.$refs.addOrUpdate.init(id);
			});
			
		},
		
		/**
		 * 删除部门，通过传入ids来删除
		 */
		deleteHandle: function(id){
			
			let that = this;
			//通过三元运算符来判断是单个删除还是批量删除,当id为一个时，则判断为true，将其存储为数组[id];
			let ids = id ? [id] : that.dataListSelections.map(item => {
				
				//因为dataListSelections中存储了所有的属性，而我们只需要提取他的id属性就行了
				return item.id;
			});
			
			//判断是否选中id删除
			if(ids.length == 0){	//没有选中时
				
				that.$message({
					
					message: "没有选中记录",
					type: "warning",
					duration: 1200
					
				});
				
			}else{	//选中id时
				
				that.$confirm(`确定要删除选中的记录？`, "提示", {
					
					confirmButtonText: "确定",
					cancelButtonText: "取消",
					type: "warning"
					
				}).then(() => {
					
					//发送Ajax获取数据,返回的是rows
					that.$http("dept/deleteDeptByIds", "POST", {ids: ids}, true, function(resp){
						
						//rows不能写成rows==1,因为当批量删除时，返回的不止一条rows，故要判断rows>0
						if(resp.rows > 0){
							
							that.$message({
								
								message: "操作成功",
								type: "success",
								duration: 1200
								
							});	
							//删除成功后重新渲染页面
							that.loadDataList();
							
						}else{
							
							that.$message({
								
								message: "操作失败",
								type: "error",
								duration: 1200
								
							});
							
						}
						
						
					});
					
					
				})
				
				
				
			}
			
			
			
		}
		
		
		
		
	},
	
	created: function(){
		
		this.loadDataList();
	}
	
};
</script>

<style></style>
