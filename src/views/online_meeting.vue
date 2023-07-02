<template>
    <div>
        <el-form :inline="true" :model="dataForm" :rules="dataRule" ref="dataForm" class="form">
            <el-form-item prop="date">
                <el-date-picker
                    v-model="dataForm.date"
                    type="date"
                    placeholder="选择日期"
                    class="input"
                    size="medium"
                ></el-date-picker>
            </el-form-item>
            <el-form-item>
                <el-button size="medium" type="primary" @click="searchHandle()">查询</el-button>
                <el-button size="medium" type="danger" @click="addHandle()">会议申请</el-button>
            </el-form-item>
            <el-form-item class="mold">
                <el-radio-group v-model="dataForm.mold" size="medium" @change="changeHandle">
                    <el-radio-button label="全部会议"></el-radio-button>
                    <el-radio-button label="我的会议"></el-radio-button>
                </el-radio-group>
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
            <el-table-column width="40px" prop="desc" header-align="center" align="center" type="expand">
                <template #default="scope">
                    会议内容：{{ scope.row.desc }}
                </template>
            </el-table-column>
			<!-- 标题 -->
            <el-table-column type="index" header-align="center" align="center" width="100" label="序号">
                <template #default="scope">
                    <span>{{ (pageIndex - 1) * pageSize + scope.$index + 1 }}</span>
                </template>
            </el-table-column>
            <el-table-column prop="title" header-align="center" align="center" label="会议主题" min-width="400" />
            <el-table-column prop="creatorName" header-align="center" align="center" min-width="150"  label="创建者" />
            <el-table-column prop="date" header-align="center" align="center" min-width="150"  label="日期" />
            <el-table-column header-align="center" align="center" min-width="150"  label="时间">
                <template #default="scope">
                    <span>{{ scope.row.start }} ~ {{ scope.row.end }}</span>
                </template>
            </el-table-column>
            <el-table-column prop="num" header-align="center" align="center" min-width="100"  label="人数" />
            <el-table-column prop="status" header-align="center" align="center" min-width="100" label="状态" />
            <el-table-column header-align="center" align="center" width="150" label="操作">
				<!-- 操作表单 当canEnterMeeting为true时，进入键不被禁止，则可以进入会议室了-->
                <template #default="scope">
                    <el-button
                        type="text"
                        size="medium"
                        :disabled="!scope.row.canEnterMeeting"
                        @click="enterHandle(scope.row.id, scope.row.uuid)"
                    >
                        进入
                    </el-button>
                    <el-button
                        type="text"
                        size="medium"
                        :disabled="
                            !(
                                (scope.row.status == '待审批' || scope.row.status == '未开始') &&
                                scope.row.isCreator == 'true'
                            )
                        "
                        @click="deleteHandle(scope.row)"
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
		<!-- 添加页面表单 -->
        <add v-if="addVisible" ref="add" @refresh="refresh"></add>
    </div>
</template>

<script>
import Add from './online_meeting-add.vue';
import dayjs from 'dayjs';

export default {
    components: { Add },
    data: function() {
        return {
            dataForm: {
                date: null,
                mold: '全部会议'
            },
            dataList: [],
            pageIndex: 1,
            pageSize: 10,
            totalCount: 0,
            dataListLoading: false,
            addVisible: false,
            dataRule: {}
        };
    },
    methods: {
		
		loadDataList : function(){
			
			let that = this;
			that.dataListLoading = true;
			
			let data = {
				
				mold: that.dataForm.mold,
				page: that.pageIndex,
				length: that.pageSize
				
			};
			
			//如果没有输入时间,则赋值今天的时间
			if (that.dataForm.date != null && that.dataForm.date != '') {
			        data.date = dayjs(that.dataForm.date).format('YYYY-MM-DD');
			}
			
			//发送ajax
			that.$http("meeting/searchOnlineMeetingByPage", "POST", data, true, function(resp){
				//resp{page{list, totalCount, pageSize, pageIndex, totalPage}}
				//后端返回page
				let page = resp.page;
				
				//循环判断List中的内容，
				for (let one of page.list){
					
					//提取参数中的状态status,将数字型状态转换成文字型状态
					 if (one.status == 1) {
						one.status = '待审批';
					} else if (one.status == 3) {
						one.status = '未开始';
					} else if (one.status == 4) {
						one.status = '进行中';
					} else if (one.status == 5) {
						one.status = '已结束';
					}
					
					/**
					 *计算会议是否可以进入
					 * 1.开会前25分钟可以进入会议室
					 * 2.会议状态必循是未开始或者进行中
					 */
					let minute = dayjs(new Date()).diff(dayjs(`${one.date} ${one.start}`), 'minute');
					if (
						one.mine == 'true' &&
						//当开会前25分钟可以进入会议,可以通过修改这里改变时间
						((minute >= -15 && minute <= 0 && one.status == '未开始') || one.status == '进行中')
					) {
						//当处于会议开始25分钟或者开始时，进入选项可以点击（不被禁用）
						one.canEnterMeeting = true;
					} else {
						//否则进入选项被禁用
						one.canEnterMeeting = false;
					}
				
				}
				
				//将数据存储到模型层中的dataList和totalCount中
				that.dataList = page.list;
				that.totalCount = page.totalCount;
				that.dataListLoading = false;
				
				
			});
			
		},
		
		
		/**
		 * 勾选条数
		 * @param {Object} val
		 */
		sizeChangeHandle: function(val) {
		    this.pageSize = val;
		    this.pageIndex = 1;
		    this.loadDataList();
		},
		/**
		 * 点击跳转页面
		 * @param {Object} val
		 */
		currentChangeHandle: function(val) {
		    this.pageIndex = val;
		    this.loadDataList();
		},
        
		/**
		 * 查询页面
		 */
		searchHandle : function(){
			
			//表单校验
			this.$refs["dataForm"].validate(valid => {
				
				if(valid){
					
					this.$refs["dataForm"].clearValidate();
					
					if(this.pageIndex != 1){
						this.pageIndex = 1;
					}
					
					//渲染表单
					this.loadDataList();
					
				}else{
					return false;
				}
				
			});
			
		},
		
		changeHandle : function(val){
			this.searchHandle();
		},
		
		/**
		 * 申请会议
		 */
		addHandle : function(){
			
			this.addVisible = true;
			this.$nextTick(() =>{
				
				this.$refs.add.init();
			});
			
		},
		
		/**
		 * 会议申请成功之后，弹窗页面消失之后，online_meeting.vue页面需要更新内容
		 */
		refresh: function() {
		    this.$refs['dataForm'].resetFields();
		    this.loadDataList();
		},
		
		
		/**
		 * 删除会议
		 */
		deleteHandle : function(json){
			
			let that = this;
			that.$confirm("是否删除会议？", "提示", {
				
				confirmButtonText: '确定',
				cancelButtonText: '取消',
				type: 'warning',
				
			}).then(() => {
				
				let data = {
					id: json.id,
					uuid: json.uuid,
					instanceId: json.instanceId,
					reason: "删除会议申请"
				};
				
				//发送ajax
				that.$http("meeting/deleteMeetingApplication", "POST", data, true, function(resp){
					
					if(resp.rows == 1){
						that.$message({
							message: '删除成功',
							type: 'success',
							duration: 1200
						});
						that.searchHandle();
					}else{
						
						that.$message({
							message: '删除失败',
							type: 'error',
							duration: 1200
						});
						
					}
					
				});
			});
			
		},
		
		/**
		 * 通过路由跳转到meeting_video.vue页面，
		 * 然后传入meetingId和uuid俩个参数。让meeting_video.vue页面查询RoomID和参会人列表。
		 * 
		 * @param {Object} id(meetingId)
		 * @param {Object} uuid(userId)
		 */
		enterHandle: function(id, uuid){
			
			let that = this;
			that.$router.push({name: 'MeetingVideo', params: {meetingId: id, uuid: uuid}});
			
		}
		
		

    },
    created: function() {
        this.loadDataList();
    }
};
</script>

<style lang="less" scoped="scoped">
@import url('online_meeting.less');
</style>
