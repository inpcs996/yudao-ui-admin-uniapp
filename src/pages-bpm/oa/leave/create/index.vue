<template>
  <view class="yd-page-container">
    <!-- 顶部导航栏 -->
    <wd-navbar
      :title="formData.id ? '编辑请假' : '发起请假'"
      left-arrow placeholder safe-area-inset-top fixed
      @click-left="handleBack"
    />

    <!-- 表单内容 -->
    <view class="p-24rpx">
      <!-- 基本信息卡片 -->
      <view class="overflow-hidden rounded-16rpx bg-white">
        <view class="p-24rpx">
          <view class="mb-24rpx text-32rpx text-[#333] font-bold">
            请假信息
          </view>
          <!-- 请假类型 -->
          <view class="mb-24rpx">
            <view class="mb-12rpx text-28rpx text-[#333]">
              <text class="text-[#ff4d4f]">*</text> 请假类型
            </view>
            <wd-picker
              v-model="formData.type"
              :columns="leaveTypeOptions"
              label=""
              placeholder="请选择请假类型"
            />
          </view>
          <!-- 开始时间 -->
          <view class="mb-24rpx">
            <view class="mb-12rpx text-28rpx text-[#333]">
              <text class="text-[#ff4d4f]">*</text> 开始时间
            </view>
            <wd-datetime-picker
              v-model="formData.startTime"
              type="datetime"
              label=""
              placeholder="请选择开始时间"
            />
          </view>
          <!-- 结束时间 -->
          <view class="mb-24rpx">
            <view class="mb-12rpx text-28rpx text-[#333]">
              <text class="text-[#ff4d4f]">*</text> 结束时间
            </view>
            <wd-datetime-picker
              v-model="formData.endTime"
              type="datetime"
              label=""
              placeholder="请选择结束时间"
            />
          </view>
          <!-- 请假原因 -->
          <view>
            <view class="mb-12rpx text-28rpx text-[#333]">
              <text class="text-[#ff4d4f]">*</text> 请假原因
            </view>
            <wd-textarea
              v-model="formData.reason"
              placeholder="请输入请假原因"
              :maxlength="200"
              show-word-limit
            />
          </view>
        </view>
      </view>

      <!-- TODO：流程预览卡片 -->
      <!-- 原始 vben 版本有流程节点预览和发起人选择审批人功能 -->
      <!-- 参考：yudao-ui-admin-vben-v5/apps/web-antd/src/views/bpm/oa/leave/create.vue 第 40-50 行 -->
      <!-- uniapp 端暂不实现流程节点预览，因为需要复杂的 ProcessInstanceTimeline 组件 -->
      <view class="mt-24rpx overflow-hidden rounded-16rpx bg-white">
        <view class="p-24rpx">
          <view class="mb-16rpx text-32rpx text-[#333] font-bold">
            流程信息
          </view>
          <view class="text-28rpx text-[#999]">
            提交后将进入审批流程
          </view>
          <!-- TODO：实现流程节点预览 -->
          <!-- 参考：yudao-ui-admin-vben-v5/apps/web-antd/src/views/bpm/oa/leave/create.vue 第 40-50 行 -->
          <!-- 需要实现 ProcessInstanceTimeline 组件和 getApprovalDetail API -->
        </view>
      </view>
    </view>

    <!-- 底部提交按钮 -->
    <view class="yd-detail-footer">
      <view class="yd-detail-footer-actions">
        <wd-button type="default" class="flex-1" @click="handleBack">
          取消
        </wd-button>
        <wd-button type="primary" class="flex-1" :loading="formLoading" @click="handleSubmit">
          提交
        </wd-button>
      </view>
    </view>
  </view>
</template>

<script lang="ts" setup>
import type { Leave } from '@/api/bpm/oa/leave'
import { onLoad } from '@dcloudio/uni-app'
import { computed, ref } from 'vue'
import { useMessage, useToast } from 'wot-design-uni'
import { createLeave, updateLeave } from '@/api/bpm/oa/leave'
import { getIntDictOptions } from '@/hooks/useDict'
import { navigateBackPlus } from '@/utils'
import { DICT_TYPE } from '@/utils/constants'

definePage({
  style: {
    navigationBarTitleText: '',
    navigationStyle: 'custom',
  },
})

const toast = useToast()
const message = useMessage()
const formLoading = ref(false)

const formData = ref<Partial<Leave>>({
  type: undefined,
  startTime: undefined,
  endTime: undefined,
  reason: undefined,
})

/** 请假类型选项 */
const leaveTypeOptions = computed(() => {
  return getIntDictOptions(DICT_TYPE.BPM_OA_LEAVE_TYPE).map(item => ({
    label: item.label,
    value: item.value,
  }))
})

/** 返回上一页 */
function handleBack() {
  message.confirm({
    title: '提示',
    msg: '确定要返回吗？请先保存您填写的信息！',
  }).then(({ action }) => {
    if (action === 'confirm') {
      navigateBackPlus('/pages-bpm/oa/leave/index')
    }
  })
}

/** 表单校验 */
function validateForm(): boolean {
  if (formData.value.type === undefined) {
    toast.show('请选择请假类型')
    return false
  }
  if (!formData.value.startTime) {
    toast.show('请选择开始时间')
    return false
  }
  if (!formData.value.endTime) {
    toast.show('请选择结束时间')
    return false
  }
  if (formData.value.startTime >= formData.value.endTime) {
    toast.show('结束时间必须大于开始时间')
    return false
  }
  if (!formData.value.reason?.trim()) {
    toast.show('请输入请假原因')
    return false
  }
  return true
}

/** 提交表单 */
async function handleSubmit() {
  if (!validateForm()) {
    return
  }

  // TODO：校验发起人选择审批人
  // 参考：yudao-ui-admin-vben-v5/apps/web-antd/src/views/bpm/oa/leave/create.vue 第 68-78 行
  // uniapp 端暂不实现发起人选择审批人功能

  try {
    formLoading.value = true
    const submitData: Partial<Leave> = {
      ...formData.value,
      startTime: Number(formData.value.startTime),
      endTime: Number(formData.value.endTime),
    }

    if (formData.value.id) {
      await updateLeave(submitData)
    } else {
      await createLeave(submitData)
    }

    uni.showToast({ title: '提交成功', icon: 'success' })
    // 返回列表页
    setTimeout(() => {
      uni.navigateBack()
    }, 1500)
  } catch (error) {
    console.error('[leave create] 提交失败:', error)
  } finally {
    formLoading.value = false
  }
}

/** 初始化 */
onLoad((options) => {
  // 如果有 id 参数，则为编辑模式
  if (options?.id) {
    // TODO：加载请假详情进行编辑
    // 参考：yudao-ui-admin-vben-v5/apps/web-antd/src/views/bpm/oa/leave/create.vue
    toast.show('编辑功能开发中')
  }
})
</script>
