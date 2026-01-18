<template>
  <view class="yd-page-container">
    <!-- TODO @jason：还有一些细节，在审批通过没搞完！1）签名；2）选择审批人；3）其它等等 -->
    <!-- 顶部导航栏 -->
    <wd-navbar
      :title="isApprove ? '审批同意' : '审批拒绝'"
      left-arrow placeholder safe-area-inset-top fixed
      @click-left="handleBack"
    />

    <!-- 审批表单 -->
    <view class="p-24rpx">
      <wd-form ref="formRef" :model="formData" :rules="formRules">
        <wd-cell-group border>
          <!-- 审批意见 -->
          <wd-textarea
            v-model="formData.reason"
            prop="reason"
            label="审批意见："
            label-width="180rpx"
            placeholder="请输入审批意见"
            :maxlength="500"
            show-word-limit
            clearable
          />
        </wd-cell-group>
      </wd-form>

      <!-- 提交按钮 -->
      <view class="mt-48rpx">
        <wd-button
          :type="isApprove ? 'primary' : 'error'"
          block
          :loading="formLoading"
          :disabled="formLoading"
          @click="handleSubmit"
        >
          {{ isApprove ? '同意' : '拒绝' }}
        </wd-button>
      </view>
    </view>
  </view>
</template>

<script lang="ts" setup>
import type { FormInstance } from 'wot-design-uni/components/wd-form/types'
import { computed, onMounted, reactive, ref } from 'vue'
import { useToast } from 'wot-design-uni'
import { approveTask, rejectTask } from '@/api/bpm/task'
import { navigateBackPlus } from '@/utils'

const props = defineProps<{
  processInstanceId?: string
  taskId?: string
  pass?: string
}>()

definePage({
  style: {
    navigationBarTitleText: '',
    navigationStyle: 'custom',
  },
})

const taskId = computed(() => props.taskId || '')
const processInstanceId = computed(() => props.processInstanceId)
const isPass = computed(() => props.pass !== 'false') // true: 同意, false: 拒绝
const toast = useToast()
const formLoading = ref(false)
const formData = reactive({
  reason: '',
})

const formRules = {
  reason: [
    { required: true, message: '审批意见不能为空' },
  ],
}

const formRef = ref<FormInstance>()

/** 是否为同意操作 */
const isApprove = computed(() => isPass.value)

/** 返回上一页 */
function handleBack() {
  navigateBackPlus(`/pages-bpm/processInstance/detail/index?id=${processInstanceId.value}&taskId=${taskId.value}`)
}

/** 提交审批 */
async function handleSubmit() {
  if (formLoading.value) {
    return
  }
  const { valid } = await formRef.value!.validate()
  if (!valid) {
    return
  }

  formLoading.value = true
  try {
    const api = isApprove.value ? approveTask : rejectTask
    await api({
      id: taskId.value as string,
      reason: formData.reason,
    })
    toast.success('审批成功')
    setTimeout(() => {
      uni.redirectTo({
        url: `/pages-bpm/processInstance/detail/index?id=${processInstanceId.value}&taskId=${taskId.value}`,
      })
    }, 1000)
  } finally {
    formLoading.value = false
  }
}

/** 页面加载时 */
onMounted(() => {
  /** 初始化校验 */
  if (!props.taskId || !props.processInstanceId) {
    toast.show('参数错误')
  }
})
</script>
