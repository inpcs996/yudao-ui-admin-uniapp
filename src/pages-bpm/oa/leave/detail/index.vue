<template>
  <view class="yd-page-container">
    <!-- 顶部导航栏 -->
    <wd-navbar
      title="请假详情"
      left-arrow placeholder safe-area-inset-top fixed
      @click-left="handleBack"
    />

    <!-- 加载中 -->
    <view v-if="loading" class="flex items-center justify-center py-100rpx">
      <wd-loading />
    </view>

    <!-- 详情内容 -->
    <view v-else class="p-24rpx">
      <!-- 基本信息卡片 -->
      <view class="overflow-hidden rounded-16rpx bg-white">
        <view class="p-24rpx">
          <view class="mb-24rpx text-32rpx text-[#333] font-bold">
            请假信息
          </view>
          <!-- 请假类型 -->
          <view class="mb-16rpx flex items-center">
            <text class="w-160rpx text-28rpx text-[#999]">请假类型</text>
            <dict-tag :type="DICT_TYPE.BPM_OA_LEAVE_TYPE" :value="formData.type" />
          </view>
          <!-- 开始时间 -->
          <view class="mb-16rpx flex items-center">
            <text class="w-160rpx text-28rpx text-[#999]">开始时间</text>
            <text class="text-28rpx text-[#333]">{{ formatDateTime(formData.startTime) }}</text>
          </view>
          <!-- 结束时间 -->
          <view class="mb-16rpx flex items-center">
            <text class="w-160rpx text-28rpx text-[#999]">结束时间</text>
            <text class="text-28rpx text-[#333]">{{ formatDateTime(formData.endTime) }}</text>
          </view>
          <!-- 请假原因 -->
          <view class="flex">
            <text class="w-160rpx text-28rpx text-[#999]">请假原因</text>
            <text class="flex-1 text-28rpx text-[#333]">{{ formData.reason }}</text>
          </view>
        </view>
      </view>

      <!-- 审批状态卡片 -->
      <view class="mt-24rpx overflow-hidden rounded-16rpx bg-white">
        <view class="p-24rpx">
          <view class="mb-24rpx text-32rpx text-[#333] font-bold">
            审批状态
          </view>
          <view class="flex items-center">
            <text class="w-160rpx text-28rpx text-[#999]">当前状态</text>
            <dict-tag :type="DICT_TYPE.BPM_PROCESS_INSTANCE_STATUS" :value="formData.status" />
          </view>
        </view>
      </view>

      <!-- 查看审批进度按钮 -->
      <view v-if="formData.processInstanceId" class="mt-24rpx">
        <wd-button type="primary" block @click="handleProgress">
          查看审批进度
        </wd-button>
      </view>
    </view>
  </view>
</template>

<script lang="ts" setup>
import type { Leave } from '@/api/bpm/oa/leave'
import { onLoad } from '@dcloudio/uni-app'
import { ref } from 'vue'
import { useToast } from 'wot-design-uni'
import { getLeave } from '@/api/bpm/oa/leave'
import { navigateBackPlus } from '@/utils'
import { DICT_TYPE } from '@/utils/constants'
import { formatDateTime } from '@/utils/date'

definePage({
  style: {
    navigationBarTitleText: '',
    navigationStyle: 'custom',
  },
})

const toast = useToast()
const loading = ref(false)
const formData = ref<Partial<Leave>>({})

/** 返回上一页 */
function handleBack() {
  navigateBackPlus('/pages-bpm/oa/leave/index')
}

/** 查看审批进度 */
function handleProgress() {
  if (formData.value.processInstanceId) {
    uni.navigateTo({ url: `/pages-bpm/processInstance/detail/index?id=${formData.value.processInstanceId}` })
  }
}

/** 获取详情数据 */
async function getDetailData(id: number) {
  try {
    loading.value = true
    formData.value = await getLeave(id)
  } catch (error) {
    console.error('[leave detail] 获取详情失败:', error)
  } finally {
    loading.value = false
  }
}

/** 初始化 */
onLoad((options) => {
  if (!options?.id) {
    toast.show('参数错误')
    return
  }
  getDetailData(Number(options.id))
})
</script>
