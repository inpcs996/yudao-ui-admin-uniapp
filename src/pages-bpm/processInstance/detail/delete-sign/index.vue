<template>
  <view class="yd-page-container">
    <!-- 顶部导航栏 -->
    <wd-navbar
      title="减签任务"
      left-arrow placeholder safe-area-inset-top fixed
      @click-left="handleBack"
    />

    <!-- 操作表单 -->
    <view class="p-24rpx">
      <wd-form ref="formRef" :model="formData" :rules="formRules">
        <wd-cell-group border>
          <!-- 减签人员选择 -->
          <wd-picker
            v-model="formData.deleteSignTaskId"
            :columns="taskOptions"
            value-key="id"
            label-key="label"
            label="减签人员："
            label-width="180rpx"
            placeholder="请选择减签人员"
            prop="deleteSignTaskId"
          />

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
        <!-- 提交按钮 -->
        <view class="mt-48rpx">
          <wd-button
            type="primary"
            block
            :loading="submitting"
            :disabled="submitting"
            @click="handleSubmit"
          >
            减签
          </wd-button>
        </view>
      </wd-form>
    </view>
  </view>
</template>

<script lang="ts" setup>
import type { FormInstance } from 'wot-design-uni/components/wd-form/types'
import { computed, onMounted, reactive, ref } from 'vue'
import { useToast } from 'wot-design-uni'
import { signDeleteTask } from '@/api/bpm/task'
import { navigateBackPlus } from '@/utils'

const props = defineProps<{
  processInstanceId: string
  taskId: string
  children?: string // JSON 字符串格式的子任务数据
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
const taskOptions = ref<any[]>([])
const formData = reactive({
  deleteSignTaskId: '',
  reason: '',
})
const formRules = {
  deleteSignTaskId: [
    { required: true, message: '减签人员不能为空' },
  ],
  reason: [
    { required: true, message: '审批意见不能为空' },
  ],
}
const formRef = ref<FormInstance>()

/** 返回上一页 */
function handleBack() {
  navigateBackPlus(`/pages-bpm/processInstance/detail/index?id=${processInstanceId.value}&taskId=${taskId.value}`)
}

/** 初始化校验 */
// TODO @jason：最好放在 onMounted 里？或者其他地方，有个入口方法。
if (!props.taskId || !props.processInstanceId) {
  toast.show('参数错误')
}

/** 获取减签人员标签 */
function getDeleteSignUserLabel(task: any): string {
  const deptName = task?.assigneeUser?.deptName || task?.ownerUser?.deptName
  const nickname = task?.assigneeUser?.nickname || task?.ownerUser?.nickname
  return `${nickname} ( 所属部门：${deptName} )`
}

/** 获取可减签的任务列表 */
async function loadDeleteSignTaskList() {
  try {
    let childTasks = []
    // TODO @jason：这里应该是从 props 里获取？
    // 从 URL 参数中获取子任务数据
    if (props.children) {
      try {
        childTasks = JSON.parse(decodeURIComponent(props.children))
      } catch (parseError) {
        console.error('[delete-sign] 解析子任务数据失败:', parseError)
      }
    }
    // 提示没有子任务数据
    if (childTasks.length === 0) {
      toast.show('没有可减签的任务')
      return
    }

    taskOptions.value = childTasks.map(task => ({
      id: task.id,
      label: getDeleteSignUserLabel(task),
    }))
  } catch (error) {
    // TODO @jason：这里不用 try catch 哈
    console.error('[delete-sign] 获取可减签任务失败:', error)
    toast.error('获取可减签任务失败')
  }
}

/** 提交操作 */
async function handleSubmit() {
  if (submitting.value) {
    return
  }
  const { valid } = await formRef.value!.validate()
  if (!valid) {
    return
  }

  // TODO @jason：submitting 改成 formLoading 哇？统一代码风格哈；
  submitting.value = true
  try {
    // TODO @jason：这里是不是不用判断 result 哈？
    const result = await signDeleteTask({
      id: formData.deleteSignTaskId,
      reason: formData.reason,
    })
    if (result) {
      toast.success('减签成功')
      setTimeout(() => {
        uni.redirectTo({
          url: `/pages-bpm/processInstance/detail/index?id=${processInstanceId.value}&taskId=${taskId.value}`,
        })
      }, 500)
    }
  } catch (error) {
    // TODO @jason：可以不用这里的 catch 哈？
    console.error('[delete-sign] 减签失败:', error)
    toast.error('减签失败')
  } finally {
    submitting.value = false
  }
}

/** 页面加载时，获取可减签任务列表 */
onMounted(() => {
  loadDeleteSignTaskList()
})
</script>
