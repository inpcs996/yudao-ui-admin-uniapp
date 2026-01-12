<!-- 操作按钮 -->
<template>
  <!-- 有待审批的任务 -->
  <view v-if="runningTask" class="yd-detail-footer">
    <view class="w-full flex items-center">
      <!-- 左侧操作按钮 -->
      <view v-for="(action, idx) in leftOperations" :key="idx" class="mr-32rpx w-60rpx flex flex-col items-center" @click="handleOperation(action.operationType)">
        <wd-icon :name="action.iconName" size="40rpx" color="#1890ff" />
        <text class="mt-4rpx text-22rpx text-[#333]">{{ action.displayName }}</text>
      </view>
      <!-- 更多操作按钮 -->
      <view v-if="moreOperations.length > 0" class="mr-32rpx w-60rpx flex flex-col items-center" @click="handleShowMore">
        <wd-icon name="ellipsis" size="40rpx" color="#1890ff" />
        <text class="mt-4rpx text-22rpx text-[#333]">更多</text>
      </view>
      <!-- 更多操作 ActionSheet -->
      <wd-action-sheet v-if="moreOperations.length > 0" v-model="showMoreActions" :actions="moreOperations" title="请选择操作" @select="handleMoreAction" />

      <!-- 右侧按钮，TODO 是否一定要保留两个按钮 -->
      <view class="flex flex-1 gap-16rpx">
        <wd-button
          v-for="(action, idx) in rightOperations"
          :key="idx"
          :plain="action.plain"
          :type="action.btnType"
          :round="false"
          class="flex-1"
          custom-style="min-width: 200rpx; width: 200rpx;"
          @click="handleOperation(action.operationType)"
        >
          {{ action.displayName }}
        </wd-button>
      </view>
    </view>
  </view>
  <!-- TODO 无待审批的任务 需要显示什么 -->
</template>

<script lang="ts" setup>
import type { Action } from 'wot-design-uni/components/wd-action-sheet/types'
import type { ButtonType } from 'wot-design-uni/components/wd-button/types'
import type { Task } from '@/api/bpm/task'
import { useToast } from 'wot-design-uni'
import {
  BpmTaskOperationButtonTypeEnum,
  BpmTaskStatusEnum,
  OPERATION_BUTTON_NAME,
} from '@/utils/constants'

const showMoreActions = ref(false)

type MoreOperationType = Action & {
  operationType: number
}

interface LeftOperationType {
  operationType: number
  iconName: string
  displayName: string
}

interface RightOperationType {
  operationType: number
  btnType: ButtonType
  displayName: string
  plain: boolean
}
const operationIconsMap: Record<number, string> = {
  [BpmTaskOperationButtonTypeEnum.TRANSFER]: 'transfer',
  [BpmTaskOperationButtonTypeEnum.ADD_SIGN]: 'add',
  [BpmTaskOperationButtonTypeEnum.DELEGATE]: 'share',
  [BpmTaskOperationButtonTypeEnum.RETURN]: 'arrow-left',
  [BpmTaskOperationButtonTypeEnum.COPY]: 'copy',
}
/** 左侧操作按钮 【最多两个】{转办, 委派, 退回, 加签， 抄送等} */
const leftOperations = ref<LeftOperationType[]>([])

/** 右侧操作按钮【最多两个】{通过，拒绝, 取消,减签} */
const rightOperationTypes = []
const rightOperations = ref<RightOperationType[]>([])
/** 更多操作 */
const moreOperations = ref<MoreOperationType[]>([])
const toast = useToast()
const runningTask = ref<Task>()
const reasonRequire = ref<boolean>(false)

/** 加载待办任务 */
function loadTodoTask(task: Task) {
  runningTask.value = task
  if (task) {
    reasonRequire.value = task.reasonRequire ?? false
    // 右侧按钮
    if (isHandleTaskStatus() && task.buttonsSetting && task.buttonsSetting[BpmTaskOperationButtonTypeEnum.REJECT]?.enable) {
      rightOperationTypes.push(BpmTaskOperationButtonTypeEnum.REJECT)
      rightOperations.value.push({
        operationType: BpmTaskOperationButtonTypeEnum.REJECT,
        displayName: getButtonDisplayName(BpmTaskOperationButtonTypeEnum.REJECT),
        btnType: 'error',
        plain: true,
      })
    }
    if (isHandleTaskStatus() && task.buttonsSetting && task.buttonsSetting[BpmTaskOperationButtonTypeEnum.APPROVE]?.enable) {
      rightOperationTypes.push(BpmTaskOperationButtonTypeEnum.APPROVE)
      rightOperations.value.push({
        operationType: BpmTaskOperationButtonTypeEnum.APPROVE,
        displayName: getButtonDisplayName(BpmTaskOperationButtonTypeEnum.APPROVE),
        btnType: 'primary',
        plain: false,
      })
    }
    // TODO 减签
    // 左侧操作，和更多操作
    Object.keys(task.buttonsSetting || {}).forEach((key) => {
      const operationType = Number(key)
      if (task.buttonsSetting[key].enable && isHandleTaskStatus()
        && !rightOperationTypes.includes(operationType)) {
        if (leftOperations.value.length >= 2) {
          moreOperations.value.push({
            name: getButtonDisplayName(operationType),
            operationType,
          })
        } else {
          leftOperations.value.push({
            operationType,
            iconName: operationIconsMap[operationType],
            displayName: getButtonDisplayName(operationType),
          })
        }
      }
    })
  }
}
/** 跳转到相应的操作页面 */
function handleOperation(operationType: number) {
  if (!runningTask.value) {
    return
  }
  switch (operationType) {
    case BpmTaskOperationButtonTypeEnum.APPROVE:
      uni.navigateTo({ url: `/pages-bpm/processInstance/detail/audit/index?id=${runningTask.value.id}&pass=true` })
      break
    case BpmTaskOperationButtonTypeEnum.REJECT:
      uni.navigateTo({ url: `/pages-bpm/processInstance/detail/audit/index?id=${runningTask.value.id}&pass=false` })
      break
    case BpmTaskOperationButtonTypeEnum.DELEGATE:
      uni.navigateTo({
        url: `/pages-bpm/processInstance/detail/reassign/index?processInstanceId=${runningTask.value.processInstanceId}&taskId=${runningTask.value.id}&type=delegate`,
      })
      break
    case BpmTaskOperationButtonTypeEnum.TRANSFER:
      uni.navigateTo({
        url: `/pages-bpm/processInstance/detail/reassign/index?processInstanceId=${runningTask.value.processInstanceId}&taskId=${runningTask.value.id}&type=transfer`,
      })
      break
    case BpmTaskOperationButtonTypeEnum.ADD_SIGN:
      uni.navigateTo({
        url: `/pages-bpm/processInstance/detail/add-sign/index?processInstanceId=${runningTask.value.processInstanceId}&taskId=${runningTask.value.id}`,
      })
      break
    case BpmTaskOperationButtonTypeEnum.RETURN:
      uni.navigateTo({
        url: `/pages-bpm/processInstance/detail/return/index?processInstanceId=${runningTask.value.processInstanceId}&taskId=${runningTask.value.id}`,
      })
      break
  }
}

/** 显示更多操作 */
function handleShowMore() {
  showMoreActions.value = true
}

/** 处理更多操作选择 */
function handleMoreAction(action: { item: MoreOperationType }) {
  handleOperation(action.item.operationType)
  showMoreActions.value = false
}

/** 获取按钮的显示名称 */
function getButtonDisplayName(btnType: BpmTaskOperationButtonTypeEnum) {
  let displayName = OPERATION_BUTTON_NAME.get(btnType)
  if (
    runningTask.value?.buttonsSetting
    && runningTask.value?.buttonsSetting[btnType]
  ) {
    displayName = runningTask.value.buttonsSetting[btnType].displayName
  }
  return displayName
}

/** 是否显示按钮 */
function isShowButton(btnType: BpmTaskOperationButtonTypeEnum): boolean {
  let isShow = true
  if (
    runningTask.value?.buttonsSetting
    && runningTask.value?.buttonsSetting[btnType]
  ) {
    isShow = runningTask.value.buttonsSetting[btnType].enable
  }
  return isShow
}

/** 任务是否为处理中状态 */
function isHandleTaskStatus() {
  let canHandle = false
  if (BpmTaskStatusEnum.RUNNING === runningTask.value?.status) {
    canHandle = true
  }
  return canHandle
}

/** 暴露方法 */
defineExpose({ loadTodoTask })
</script>
