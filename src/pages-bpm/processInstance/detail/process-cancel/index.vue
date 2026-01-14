<template>
  <view class="yd-page-container">
    <!-- 顶部导航栏 -->
    <wd-navbar
      title="取消流程"
      left-arrow placeholder safe-area-inset-top fixed
      @click-left="handleBack"
    />

    <!-- 操作表单 -->
    <view class="p-24rpx">
      <wd-form ref="formRef" :model="formData" :rules="formRules">
        <wd-cell-group border>
          <!-- 友情提醒 -->
          <view class="mb-24rpx border border-[#ffd591] rounded-16rpx bg-[#fff7e6] p-24rpx">
            <view class="mb-12rpx flex items-center">
              <wd-icon name="warning" color="#faad14" size="32rpx" />
              <text class="ml-12rpx text-28rpx text-[#faad14] font-bold">友情提醒</text>
            </view>
            <text class="text-26rpx text-[#666]">取消后，该审批流程将自动结束。</text>
          </view>

          <!-- 取消理由 -->
          <wd-textarea
            v-model="formData.cancelReason"
            prop="cancelReason"
            label="取消理由："
            label-width="180rpx"
            placeholder="请输入取消理由"
            :maxlength="500"
            show-word-limit
            clearable
          />
        </wd-cell-group>
        <!-- 提交按钮 -->
        <view class="mt-48rpx">
          <wd-button
            type="primary"
            block
            :loading="submitting"
            :disabled="submitting"
            @click="handleSubmit"
          >
            确认取消
          </wd-button>
        </view>
      </wd-form>
    </view>
  </view>
</template>

<script lang="ts" setup>
import type { FormInstance } from 'wot-design-uni/components/wd-form/types'
import { computed, reactive, ref } from 'vue'
import { useToast } from 'wot-design-uni'
import { cancelProcessInstanceByStartUser } from '@/api/bpm/processInstance'
import { navigateBackPlus } from '@/utils'

const props = defineProps<{
  processInstanceId: string
  taskId?: string
}>()

definePage({
  style: {
    navigationBarTitleText: '',
    navigationStyle: 'custom',
  },
})

const processInstanceId = computed(() => props.processInstanceId)
const taskId = computed(() => props.taskId)
const toast = useToast()
const submitting = ref(false)
const formRef = ref<FormInstance>()

const formData = reactive({
  cancelReason: '',
})

const formRules = {
  cancelReason: [
    { required: true, message: '取消理由不能为空' },
  ],
}

/** 返回上一页 */
function handleBack() {
  const backUrl = taskId.value
    ? `/pages-bpm/processInstance/detail/index?id=${processInstanceId.value}&taskId=${taskId.value}`
    : `/pages-bpm/processInstance/detail/index?id=${processInstanceId.value}`
  navigateBackPlus(backUrl)
}

/** 初始化校验 */
if (!props.processInstanceId) {
  toast.show('参数错误')
}

/** 提交操作 */
async function handleSubmit() {
  if (submitting.value)
    return

  // 使用 wd-form 的校验方法
  const { valid } = await formRef.value!.validate()
  if (!valid) {
    return
  }

  submitting.value = true
  try {
    const result = await cancelProcessInstanceByStartUser(
      processInstanceId.value,
      formData.cancelReason,
    )

    if (result) {
      toast.success('流程取消成功')
      setTimeout(() => {
        uni.redirectTo({
          url: `/pages-bpm/processInstance/detail/index?id=${processInstanceId.value}`,
        })
      }, 1500)
    }
  } catch (error) {
    console.error('[process-cancel] 取消流程失败:', error)
    toast.error('取消流程失败')
  } finally {
    submitting.value = false
  }
}
</script>
