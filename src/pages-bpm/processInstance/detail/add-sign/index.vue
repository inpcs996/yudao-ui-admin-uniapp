<template>
  <view class="yd-page-container">
    <!-- 顶部导航栏 -->
    <wd-navbar
      title="加签任务"
      left-arrow placeholder safe-area-inset-top fixed
      @click-left="handleBack"
    />

    <!-- 操作表单 -->
    <view class="p-24rpx">
      <wd-form ref="formRef" :model="formData" :rules="formRules">
        <wd-cell-group border>
          <!-- 加签处理人 -->
          <UserPicker
            v-model="formData.userIds"
            prop="userIds"
            type="checkbox"
            label="加签处理人："
            label-width="200rpx"
            placeholder="请选择加签处理人"
            :rules="formRules.userIds"
          />

          <!-- 审批意见 -->
          <wd-textarea
            v-model="formData.reason"
            prop="reason"
            label="审批意见："
            label-width="200rpx"
            placeholder="请输入审批意见"
            :maxlength="500"
            show-word-limit
            clearable
          />
        </wd-cell-group>
        <!-- 提交按钮 -->
        <view class="mt-48rpx flex gap-16rpx">
          <wd-button
            type="primary"
            class="flex-1"
            plain
            :loading="submitting"
            :disabled="submitting"
            @click="handleSubmit('before')"
          >
            向前加签
          </wd-button>
          <wd-button
            type="primary"
            class="flex-1"
            :loading="submitting"
            :disabled="submitting"
            @click="handleSubmit('after')"
          >
            向后加签
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
import { signCreateTask } from '@/api/bpm/task'
import UserPicker from '@/components/system-select/user-picker.vue'
import { navigateBackPlus } from '@/utils'

const props = defineProps<{
  processInstanceId: string
  taskId: string
}>()

definePage({
  style: {
    navigationBarTitleText: '',
    navigationStyle: 'custom',
  },
})

const taskId = computed(() => props.taskId)
const processInstanceId = computed(() => props.processInstanceId)
const toast = useToast()
const submitting = ref(false)
const formRef = ref<FormInstance>()

const formData = reactive({
  userIds: [] as number[],
  reason: '',
})

const formRules = {
  userIds: [
    { required: true, message: '加签处理人不能为空', validator: (value: number[]) => value.length > 0 },
  ],
  reason: [
    { required: true, message: '审批意见不能为空' },
  ],
}

/** 返回上一页 */
function handleBack() {
  navigateBackPlus(`/pages-bpm/processInstance/detail/index?id=${processInstanceId.value}&taskId=${taskId.value}`)
}

/** 初始化校验 */
if (!props.taskId || !props.processInstanceId) {
  toast.show('参数错误')
}

/** 提交操作 */
async function handleSubmit(type: 'before' | 'after') {
  if (submitting.value)
    return

  // 使用 wd-form 的校验方法
  const { valid } = await formRef.value!.validate()
  if (!valid) {
    return
  }
  submitting.value = true
  try {
    const result = await signCreateTask({
      id: taskId.value as string,
      type,
      userIds: formData.userIds,
      reason: formData.reason,
    })

    if (result) {
      const actionText = type === 'before' ? '向前加签' : '向后加签'
      toast.success(`${actionText}成功`)
      setTimeout(() => {
        uni.redirectTo({
          url: `/pages-bpm/processInstance/detail/index?id=${processInstanceId.value}&taskId=${taskId.value}`,
        })
      }, 1500)
    }
  } catch (error) {
    const actionText = type === 'before' ? '向前加签' : '向后加签'
    console.error(`[add-sign] ${actionText}失败:`, error)
    toast.error(`${actionText}失败`)
  } finally {
    submitting.value = false
  }
}
</script>
