<template>
  <view class="yd-page-container pb-[80rpx]">
    <!-- 顶部导航栏 -->
    <wd-navbar
      title="审批详情"
      left-arrow placeholder safe-area-inset-top fixed
      @click-left="handleBack"
    />

    <!-- 区域：流程信息（基本信息） -->
    <view class="mx-24rpx mt-24rpx overflow-hidden rounded-16rpx bg-white">
      <view class="p-24rpx">
        <!-- 标题和状态 -->
        <view class="mb-16rpx flex items-center justify-between">
          <text class="text-32rpx text-[#333] font-bold">{{ processInstance?.name }}</text>
          <wd-tag :type="getStatusType(processInstance?.status)">
            {{ getStatusText(processInstance?.status) }}
          </wd-tag>
        </view>
        <!-- 发起人信息 -->
        <view class="flex items-center">
          <view class="mr-12rpx h-64rpx w-64rpx flex items-center justify-center rounded-full bg-[#1890ff] text-white">
            {{ processInstance?.startUser?.nickname?.[0] || '?' }}
          </view>
          <view>
            <text class="text-28rpx text-[#333]">{{ processInstance?.startUser?.nickname }}</text>
            <text v-if="processInstance?.startUser?.deptName" class="ml-8rpx text-24rpx text-[#999]">
              {{ processInstance?.startUser?.deptName }}
            </text>
          </view>
        </view>
        <!-- 提交时间 -->
        <view class="mt-16rpx text-24rpx text-[#999]">
          提交于 {{ formatDateTime(processInstance?.startTime) }}
        </view>
      </view>
    </view>

    <!-- 区域：审批详情（表单） -->
    <FormDetail :process-definition="processDefinition" :process-instance="processInstance" />

    <!-- 区域：审批进度 -->
    <view class="mx-24rpx mt-24rpx rounded-16rpx bg-white">
      <view class="p-24rpx">
        <view class="mb-16rpx flex">
          <text class="text-28rpx text-[#333] font-bold">审批进度</text>
        </view>
        <!-- 流程时间线 -->
        <ProcessInstanceTimeline :activity-nodes="activityNodes" />
      </view>
    </view>

    <!-- TODO 待开发：区域：流程评论 -->

    <!-- 区域：底部操作栏 -->
    <ProcessInstanceOperationButton ref="operationButtonRef" />
  </view>
</template>

<script lang="ts" setup>
import type { ApprovalNodeInfo, ProcessDefinition, ProcessInstance } from '@/api/bpm/processInstance'
import type { Task } from '@/api/bpm/task'
import { onMounted, ref } from 'vue'
import { useToast } from 'wot-design-uni'
import { getApprovalDetail } from '@/api/bpm/processInstance'
import { getTaskListByProcessInstanceId } from '@/api/bpm/task'
import { navigateBackPlus } from '@/utils'
import { formatDateTime } from '@/utils/date'
import FormDetail from './components/form-detail.vue'
import ProcessInstanceOperationButton from './components/operation-button.vue'
import ProcessInstanceTimeline from './components/time-line.vue'

const props = defineProps<{
  id: string // 流程实例的编号
  taskId?: string // 任务编号
}>()

definePage({
  style: {
    navigationBarTitleText: '',
    navigationStyle: 'custom',
  },
})

const toast = useToast()
const processInstance = ref<ProcessInstance>()
const processDefinition = ref<ProcessDefinition>()
const tasks = ref<Task[]>([])

const activityNodes = ref<ApprovalNodeInfo[]>([]) // 审批节点信息

const operationButtonRef = ref() // 操作按钮组件 ref

/** 返回上一页 */
function handleBack() {
  navigateBackPlus('/pages/bpm/index')
}

/** 获取状态文本 */
// TODO @jason：要有标签，和 vben 一样，盖章
// TODO @jason：通过字典
function getStatusText(status?: number) {
  const map: Record<number, string> = {
    0: '待审批',
    1: '审批中',
    2: '审批通过',
    3: '审批不通过',
    4: '已取消',
    5: '已退回',
    6: '委派中',
    7: '审批通过中',
  }
  return map[status ?? 0] || '待审批'
}

/** 获取状态标签类型 */
function getStatusType(status?: number): 'default' | 'primary' | 'success' | 'warning' | 'danger' {
  if ([1, 6, 7].includes(status ?? 0)) {
    return 'primary'
  }
  if (status === 2) {
    return 'success'
  }
  if (status === 3) {
    return 'danger'
  }
  if (status === 4 || status === 5) {
    return 'warning'
  }
  return 'default'
}

/** 获取任务圆点样式 */
// TODO @jason：看看又要对齐 vben
function getTaskDotClass(task: Task) {
  if ([1, 6, 7].includes(task.status)) {
    return 'bg-[#1890ff]'
  }
  if (task.status === 2) {
    return 'bg-[#52c41a]'
  }
  if (task.status === 3) {
    return 'bg-[#ff4d4f]'
  }
  if (task.status === 5) {
    return 'bg-[#faad14]'
  }
  return 'bg-[#d9d9d9]'
}

/** 获取状态文本样式 */
// TODO @jason：看看又要对齐 vben
function getStatusTextClass(status: number) {
  if ([1, 6, 7].includes(status)) {
    return 'text-[#1890ff]'
  }
  if (status === 2) {
    return 'text-[#52c41a]'
  }
  if (status === 3) {
    return 'text-[#ff4d4f]'
  }
  if (status === 5) {
    return 'text-[#faad14]'
  }
  return 'text-[#999]'
}

/** 加载流程实例 */
async function loadProcessInstance() {
  const data = await getApprovalDetail({
    processInstanceId: props.id,
    taskId: props.taskId,
  })
  if (!data || !data.processInstance) {
    toast.show('查询不到审批详情信息')
    return
  }
  processInstance.value = data.processInstance
  processDefinition.value = data.processDefinition
  // 获取审批节点，显示 Timeline 的数据
  activityNodes.value = data.activityNodes

  operationButtonRef.value?.init(data.processInstance, data.todoTask)
}

/** 加载任务列表 */
async function loadTasks() {
  tasks.value = await getTaskListByProcessInstanceId(props.id)
}

/** 初始化 */
onMounted(async () => {
  if (!props.id) {
    toast.show('参数错误')
    return
  }
  await Promise.all([loadProcessInstance(), loadTasks()])
})
</script>
