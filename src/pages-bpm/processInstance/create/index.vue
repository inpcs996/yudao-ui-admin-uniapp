<template>
  <!-- TODO vben 对应的地址：/Users/yunai/Java/yudao-ui-admin-vben-v5/apps/web-antd/src/views/bpm/processInstance/create/index.vue -->
  <view class="yd-page-container">
    <!-- 顶部导航栏 -->
    <wd-navbar
      title="发起申请"
      left-arrow placeholder safe-area-inset-top fixed
      @click-left="handleBack"
    />

    <!-- 搜索框 -->
    <view class="bg-white p-24rpx">
      <wd-search
        v-model="searchName"
        placeholder="请输入流程名称"
        placeholder-left
        hide-cancel
        @search="handleSearch"
        @clear="handleSearch"
      />
    </view>

    <!-- 分类标签 -->
    <!-- TODO @AI：可以使用 https://wot-ui.cn/component/index-bar.html 组件么？ -->
    <view class="flex overflow-x-auto bg-white px-16rpx">
      <view
        v-for="(item, index) in categoryList"
        :key="item.id"
        class="relative whitespace-nowrap px-24rpx py-20rpx text-28rpx"
        :class="activeIndex === index ? 'font-bold text-[#1890ff]' : 'text-[#666]'"
        @click="switchCategory(index)"
      >
        {{ item.name }}
        <view
          v-if="activeIndex === index"
          class="absolute bottom-0 left-24rpx right-24rpx h-4rpx bg-[#1890ff]"
        />
      </view>
    </view>

    <!-- 流程定义列表 -->
    <scroll-view
      scroll-y
      class="h-[calc(100vh-280rpx)]"
      :scroll-into-view="scrollIntoView"
      scroll-with-animation
    >
      <view
        v-for="(definitions, category) in groupedDefinitions"
        :id="`category-${category}`"
        :key="category"
        class="mx-24rpx mt-24rpx"
      >
        <!-- 分类标题 -->
        <view class="mb-16rpx flex items-center justify-between">
          <text class="text-28rpx text-[#333] font-bold">{{ getCategoryName(category as string) }}</text>
          <wd-icon
            :name="expandedCategories[category as string] ? 'arrow-up' : 'arrow-down'"
            size="32rpx"
            @click="toggleCategory(category as string)"
          />
        </view>
        <!-- 流程列表 -->
        <view v-if="expandedCategories[category as string]" class="overflow-hidden rounded-16rpx bg-white">
          <view
            v-for="(item, index) in definitions"
            :key="item.id"
            class="flex items-center border-b border-[#f5f5f5] p-24rpx last:border-b-0"
            @click="handleSelect(item)"
          >
            <view
              class="mr-16rpx h-64rpx w-64rpx flex items-center justify-center rounded-12rpx"
              :style="{ backgroundColor: getIconColor(index) }"
            >
              <wd-icon :name="getIconName(index)" size="40rpx" color="#fff" />
            </view>
            <text class="text-28rpx text-[#333]">{{ item.name }}</text>
          </view>
        </view>
      </view>

      <!-- 空状态 -->
      <view v-if="Object.keys(groupedDefinitions).length === 0" class="py-100rpx">
        <wd-status-tip image="content" tip="暂无可发起的流程" />
      </view>
    </scroll-view>
  </view>
</template>

<script lang="ts" setup>
import type { Category } from '@/api/bpm/category'
import type { ProcessDefinition } from '@/api/bpm/definition'
import { onLoad } from '@dcloudio/uni-app'
import { computed, ref } from 'vue'
import { useToast } from 'wot-design-uni'
import { getCategorySimpleList } from '@/api/bpm/category'
import { getProcessDefinitionList } from '@/api/bpm/definition'
import { navigateBackPlus } from '@/utils'

// TODO @芋艿：【重新发起流程】支持通过 processInstanceId 参数重新发起已有流程
// 对应 vben 第 44-60 行：从路由获取 processInstanceId，查询流程实例后自动选中对应流程定义并填充表单数据

// TODO @芋艿：【流程表单填写】选择流程后跳转到表单填写页面
// 对应 vben form.vue 全部：包含以下子功能：
// - 表单渲染 (form-create)：vben form.vue 第 145-152 行
// - 审批流程预览时间线：vben form.vue 第 153-162 行
// - 流程图预览 (BPMN/简易)：vben form.vue 第 163-178 行
// - 发起人自选审批人：vben form.vue 第 30-32, 85-95 行
// - 表单字段权限控制 (读/写/隐藏)：vben form.vue 第 119-131 行
// - 业务表单跳转 (formCustomCreatePath)：vben form.vue 第 79-85 行
// - 表单值变化重新预测审批节点：vben form.vue 第 87-102 行
// - 提交流程实例：vben form.vue 第 56-76 行

definePage({
  style: {
    navigationBarTitleText: '',
    navigationStyle: 'custom',
  },
})

const toast = useToast()

const searchName = ref('')
const activeIndex = ref(0)
const scrollIntoView = ref('')
const categoryList = ref<Category[]>([])
const definitionList = ref<ProcessDefinition[]>([])
const expandedCategories = ref<Record<string, boolean>>({})

/** 图标配置 */
// TODO @芋艿：【流程定义图标】支持显示流程定义的自定义图标 definition.icon
// 对应 vben 第 175-189 行：优先显示 definition.icon，无图标时显示流程名称前两个字
// TODO @AI：优化下，图标使用 vben 对应的逻辑；
const iconConfig = [
  { icon: 'warning', color: '#D98469' },
  { icon: 'heart', color: '#7BC67C' },
  { icon: 'cart', color: '#4A7FEB' },
  { icon: 'home', color: '#4A7FEB' },
  { icon: 'location', color: '#4A9DEB' },
]

/** 过滤后的流程定义 */
const filteredDefinitions = computed(() => {
  if (!searchName.value.trim()) {
    return definitionList.value
  }
  return definitionList.value.filter(item =>
    item.name.toLowerCase().includes(searchName.value.toLowerCase()),
  )
})

/** 按分类分组的流程定义 */
const groupedDefinitions = computed<Record<string, ProcessDefinition[]>>(() => {
  const grouped: Record<string, ProcessDefinition[]> = {}
  filteredDefinitions.value.forEach((item) => {
    if (!item.category)
      return
    if (!grouped[item.category])
      grouped[item.category] = []
    grouped[item.category].push(item)
  })
  // 按 categoryList 顺序排序
  const ordered: Record<string, ProcessDefinition[]> = {}
  categoryList.value.forEach((cat) => {
    if (grouped[cat.code])
      ordered[cat.code] = grouped[cat.code]
  })
  return ordered
})

/** 返回上一页 */
function handleBack() {
  navigateBackPlus('/pages/bpm/index')
}

/** 搜索 */
function handleSearch() {
  // 搜索时展开所有分类
  categoryList.value.forEach((cat) => {
    expandedCategories.value[cat.code] = true
  })
}

/** 切换分类 */
// TODO @AI：目前有个 bug；滚动到为止后，选中的 category 不会变；
function switchCategory(index: number) {
  activeIndex.value = index
  const category = categoryList.value[index]
  if (category) {
    expandedCategories.value[category.code] = true
    // 滚动到对应分类
    scrollIntoView.value = ''
    setTimeout(() => {
      scrollIntoView.value = `category-${category.code}`
    }, 50)
  }
}

/** 切换分类展开/收起 */
function toggleCategory(code: string) {
  expandedCategories.value[code] = !expandedCategories.value[code]
}

/** 获取分类名称 */
function getCategoryName(code: string) {
  return categoryList.value.find(item => item.code === code)?.name || code
}

/** 获取图标名称 */
function getIconName(index: number) {
  return iconConfig[index % iconConfig.length].icon
}

/** 获取图标颜色 */
function getIconColor(index: number) {
  return iconConfig[index % iconConfig.length].color
}

/** 选择流程定义 */
function handleSelect(item: ProcessDefinition) {
  // TODO @芋艿：【流程描述提示】显示流程描述 Tooltip
  // 对应 vben 第 190-195 行：鼠标悬停显示 definition.description

  // TODO @芋艿：【搜索高亮动画】搜索匹配时卡片有弹跳动画效果
  // 对应 vben 第 169-173 行：搜索时添加 animate-bounce-once 动画类

  // TODO @芋艿：跳转到流程表单页面
  toast.show(`选择了: ${item.name}`)
}

/** 加载分类列表 */
async function loadCategoryList() {
  categoryList.value = await getCategorySimpleList()
  // 默认展开所有分类
  categoryList.value.forEach((cat) => {
    expandedCategories.value[cat.code] = true
  })
}

/** 加载流程定义列表 */
async function loadDefinitionList() {
  definitionList.value = await getProcessDefinitionList({ suspensionState: 1 })
}

/** 初始化 */
onLoad(async () => {
  await Promise.all([loadCategoryList(), loadDefinitionList()])
})
</script>
