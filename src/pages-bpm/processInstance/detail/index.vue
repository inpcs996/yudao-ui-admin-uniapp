<template>
  <view class="yd-page-container">
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

    <!-- 区域：审批记录 TODO @jason：抽成类似 /Users/yunai/Java/yudao-ui-admin-vben-v5/apps/web-antd/src/views/bpm/processInstance/detail/modules/task-list.vue -->
    <view class="mx-24rpx mt-24rpx overflow-hidden rounded-16rpx bg-white">
      <view class="p-24rpx">
        <view class="mb-16rpx flex items-center justify-between">
          <text class="text-28rpx text-[#333] font-bold">审批记录</text>
          <!-- TODO @AI：去掉 orderAsc，不要 toggleOrder -->
          <wd-icon :name="orderAsc ? 'arrow-up' : 'arrow-down'" size="32rpx" @click="toggleOrder" />
        </view>
        <!-- 任务列表 -->
        <view v-for="(task, index) in sortedTasks" :key="task.id || index" class="relative pb-24rpx pl-40rpx">
          <!-- 时间线 -->
          <view
            class="absolute left-12rpx top-8rpx h-16rpx w-16rpx rounded-full"
            :class="getTaskDotClass(task)"
          />
          <view v-if="index < sortedTasks.length - 1" class="absolute bottom-0 left-18rpx top-28rpx w-2rpx bg-[#e5e5e5]" />
          <!-- 任务内容 -->
          <view>
            <text class="text-28rpx text-[#333] font-bold">{{ task.name }}</text>
            <view v-if="task.assigneeUser" class="mt-8rpx flex items-center">
              <view class="mr-8rpx h-48rpx w-48rpx flex items-center justify-center rounded-full bg-[#1890ff] text-24rpx text-white">
                {{ task.assigneeUser?.nickname?.[0] || '?' }}
              </view>
              <view class="flex-1">
                <view class="flex items-center justify-between">
                  <view class="flex items-center">
                    <text class="text-26rpx text-[#333]">{{ task.assigneeUser?.nickname }}</text>
                    <text v-if="task.assigneeUser?.deptName" class="ml-8rpx text-22rpx text-[#999]">
                      {{ task.assigneeUser?.deptName }}
                    </text>
                  </view>
                  <text v-if="task.endTime" class="text-22rpx text-[#999]">{{ formatPast(task.endTime) }}</text>
                </view>
                <view class="mt-4rpx flex items-center">
                  <text :class="getStatusTextClass(task.status)" class="text-24rpx">
                    {{ getStatusText(task.status) }}
                  </text>
                  <text v-if="task.reason" class="ml-8rpx text-24rpx text-[#666]">{{ task.reason }}</text>
                </view>
              </view>
            </view>
          </view>
        </view>
      </view>
    </view>

    <!-- TODO 待开发：区域：流程评论 -->

    <!-- 区域：底部操作栏 -->
    <ProcessInstanceOperationButton ref="operationButtonRef" :process-instance="processInstance" />
  </view>
</template>

<script lang="ts" setup>
import type { ProcessDefinition, ProcessInstance } from '@/api/bpm/processInstance'
import type { Task } from '@/api/bpm/task'
import { computed, onMounted, ref } from 'vue'
import { useToast } from 'wot-design-uni'
import { getApprovalDetail } from '@/api/bpm/processInstance'
import { getTaskListByProcessInstanceId } from '@/api/bpm/task'
import { useUserStore } from '@/store'
import { navigateBackPlus } from '@/utils'
import { formatDateTime, formatPast } from '@/utils/date'
import FormDetail from './components/form-detail.vue'
import ProcessInstanceOperationButton from './components/operation-button.vue'

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

const userStore = useUserStore()
const toast = useToast()
const processInstance = ref<ProcessInstance>()
const processDefinition = ref<ProcessDefinition>()
const tasks = ref<Task[]>([])
const orderAsc = ref(true)

// 操作按钮组件 ref
const operationButtonRef = ref()

/** 排序后的任务列表 */
const sortedTasks = computed(() => {
  const list = [...tasks.value].filter(t => t.status !== 4) // 过滤已取消
  list.sort((a, b) => {
    if (a.endTime && b.endTime) {
      return orderAsc.value ? a.endTime - b.endTime : b.endTime - a.endTime
    }
    if (a.endTime) {
      return orderAsc.value ? -1 : 1
    }
    if (b.endTime) {
      return orderAsc.value ? 1 : -1
    }
    return orderAsc.value ? a.createTime - b.createTime : b.createTime - a.createTime
  })
  return list
})

/** 返回上一页 */
function handleBack() {
  navigateBackPlus('/pages/bpm/index')
}

/** 切换排序 */
function toggleOrder() {
  orderAsc.value = !orderAsc.value
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
  const param = {
    processInstanceId: props.id,
    taskId: props.taskId,
  }
  const data = await getApprovalDetail(param)
  if (!data || !data.processInstance) {
    toast.show('查询不到审批详情信息')
    return
  }
  processInstance.value = data.processInstance
  processDefinition.value = data.processDefinition
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
