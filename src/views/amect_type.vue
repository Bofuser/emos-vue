<template>
	<div v-if="isAuth(['ROOT'])">
		<el-form :inline="true" :model="dataForm" :rules="dataRule" ref="dataForm">
			<!-- 查询栏 -->
			<el-form-item prop="type">
				<el-input
					v-model="dataForm.type"
					placeholder="类型名称"
					size="medium"
					class="input"
					clearable="clearable"
				/>
			</el-form-item>
			<!-- 查询条件 -->
			<el-form-item>
				<el-button size="medium" type="primary" @click="searchHandle()">查询</el-button>
				<el-button size="medium" type="primary" @click="addHandle()">新增</el-button>
				<el-button size="medium" type="danger" @click="deleteHandle()">批量删除</el-button>
			</el-form-item>
		</el-form>
		<el-table
			:data="dataList"
			border
			v-loading="dataListLoading"
			@selection-change="selectionChangeHandle"
			cell-style="padding: 4px 0"
			style="width: 100%;"
			size="medium"
		>
			<!-- 表示复选框，可以勾选  -->
			<el-table-column
				type="selection"
				:selectable="selectable"
				header-align="center"
				align="center"
				width="50"
			/>
			<el-table-column type="index" header-align="center" align="center" width="100" label="序号">
				<template #default="scope">
					<span>{{ (pageIndex - 1) * pageSize + scope.$index + 1 }}</span>
				</template>
			</el-table-column>
			<el-table-column prop="type" header-align="center" align="center" label="罚款类型" />
			<el-table-column header-align="center" align="center" label="罚款金额">
				<template #default="scope">
					<span>{{ scope.row.money }}元</span>
				</template>
			</el-table-column>
			<el-table-column header-align="center" align="center" label="系统自带">
				<template #default="scope">
					<span>{{ scope.row.systemic ? '是' : '否' }}</span>
				</template>
			</el-table-column>
			<el-table-column header-align="center" align="center" label="未缴罚款数量">
				<template #default="scope">
					<span>{{ scope.row.notPay == 0 ? '--' : scope.row.notPay + '个' }}</span>
				</template>
			</el-table-column>
			<el-table-column fixed="right" header-align="center" align="center" width="150" label="操作">
				<template #default="scope">
					<el-button type="text" size="medium" @click="updateHandle(scope.row.id)">修改</el-button>
					<el-button
						type="text"
						size="medium"
						:disabled="scope.row.canDelete == 'false'"
						@click="deleteHandle(scope.row.id)"
					>
						删除
					</el-button>
				</template>
			</el-table-column>
		</el-table>
		<el-pagination
			@size-change="sizeChangeHandle"
			@current-change="currentChangeHandle"
			:current-page="pageIndex"
			:page-sizes="[10, 20, 50]"
			:page-size="pageSize"
			:total="totalCount"
			layout="total, sizes, prev, pager, next, jumper"
		></el-pagination>
		<add-or-update v-if="addOrUpdateVisible" ref="addOrUpdate" @refreshDataList="loadDataList"></add-or-update>
	</div>
</template>

<script>
import AddOrUpdate from './amect_type-add-or-update.vue';
export default {
	components: {
		AddOrUpdate
	},
	data: function() {
		return {
			dataForm: {
				type: null
			},
			dataList: [],
			pageIndex: 1,
			pageSize: 10,
			totalCount: 0,
			dataListLoading: false,
			dataListSelections: [],
			addOrUpdateVisible: false,
			dataRule: {
				type: [{ required: false, pattern: '^[a-zA-Z0-9\u4e00-\u9fa5]{1,10}$', message: '类型名称格式错误' }]
			}
		};
	},
	methods: {
		/**
		 * 加载页面表单函数
		 */
		loadDataList: function() {
			let that = this;
			that.dataListLoading = true;
			let data = {
				type: that.dataForm.type,
				page: that.pageIndex,
				length: that.pageSize
			};
			// 发送Ajax
			that.$http('amect_type/searchAmectTypeByPage', 'POST', data, true, function(resp){
				let page = resp.page;

				//将后端返回的 page 值传送到模型层中中
				that.dataList = page.list;
				that.totalCount = page.totalCount;
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
		//条件查询函数
		searchHandle: function() {
			this.$refs['dataForm'].validate(valid => {
				if(valid){
					this.$refs['dataForm'].clearValidate();
					//当查询条件 type没写时，赋值为null
					if(this.dataForm.type == ''){
						this.dataForm.type = null;
					}
					//将页面返回到首页
					if(this.pageIndex != 1){
						this.pageIndex = 1;
					}
					//重新刷新页面 
					this.loadDataList();
				}else{
					return false;
				}
			});
		},
		/**
		 * 添加新的罚款类型
		 */
		addHandle: function(){
			//展示添加小窗口
			this.addOrUpdateVisible = true;
			//初始化该表单
			this.$nextTick(() => {
				this.$refs.addOrUpdate.init();
			})
		},

		/**
		 * 修改罚款类型表单
		 * @param {} id 
		 */
		updateHandle: function(id) {
			//展示添加小窗口
			this.addOrUpdateVisible = true;
			//初始化该表单
			this.$nextTick(() => {
				this.$refs.addOrUpdate.init(id);
			})

		},

		//selectable()函数，用于是否可以勾选
		selectable: function(row,index){
			if (row.canDelete == 'true'){
				return true;
			}
			return false;
		 },
		 //selectionChangHandle函数，用于保存选中的条数
		 selectionChangeHandle: function(val){
			this.dataListSelections = val;
		 },

		/**
		 * 删除函数
		 */
		deleteHandle: function(id){

			let that = this;

			//通过三元字符判断 id （单个）是否存在，如果存在则指包含一个元素id，否则它将包含dataListSelection中数组的元素，即勾选的id数
			let ids = id ? [id] : that.dataListSelections.map(item => {

				return item.id;
			});

			// 判断id是否存在，存在则删除
			if(ids.length == 0){
				//不存在则提示没有记录
				that.$message({
					message: '没有选中记录',
					type: 'warning',
					duration: 1200
				});
			}else{

				//发送Ajax获取请求删除
				that.$confirm(`确定要删除选中的记录？`, '提示', {

					confirmButtonText: '确定',
					cancelButtonText: '取消',
					type: 'warning'
				}).then(() => {
					//点击确定后，执行该函数,发送Ajax
					that.$http('amect_type/delete', 'POST', {ids: ids}, true, function(resp){

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

	},
	created: function() {
		this.loadDataList();
	}
};
</script>

<style lang="less" scoped="scoped"></style>
