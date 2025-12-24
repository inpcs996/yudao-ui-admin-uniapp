<template>
  <view class="yd-page-container">
    <!-- 顶部导航栏 -->
    <wd-navbar
      title="请假列表"
      left-arrow placeholder safe-area-inset-top fixed
      @click-left="handleBack"
    />

    <!-- 搜索组件 -->
    <LeaveSearchForm @search="handleSearch" @reset="handleReset" />

    <view class="bpm-list">
      <!-- 请假列表 -->
      <view
        v-for="item in list"
        :key="item.id"
        class="bpm-card"
        @click="handleDetail(item)"
      >
        <view class="bpm-card-content">
          <view class="bpm-card-header">
            <view class="bpm-card-title">
              <dict-tag :type="DICT_TYPE.BPM_OA_LEAVE_TYPE" :value="item.type" />
            </view>
            <dict-tag :type="DICT_TYPE.BPM_PROCESS_INSTANCE_STATUS" :value="item.status" />
          </view>
          <view class="bpm-summary">
            <view class="bpm-summary-item">
              <text class="text-[#999]">开始时间：</text>
              <text>{{ formatDateTime(item.startTime) }}</text>
            </view>
            <view class="bpm-summary-item">
              <text class="text-[#999]">结束时间：</text>
              <text>{{ formatDateTime(item.endTime) }}</text>
            </view>
            <view class="bpm-summary-item">
              <text class="text-[#999]">请假原因：</text>
              <text>{{ item.reason }}</text>
            </view>
          </view>
          <view class="bpm-card-info">
            <view class="bpm-user">
              <view class="bpm-avatar">
                {{ userNickname?.[0] }}
              </view>
              <text class="bpm-nickname">{{ userNickname }}</text>
            </view>
            <text class="bpm-time">{{ formatDateTime(item.createTime) }}</text>
          </view>
        </view>
        <!-- 操作按钮 -->
        <view class="bpm-actions">
          <view class="bpm-action-btn" @click.stop="handleDetail(item)">
            <wd-icon name="eye-on" size="32rpx" />
            <text class="ml-8rpx">详情</text>
          </view>
          <view class="bpm-action-btn" @click.stop="handleProgress(item)">
            <wd-icon name="flow" size="32rpx" />
            <text class="ml-8rpx">审批进度</text>
          </view>
          <view
            v-if="item.status === BpmProcessInstanceStatus.RUNNING"
            class="bpm-action-btn text-[#ff4d4f]!"
            @click.stop="handleCancel(item)"
          >
            <wd-icon name="close" size="32rpx" color="#ff4d4f" />
            <text class="ml-8rpx">取消</text>
          </view>
        </view>
      </view>

      <!-- 加载更多 -->
      <view v-if="loadMoreState !== 'loading' && list.length === 0" class="bpm-empty">
        <wd-status-tip image="content" tip="暂无请假记录" />
      </view>
      <wd-loadmore
        v-if="list.length > 0"
        :state="loadMoreState"
        @reload="loadMore"
      />

      <!-- 新增按钮 -->
      <view
        class="fixed bottom-100rpx right-32rpx z-10 h-100rpx w-100rpx flex items-center justify-center rounded-full bg-[#1890ff] shadow-lg"
        @click="handleCreate"
      >
        <wd-icon name="add" size="24px" color="#fff" />
      </view>
    </view>
  </view>
</template>

<script lang="ts" setup>
import type { Leave } from '@/api/bpm/oa/leave'
import type { LoadMoreState } from '@/http/types'
import { onReachBottom } from '@dcloudio/uni-app'
import { computed, onMounted, ref } from 'vue'
import { useMessage } from 'wot-design-uni'
import { getLeavePage } from '@/api/bpm/oa/leave'
import { cancelProcessInstanceByStartUser } from '@/api/bpm/processInstance'
import { useUserStore } from '@/store'
import { navigateBackPlus } from '@/utils'
import { DICT_TYPE } from '@/utils/constants'
import { formatDateTime } from '@/utils/date'
import LeaveSearchForm from './components/search-form.vue'
import '@/pages/bpm/styles/index.scss'

/** 流程实例状态枚举 */
const BpmProcessInstanceStatus = {
  RUNNING: 1, // 审批中
  APPROVE: 2, // 审批通过
  REJECT: 3, // 审批不通过
  CANCEL: 4, // 已取消
}

definePage({
  style: {
    navigationBarTitleText: '',
    navigationStyle: 'custom',
  },
})

const userStore = useUserStore()
const message = useMessage()
const userNickname = computed(() => userStore.userInfo?.nickname || '')

const total = ref(0)
const list = ref<Leave[]>([])
const loadMoreState = ref<LoadMoreState>('loading')

const queryParams = ref({
  pageNo: 1,
  pageSize: 10,
})

/** 返回上一页 */
function handleBack() {
  navigateBackPlus('/pages/bpm/index')
}

/** 查询列表 */
async function getList() {
  loadMoreState.value = 'loading'
  try {
    const data = await getLeavePage(queryParams.value)
    list.value = [...list.value, ...data.list]
    total.value = data.total
    loadMoreState.value = list.value.length >= total.value ? 'finished' : 'loading'
  } catch {
    queryParams.value.pageNo = queryParams.value.pageNo > 1 ? queryParams.value.pageNo - 1 : 1
    loadMoreState.value = 'error'
  }
}

/** 加载更多 */
function loadMore() {
  if (loadMoreState.value === 'finished') {
    return
  }
  queryParams.value.pageNo++
  getList()
}

/** 搜索 */
function handleSearch(data?: Record<string, any>) {
  queryParams.value = {
    ...data,
    pageNo: 1,
    pageSize: queryParams.value.pageSize,
  }
  list.value = []
  getList()
}

/** 重置 */
function handleReset() {
  handleSearch()
}

/** 查看详情 */
function handleDetail(item: Leave) {
  uni.navigateTo({ url: `/pages-bpm/oa/leave/detail/index?id=${item.id}` })
}

/** 审批进度 */
function handleProgress(item: Leave) {
  uni.navigateTo({ url: `/pages-bpm/processInstance/detail/index?id=${item.processInstanceId}` })
}

/** 取消请假 */
function handleCancel(item: Leave) {
  message.confirm({
    title: '取消流程',
    msg: '确定要取消该请假申请吗？',
  }).then(async ({ action }) => {
    if (action !== 'confirm') {
      return
    }
    // TODO：原始 vben 版本支持输入取消原因，uniapp 的 message.confirm 不支持输入框
    // 参考：yudao-ui-admin-vben-v5/apps/web-antd/src/views/bpm/oa/leave/index.vue 第 35-60 行
    try {
      await cancelProcessInstanceByStartUser(String(item.id), '用户取消')
      uni.showToast({ title: '取消成功', icon: 'success' })
      handleSearch()
    } catch (error) {
      console.error('[leave] 取消失败:', error)
    }
  })
}

/** 发起请假 */
function handleCreate() {
  uni.navigateTo({ url: '/pages-bpm/oa/leave/create/index' })
}

/** 触底加载更多 */
onReachBottom(() => {
  loadMore()
})

/** 初始化 */
onMounted(() => {
  getList()
})
</script>
