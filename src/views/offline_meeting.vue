<template>
	<div>
		<el-form :inline="true" :model="dataForm" :rules="dataRule" ref="dataForm" class="form">
			<el-form-item prop="name">
				<el-select
					v-model="dataForm.name"
					class="input"
					placeholder="选择会议室"
					size="medium"
					clearable="clearable"
				>
					<el-option v-for="one in roomList" :label="one.name" :value="one.name" />
				</el-select>
			</el-form-item>
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
		<!-- 甘特图的样式, 当v-show=true成立时，显示甘特图 -->
		<div class="gantt" ref="gantt" v-show="mode == 'gantt'">
			<!-- 渲染甘特图时间 -->
			<div class="row">
				<div class="cell-time"></div>
				<div class="cell-time" v-for="one in time">
					<span class="time">{{ one }}</span>
				</div>
			</div>
			<!-- 渲染甘特图中的会议室信息 -->
			<div class="row" v-for="room in gantt.meetingRoom">
				<!-- 渲染会议室的名字 name -->
				<div class="cell room">{{ room.name }}</div>
				<!-- 渲染会议的时间 meeting -->
				<div class="cell" v-for="one in time">
					<div
						v-if="room.meeting.hasOwnProperty(one)"
						class="meeting"
						:class="room.meeting[one].split('#')[1]"
						:style="'width:' + room.meeting[one].split('#')[0] * gantt.cellWidth + 'px'"
					></div>
				</div>
			</div>
		</div>
		<!-- 分页条 -->
		<el-pagination
			v-show="mode == 'gantt'"
			@size-change="sizeChangeHandle"
			@current-change="currentChangeHandle"
			:current-page="pageIndex"
			:page-sizes="[10, 20, 50]"
			:page-size="pageSize"
			:total="totalCount"
			layout="total, sizes, prev, pager, next, jumper"
		/>

		<!-- 周日历的渲染方式  当v-show=true成立时，展示周日历-->
		<div class="calendar" v-show="mode == 'calendar'">
			<!-- 表头，为时间和7个日期时间行 -->
			<div class="row">
				<div class="cell">时间</div>
				<div class="cell" v-for="one in calendar.days">{{ one.date }}（{{ one.day }}）</div>
			</div>
			<!-- 双层循环，循环出各个行和列 -->
			<div class="row" v-for="(one, index) in time">
				<!-- 循环出时间段 -->
				<div class="cell-time" v-if="time[index + 1] != null">{{ one }} ~ {{ time[index + 1] }}</div>
				<div class="cell" v-for="day in calendar.days" v-if="time[index + 1] != null">
					<!-- 渲染有会议时间段的页面,且click中的infoHandle 为弹出详细信息的卡片 -->
					<div
						class="meeting"
						v-if="calendar.map.hasOwnProperty(`${day.date}#${one}`)"
						:style="
							'height:' +
								calendar.map[day.date + '#' + one].time * 31 +
								'px; line-height:' +
								calendar.map[day.date + '#' + one].time * 31 +
								'px'
						"
						:ref="`${day.date}#${one}`"
						@click="
							infoHandle(calendar.map[day.date + '#' + one].id, calendar.map[day.date + '#' + one].status)
						"
					>
					<!-- 会议申请的 "x" 按钮，用于删除申请的会议 判断creator是否是会议申请人，从而判断能否删除， 且看是1，3状态-->
					<!-- SvIcon是图标， click.stop是阻止子控件触发父控件 -->
						<SvgIcon
							name="close"
							class="icon-svg-close"
							v-if="
								calendar.map[`${day.date}#${one}`].isCreator == 'true' &&
									[1, 3].includes(calendar.map[`${day.date}#${one}`].status)
							"
							@click.stop="deleteHandle(`${day.date}#${one}`)"
						/>
						<!-- 提取出会议属性的标题 -->
						{{ calendar.map[`${day.date}#${one}`].title }}
					</div>
				</div>
			</div>
		</div>
		<add v-if="addVisible" ref="add" @refresh="refresh"></add>
		<info v-if="infoVisble" ref="info"></info>
	</div>
</template>

<script>
import SvgIcon from '../components/SvgIcon.vue';
import dayjs from 'dayjs';
import Add from './offline_meeting-add.vue';
import Info from './offline_meeting-info.vue';
export default {
	components: { SvgIcon, Add, Info },
	data: function() {
		return {
			//甘特图时间
			time: [
				'08:30',
				'09:00',
				'09:30',
				'10:00',
				'10:30',
				'11:00',
				'11:30',
				'12:00',
				'12:30',
				'13:00',
				'13:30',
				'14:00',
				'14:30',
				'15:00',
				'15:30',
				'16:00',
				'16:30',
				'17:00',
				'17:30',
				'18:00',
				'18:30'
			],
			//甘特图的各种业务数据
			gantt: {
				meetingRoom: [],		//meetingRoom中保存{name, meeting{"13:00" : "4#blue", "09:00" : "4#blue"}}信息
				cellWidth: 0
			},
			//周日历的各种业务数据
			calendar: {
				map: {},
				days: []
			},
			//按条件查询的下拉列表数据
			roomList: [],
			dataForm: {
				name: null,
				date: null,
				mold: '全部会议'
			},
			pageIndex: 1,
			pageSize: 10,
			totalCount: 0,
			dataListLoading: false,
			addVisible: false,
			infoVisble: false,
			dataRule: {},
			//显示那种界面（日程表或者周日历）
			mode: 'gantt'
		};
	},
	methods: {
		backHandle: function() {
			let that = this;
			that.mode = 'gantt';
		},
		
		/**
		 * 渲染数据
		 */
		loadDataList: function(){
			
			let that = this;
			
			that.dataListLoading = false;
			//初始化data数据
			let data = {
				
				name: that.dataForm.name,
				mold: that.dataForm.mold,
				page: that.pageIndex,
				length: that.pageSize
				
			};
			
			//如果没选会议时间，则分配当前时间
			if(that.dataForm.date == null || that.dataForm.date == ''){
				data.date = dayjs(new Date()).format('YYYY-MM-DD');
			}else{//选择了会议，则赋值会议给它
				data.date = dayjs(that.dataForm.date).format('YYYY-MM-DD')
			}
			
			that.$http("meeting/searchOfflineMeetingByPage", "POST", data, true, function(resp){
				//resp{page{list, totalCount, pageSize, pageIndex, totalPage}}
				let page = resp.page;
				//定义一个temp的参数，用来存储重构后的name 和 meeting属性
				let temp = [];
				//查询出来的list : {name: meeting{"end","time","start", "status"}}
				for(let room of page.list){
					//定义一个json变量，将list的值遍历存储到json变量中的name 和 meeting属性中
					let json = {};
					json.name = room.name;
					json.meeting = {};
					//判断room中是否有"meeting"这个属性
					if(room.hasOwnProperty("meeting")){
						for(let meeting of room.meeting){
							let color;
							//根据status的值，使其在甘特图上显示不同的颜色
							if (meeting.status == 1) {		//待审批状态，status==2为已拒绝状态
								color = 'yellow';
							} else if (meeting.status == 3) {	//未开始状态
								color = 'blue';
							} else if (meeting.status == 4) {	//进行中状态
								color = 'pink';
							} else if (meeting.status == 5) {	//已结束状态
								color = 'gray';
							}
							//time属性表示以30分为一次，当9点到11点时，time=4
							json.meeting[meeting.start] = meeting.time + '#' + color;
						}
						
					}
					
					temp.push(json);			
				}
				//将temp值存储到meetingRoom中进行渲染  meetingRoom中保存{name, meeting{"13:00" : "4#blue", "09:00" : "4#blue"}}信息
				that.gantt.meetingRoom = temp;
				that.totalCount = page.totalCount;
				that.dataListLoading = false;
				
			});
			
		},
		
		/**
		 * 条件查询
		 */
		searchHandle: function() {
		    let that = this;
		
		    //查询甘特图数据
		    if (that.dataForm.name == null || that.dataForm.name == '') {
		        that.$refs['dataForm'].validate(valid => {
		            if (valid) {
		                that.$refs['dataForm'].clearValidate();
		                that.dataForm.name = null;
		                if (that.pageIndex != 1) {
		                    that.pageIndex = 1;
		                }
		                that.loadDataList();
		                that.mode = 'gantt';
		            } else {
		                return false;
		            }
		        });
		    }else{
				
				//校验表单
				that.$refs['dataForm'].validate(valid => {
				    if (valid) {
				        that.$refs['dataForm'].clearValidate();
						
						//设置data
						let data = {
							name: that.dataForm.name,
							mold: that.dataForm.mold
						};
						
						//日历控件的值默认为null，如有具体日期的日历被清除日期之后，值为空字符串
						if (that.dataForm.date != null && that.dataForm.date != '') {
							data.date = dayjs(that.dataForm.date).format('YYYY-MM-DD');
						}
						
						//发送ajax
						that.$http("meeting/searchOfflineMeetingInWeek", "POST", data, true, function(resp){
							
							let map = {};
							//视图层输出会议卡片做判断遍历数组太麻烦，所以把周日历数据从数组转换成JSON对象
							for (let one of resp.list) {
								map[`${one.date}#${one.start}`] = one;
							}
							
							that.calendar.map = map;
							//周日历表头标题
							that.calendar.days = resp.days;
							//将周日历表弄成calendar形式
							that.mode = 'calendar';
							
						});
					   
					   
				    } else {
				        return false;
				    }
				});
				
				
				
			}
		},
		
		//切换“我的会议”和“全部会议”时候触发事件的回调函数
		changeHandle: function(val) {
		    this.searchHandle();
		},
		
		sizeChangeHandle: function(val) {
		    this.pageSize = val;
		    this.pageIndex = 1;
		    this.loadDataList();
		},
		currentChangeHandle: function(val) {
		    this.pageIndex = val;
		    this.loadDataList();
		},
		
		
		/**
		 * 会议申请函数
		 */
		addHandle: function(){
			
			this.addVisible = true;
			
			this.$nextTick(() =>{
				this.$refs.add.init();
			});
		},
		
		/**
		 * refresh函数，由子组件offline-meeting-add调用
		 */
		refresh : function(){
			
			this.mode = 'gantt';
			//重置页面
			this.$refs["dataForm"].resetFields();
			//渲染画面
			this.loadDataList();
			
		},
		
		/**
		 * 初始化meeting-info页面
		 * @param {Object} id
		 * @param {Object} status
		 */
		infoHandle : function(id, status){
			//显示页面
			this.infoVisble = true;
			this.$nextTick(() => {
				
				this.$refs.info.init(id, status);
			});
			
		},
		
		/**
		 * 删除会议信息
		 */
		deleteHandle : function(key){
			
			let that = this;
			that.$confirm('是否删除该会议?', '提示', {
				confirmButtonText: '确定',
				cancelButtonText: '取消',
				type: 'warning',
			}).then(() => {
				
				let json = that.calendar.map[key];
				let data = {
					id: json.id,
					uuid: json.uuid,
					instanceId: json.instanceId,
					reason: '删除会议'
				};
				
				that.$http('meeting/deleteMeetingApplication', 'post', data, true, function(resp) {
					if (resp.rows == 1) {
						that.$message({
							message: '删除成功',
							type: 'success',
							duration: 1200
						});
						that.searchHandle();
					} else {
						that.$message({
							message: '删除失败',
							type: 'error',
							duration: 1200
						});
					}
				});
				
			});
			
		}
		
	},
	mounted: function() {
		let that=this
		//根据实际情况设置甘特图每个单元格应该有的宽度
		let rowWidth = that.$refs['gantt'].offsetWidth - 1;
		let cellWidth = rowWidth * 0.042 + 0.01;
		that.gantt.cellWidth = cellWidth;

		//当浏览器窗口尺寸改变的时候，重新设置甘特图单元格的宽度
		window.addEventListener('resize', () => {
			let rowWidth = that.$refs['gantt'].offsetWidth - 1;
			let cellWidth = rowWidth * 0.042 + 0.01;
			that.gantt.cellWidth = cellWidth;
		})
	},
	created: function() {
		
		let that = this;	
		//加载会议室列表,即下拉菜单中的会议室信息roomList
		that.$http('meeting_room/searchAllMeetingRoom', 'GET', null, true, function(resp) {
			that.roomList = resp.list;
		});

		//加载分页数据
		that.loadDataList();
	}
};
</script>

<style lang="less" scoped="scoped">
@import url('offline_meeting.less');
</style>
