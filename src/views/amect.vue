<template>
	<div>
		<el-form :inline="true" :model="dataForm" :rules="dataRule" ref="dataForm">
			<!-- 表单查询条件中的姓名 -->
			<el-form-item prop="name">
				<el-input
					v-model="dataForm.name"
					placeholder="姓名"
					size="medium"
					class="input"
					clearable="clearable"
				/>
			</el-form-item>
			<!-- 表单查询条件中的部门 -->
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
			<!-- 表达男查询条件中的罚款类型-->
			<el-form-item>
				<el-select
					v-model="dataForm.typeId"
					class="input"
					placeholder="罚款类型"
					size="medium"
					clearable="clearable"
				>
					<el-option v-for="one in amectTypeList" :key="one.id" :label="one.type" :value="one.id" />
				</el-select>
			</el-form-item>
			<!-- 条件查询中的日期 -->
			<el-form-item>
				<el-date-picker
					v-model="dataForm.date"
					type="daterange"
					range-separator="~"
					start-placeholder="开始日期"
					end-placeholder="结束日期"
					size="medium"
				></el-date-picker>
			</el-form-item>
			<!-- 条件查询状态 -->
			<el-form-item>
				<el-select
					v-model="dataForm.status"
					class="input"
					placeholder="状态"
					size="medium"
					clearable="clearable"
				>
					<el-option label="未缴纳" value="1" />
					<el-option label="已缴纳" value="2" />
				</el-select>
			</el-form-item>
			<el-form-item>
				<el-button size="medium" type="primary" @click="searchHandle()">查询</el-button>
				<el-button
					size="medium"
					type="primary"
					:disabled="!isAuth(['ROOT', 'AMECT:INSERT'])"
					@click="addHandle()"
				>
					新增
				</el-button>
				<el-button
					size="medium"
					type="danger"
					:disabled="!isAuth(['ROOT', 'AMECT:DELETE'])"
					@click="deleteHandle()"
				>
					批量删除
				</el-button>
				<el-button
					size="medium"
					type="warning"
					:disabled="!isAuth(['ROOT', 'AMECT:SELECT'])"
					@click="reportHandle()"
				>
					查看报告
				</el-button>
			</el-form-item>
		</el-form>
		<!-- 这个表单用于显示从后端获取的数据，通过 :data = "dataList"属性将值进行渲染 -->
		<el-table
			:data="dataList"
			border
			v-loading="dataListLoading"
			@selection-change="selectionChangeHandle"
			cell-style="padding: 4px 0"
			style="width: 100%;"
			size="medium"
		>
			<!-- 为表单中的选项框，当selectable为true是选项框可以选，当selectable为false时不可以选择 -->
			<el-table-column
				type="selection"
				:selectable="selectable"
				header-align="center"
				align="center"
				width="50"
			/>
			<!-- 为响应数据中罚款的原因，用来展示用户的原因 -->
			<el-table-column width="40px" prop="reason" header-align="center" align="center" type="expand">
				<template #default="scope">
					罚款原因：{{ scope.row.reason }}
				</template>
			</el-table-column>
			<!-- 该表单主要用于显示代码中的行数 -->
			<el-table-column type="index" header-align="center" align="center" width="100" label="序号">
				<template #default="scope">
					<span>{{ (pageIndex - 1) * pageSize + scope.$index + 1 }}</span>
				</template>
			</el-table-column>
			<el-table-column prop="type" header-align="center" align="center" label="罚款类型" />
			<el-table-column prop="name" header-align="center" align="center" label="当事人" />
			<el-table-column prop="deptName" header-align="center" align="center" label="所属部门" />
			<el-table-column header-align="center" align="center" label="罚款金额">
				<template #default="scope">
					<span>{{ scope.row.amount }}元</span>
				</template>
			</el-table-column>
			<el-table-column prop="status" header-align="center" align="center" label="状态" />
			<el-table-column prop="createTime" header-align="center" align="center" label="日期时间" />
			<el-table-column fixed="right" header-align="center" align="center" width="150" label="操作">
				<template #default="scope">
					<el-button
						type="text"
						size="medium"
						:disabled="!(isAuth(['ROOT', 'AMECT:UPDATE']) && scope.row.status != '已缴纳')"
						@click="updateHandle(scope.row.id)"
					>
						修改
					</el-button>
					<el-button
						type="text"
						size="medium"
						:disabled="!(isAuth(['ROOT', 'AMECT:DELETE']) && scope.row.status != '已缴纳')"
						@click="deleteHandle(scope.row.id)"
					>
						删除
					</el-button>
					<el-button
						type="text"
						size="medium"
						:disabled="!(scope.row.mine == 'true' && scope.row.status == '未缴纳')"
						@click="payHandle(scope.row.id)"
					>
						交款
					</el-button>
				</template>
			</el-table-column>
		</el-table>
		 <!-- 表示查询的表单栏 -->
		<el-pagination
			@size-change="sizeChangeHandle"
			@current-change="currentChangeHandle"
			:current-page="pageIndex"
			:page-sizes="[10, 20, 50]"
			:page-size="pageSize"
			:total="totalCount"
			layout="total, sizes, prev, pager, next, jumper"
		></el-pagination>
		<!-- 引用了添加信息的表单 -->
		<add-or-update v-if="addOrUpdateVisible" ref="addOrUpdate" @refreshDataList="loadDataList"></add-or-update>
		<pay v-if="payVisible" ref="pay" @refreshDataList="loadDataList"></pay>
	</div>
</template>

<script>
import dayjs from 'dayjs';
import AddOrUpdate from './amect-add-or-update.vue';
import Pay from './amect-pay.vue';

export default {
	// 引入支付的组件
	components: { AddOrUpdate, Pay },
	data: function() {
		return {
			dataForm: {
				name: null,
				deptId: null,
				typeId: null,
				status: null,
				date: null
			},
			deptList: [],
			amectTypeList: [],
			dataList: [],
			pageIndex: 1,
			pageSize: 10,
			totalCount: 0,
			dataListLoading: false,
			dataListSelections: [],
			dataRule: {
				name: [{ required: false, pattern: '^[\u4e00-\u9fa5]{1,10}$', message: '姓名格式错误' }]
			},
			addOrUpdateVisible: false,
			payVisible: false
		};
	},
	methods: {
		//获取部门信息
		loadDeptList: function() {
			let that = this;
			that.$http('dept/searchAllDept', 'GET', null, true, function(resp) {
				that.deptList = resp.list;
			});
		},
		//获取罚款类型
		loadAmectTypeList: function() {
			let that = this;
			that.$http('amect_type/searchAllAmectType', 'GET', {}, true, function(resp) {
				that.amectTypeList = resp.list;
			});
		},
		//加载页面函数
		loadDataList: function(){

			let that = this;
			that.dataListLoading = true;
			let data = {

				name: that.dataForm.name,
				deptId: that.dataForm.deptId,
				typeId: that.dataForm.typeId,
				status: that.dataForm.status,
				page: that.pageIndex,
				length: that.pageSize

			};
			if (that.dataForm.date != null && that.dataForm.date.length == 2) {
				let startDate = that.dataForm.date[0];
				let endDate = that.dataForm.date[1];
				data.startDate = dayjs(startDate).format('YYYY-MM-DD');
				data.endDate = dayjs(endDate).format('YYYY-MM-DD');
			}
			that.$http('amect/searchAmectByPage', 'POST', data, true, function(resp){

				//后端返回一个封装了  list、totalCount、pageIndex 和 pageSize 4个数据的page
				let page = resp.page;
				for(let one of page.list){
					if(one.status == 1){
						one.status = '未缴纳';
					} else if(one.status == 2){
						one.status = '已缴纳';
					}
				}
				//赋值信息给dataList 
				that.dataList = page.list;
				//赋值信息给totalCount
				that.totalCount = page.totalCount;
				//关掉圆圈响应
				that.dataListLoading = false;

			});
		},
		// SizeChangeHandle函数,用于更改每页数据显示的条数
		sizeChangeHandle:function(val){
			this.pageSize = val;
			this.pageIndex = 1;
			this.loadDataList();
		},
		//该函数用于更改当前页面
		currentChangeHandle: function(val) {
			this.pageIndex = val;
			this.loadDataList();
		},
		/**
		 *   条件查询，绑定了页面的 查询 组件
		 */
		 searchHandle: function() {
			
			// 对前面提交的查询表单进行验证
			this.$refs['dataForm'].validate(valid => {
				//表单校验通过则valid 为true， 对下面进行传输
				if (valid) {
					//此方法清除先前在表单元素上显示的任何验证消息。
					this.$refs['dataForm'].clearValidate();
					//如果查询条件中 name 栏没有填写，则将其赋值为null
					if (this.dataForm.name == '') {
						this.dataForm.name = null;
					}
					//页面将跳转到第一页
					if (this.pageIndex != 1) {
						this.pageIndex = 1;
					}
					//重新加载页面
					this.loadDataList();
				} else {
					return false;
				}
			});
		},

		//添加新的罚款
		addHandle: function(){

			//当点击该按钮时，显示弹窗
			this.addOrUpdateVisible = true;
			//初始化该表单
			this.$nextTick(() => {
				this.$refs.addOrUpdate.init();
			})
		} ,
		//修改用户信息 
		 updateHandle: function(id) {

			//当点击该按钮时，显示弹窗
			this.addOrUpdateVisible = true;
			//初始化该表单
			this.$nextTick(() => {
				this.$refs.addOrUpdate.init(id);
			});

		 },
		 //selectable()函数，用于是否可以勾选
		 selectable: function(row,index){
			if (row.status != '已缴纳'){
				return true;
			}
			return false;
		 },
		 //selectionChangHandle函数，用于保存选中的条数
		 selectionChangeHandle: function(val){
			this.dataListSelections = val;
		 },
		 
		 /**
		  * 
		  * 删除用户操作
		  * @param {} id 
		  */
		 deleteHandle: function(id){

			let that = this;

			//通过三元字符判断 id （单个）是否存在，如果存在则指包含一个元素id，否则它将包含dataListSelection中数组的元素，即勾选的id数
			let ids = id ? [id] : that.dataListSelections.map(item => {

				return item.id;
			});

			//判断 id 是否存在
			if(ids.length == 0){
				
				//不存在则提示没有记录 
				that.$message({
					message: '没有选中记录',
					type: 'warning',
					duration: 1200
				});


			}else{  //如果有选中删除选项 

				//提示是否删除
				that.$confirm(`确定要删除选中的记录？`, '提示', {

					confirmButtonText: '确定',
					cancelButtonText: '取消',
					type: 'warning'
				}).then(() => {

					//点击确定后，执行该函数,发送Ajax
					that.$http('amect/deleteAmectByIds', 'POST', {ids: ids}, true, function(resp){

						//当接收到后端返回的rows，则表示删除成功
						if(resp.rows > 0){
							that.$message({

								message: '操作成功',
								type: 'success',
								duration: 1200

							});
							//重新加载列表数据 
							that.loadDataList();
						}else{

							that.$message({
								message: '操作失败',
								type: 'error',
								duration: 1200
							});
						}
					});
				});
			}
		 },
		 /**
		  * 支付函数，用来支付罚款
		  * @param {} id 
		  */
		 payHandle: function(id){
			//展示支付页面
			this.payVisible = true;
			//调用amect-pay组件
			this.$nextTick(() => {
				this.$refs.pay.init(id);
			});
		 },
		 reportHandle: function() {
			this.$router.push({ name: 'AmectReport' });
		 }
		 
		

	},
	created: function() {
		this.loadDeptList();
		this.loadAmectTypeList();
		this.loadDataList();

	}
};
</script>

<style></style>
