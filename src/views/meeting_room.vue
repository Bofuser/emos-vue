<template>
	<div>
		<el-form :inline="true" :model="dataForm" :rules="dataRule" ref="dataForm">
			<el-form-item prop="name">
				<el-input
					v-model="dataForm.name"
					placeholder="会议室名称"
					size="medium"
					class="input"
					clearable="clearable"
				/>
			</el-form-item>
			<el-form-item>
				<el-select v-model="dataForm.canDelete" class="input" placeholder="条件" size="medium">
					<el-option label="全部" value="all" />
					<el-option label="可删除" value="true" />
					<el-option label="不可删除" value="false" />
				</el-select>
			</el-form-item>
			<el-form-item>
				<el-button size="medium" type="primary" @click="searchHandle()">查询</el-button>
				<el-button
					size="medium"
					type="primary"
					:disabled="!isAuth(['ROOT', 'MEETING_ROOM:INSERT'])"
					@click="addHandle()"
				>
					新增
				</el-button>
				<el-button
					size="medium"
					type="danger"
					:disabled="!isAuth(['ROOT', 'MEETING_ROOM:DELETE'])"
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
			style="width: 100%;"
			size="medium"
		>
			<el-table-column type="selection" header-align="center" align="center" width="50" />
			<el-table-column type="index" header-align="center" align="center" width="100" label="序号">
				<template #default="scope">
					<span>{{ (pageIndex - 1) * pageSize + scope.$index + 1 }}</span>
				</template>
			</el-table-column>
			<el-table-column prop="name" header-align="center" align="center" min-width="150" label="会议室名称" />
			<el-table-column header-align="center" align="center" min-width="120" label="人数上限">
				<template #default="scope">
					<span>{{ scope.row.max }}人</span>
				</template>
			</el-table-column>
			<el-table-column header-align="center" align="center" min-width="100" label="状态">
				<template #default="scope">
					<span>{{ scope.row.status == 1 ? '可使用' : '已停用' }}</span>
				</template>
			</el-table-column>
			<el-table-column prop="desc" header-align="center" align="center" label="备注" min-width="400" />
			<el-table-column header-align="center" align="center" width="150" label="操作">
				<template #default="scope">
					<el-button
						type="text"
						size="medium"
						:disabled="!isAuth(['ROOT', 'MEETING_ROOM:UPDATE']) || scope.row.id == 0"
						@click="updateHandle(scope.row.id)"
					>
						修改
					</el-button>
					<el-button
						type="text"
						size="medium"
						:disabled="!isAuth(['ROOT', 'MEETING_ROOM:DELETE']) || scope.row.id == 0"
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
import AddOrUpdate from './meeting_room-add-or-update.vue';
export default {
	components: {
		AddOrUpdate
	},
	data: function() {
		return {
			dataForm: {
				name: null,
				canDelete: null
			},
			dataList: [],
			pageIndex: 1,
			pageSize: 10,
			totalCount: 0,
			dataListLoading: false,
			dataListSelections: [],
			addOrUpdateVisible: false,
			dataRule: {
				name: [{ required: false, pattern: '^[a-zA-Z0-9\u4e00-\u9fa5]{1,20}$', message: '会议室名称格式错误' }]
			}
		};
	},
	methods: {
		
		/**
		 * 绑定@selection-change，勾选时将会使该行所有的值通过val存储到dataListSeletions中
		 * @param {Object} val
		 */
		selectionChangeHandle: function(val) {
			this.dataListSelections = val;
		},
		/**
		 * 每页显示几条记录
		 * @param {Object} val 为条数，即上述自定义的[10,20,50]
		 */
		sizeChangeHandle: function(val) {
			this.pageSize = val;
			this.pageIndex = 1;
			this.loadDataList();
		},
		/**
		 * 用户点击当前页
		 * @param {Object} val 表示当前的页数
		 */
		currentChangeHandle: function(val) {
			this.pageIndex = val;
			this.loadDataList();
		},
		
		
		/**
		 * 渲染数据到表格中
		 */
		loadDataList: function(){
			
			let that = this;
			//加载数据前的小圆圈
			that.dataListLoading = true;
			//定义需要发送的前端数据
			let data = {
				
				name: that.dataForm.name,
				canDelete: that.dataForm.canDelete == 'all' ? null : that.dataForm.canDelete,
				page: that.pageIndex,		//当前页
				length: that.pageSize		//展示行数
				
			};
			
			//发送Ajax，请求和响应数据
			that.$http("meeting_room/searchMeetingRooByPage", "POST", data, true, function(resp){	
				//resp{page{list, totalCount, pageSize, pageIndex, totalPage}}
				
				//将resp的对象取出来封装到page中
				let page = resp.page;
				//将list数据取出来保存到dataList中
				that.dataList = page.list;
				//总条数
				that.totalCount = page.totalCount;		
				//页面渲染完毕后，关闭小圆圈
				that.dataListLoading = false;				
			});
			
			
		},
		
		/**
		 * 查询会议条件
		 * 条件：name和canDelete
		 */
		searchHandle: function(){
			
			//校验输入表单
			this.$refs["dataForm"].validate(valid => {
				
				//校验成功后，判断是否输入数据，没有则为其赋值null
				if(valid){
					
					//移除表单校验
					this.$refs["dataForm"].clearValidate();
					
					if(this.dataForm.name == ''){
						this.dataForm.name = null;
					}
					
					if( this.dataForm.canDelete == ''){
						this.dataForm.canDelete = null;
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
		
		
		/**
		 * 添加会议信息
		 */
		addHandle: function(){
			
			//将自定义的组件可视化
			this.addOrUpdateVisible = true;
			//调用meeting_room-add-or-update
			this.$nextTick(() =>{
				
				this.$refs.addOrUpdate.init();
				
			});
			
			
		},
		
		/**
		 * 修改表单
		 * @param {Object} id
		 */
		updateHandle: function(id){
			
			//将自定义的组件可视化
			this.addOrUpdateVisible = true;
			//调用meeting_room-add-or-update
			this.$nextTick(() =>{
				
				this.$refs.addOrUpdate.init(id);
				
			});
			
		},
		
		
		/**
		 * 删除会议室
		 * @param {Object} id
		 */
		deleteHandle: function(id){
			
			let that = this;
			//通过三元运算符来判断是单个删除还是批量删除,当id为一个时，则判断为true，将其存储为数组[id];
			let ids = id ? [id] : that.dataListSelections.map(item => {
				
				return item.id;
				
			});
			
			//判断是否选中
			if(ids.length == 0){
				
				that.$message({
					
					message: "没有选中记录",
					type: "warning",
					duration: 1200
					
				});
				
			}else{
				
				//调用comfirm表单确认是否删除
				that.$confirm(`确定要删除选中的记录？`, "提示", {
					
					confirmButtonText: "确定",
					cancelButtonText: "取消",
					type: "warning"
					
				}).then(() => {
					
					//返送Ajax
					that.$http("meeting_room/deleteMeetingRoomByIds","POST", {ids: ids}, true, function(resp){
						
						if(resp.rows > 0){
							
							that.$message({
								
								message: "操作成功",
								type: "success",
								duration: 1200
								
							});
							
							//重新渲染表格
							that.loadDataList();
							
						}else{
							
							that.$message({
								
								message: "操作失败",
								type: "error",
								duration: 1200
								
							});
							
						}
						
					});
					
					
					
				});
				
				
			}
			
			
			
			
			
			
		}
		
		
	},
	
	created: function(){
		
		this.loadDataList();
		
	}

};
</script>

<style lang="less"></style>
