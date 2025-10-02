<script setup>
import { ref, onMounted } from 'vue'
import apiService from '@/services/api_service'
import storageService from '@/services/storage_service'

// 响应式数据
const key = ref(null)
const item = ref(null)
const loading = ref(true)
const error = ref(null)
const showQRDialog = ref(false)
const isFromCache = ref(false)

// 挪车通知相关
const notifyPhone = ref('')
const notifyLoading = ref(false)
const notifySuccess = ref(false)
const notifyError = ref(null)

/**
 * 获取状态对应的颜色
 * @param {string} status - 物品状态
 * @returns {string} 对应的颜色名称
 */
const getStatusColor = (status) => {
  const statusMap = {
    ok: "success",
    lost: "error",
    default: "grey"
  }
  return statusMap[status] || statusMap.default
}

/**
 * 获取状态对应的文本
 * @param {string} status - 物品状态
 * @returns {string} 对应的状态文本
 */
const getStatusText = (status) => {
  const statusMap = {
    ok: "正常",
    lost: "丢失",
    default: "未知"
  }
  return statusMap[status] || statusMap.default
}

/**
 * 格式化日期显示
 * @param {string} dateStr - 日期字符串
 * @returns {string} 格式化的日期文本
 */
const formatDate = (dateStr) => {
  if (!dateStr) return "未知时间"
  
  try {
    const date = new Date(dateStr)
    return new Intl.DateTimeFormat('zh-CN', {
      year: 'numeric',
      month: '2-digit',
      day: '2-digit',
      hour: '2-digit',
      minute: '2-digit'
    }).format(date)
  } catch (e) {
    return dateStr
  }
}

/**
 * 从缓存加载数据并获取最新数据
 * 优先显示本地缓存的数据,同时从API获取最新数据
 */
const loadFromCacheAndFetch = async () => {
  try {
    // 直接获取最新数据，不优先使用缓存
    loading.value = true
    await fetchItemDetails(false)
  } catch (err) {
    console.error("Error loading item data:", err)
    
    // 如果网络请求失败，尝试使用缓存数据作为降级方案
    const cachedItem = storageService.getItemFromCache(key.value)
    if (cachedItem) {
      console.log('Network request failed, using cached data as fallback:', key.value)
      item.value = cachedItem
      isFromCache.value = true
      error.value = "无法获取最新数据，显示的是缓存内容（可能已过期）"
    } else {
      error.value = "获取物品信息失败，且无缓存数据可用"
    }
    
    loading.value = false
  }
}

/**
 * 获取物品详情数据
 * @param {boolean} isBackground - 是否在后台获取数据（已显示缓存数据）
 */
const fetchItemDetails = async (isBackground = false) => {
  try {
    if (!isBackground) {
      loading.value = true
    }
    
    // 强制从API获取最新数据，不使用缓存
    const data = await apiService.getObject(key.value, false)
    
    // 更新本地缓存
    storageService.saveItemToCache(key.value, data)
    
    // 更新界面数据
    item.value = data
    isFromCache.value = false
    
  } catch (err) {
    console.error("Error fetching item details:", err)
    
    // 如果是后台请求且已有缓存数据显示，则不显示错误
    if (!isBackground || !item.value) {
      // 重新抛出错误，让上层处理
      throw err
    }
  } finally {
    loading.value = false
  }
}

/**
 * 发送挪车通知
 */
const sendMoveCarNotification = async () => {
  if (!item.value || !item.value.id) {
    return
  }
  
  notifyLoading.value = true
  notifyError.value = null
  notifySuccess.value = false
  
  try {
    const result = await apiService.notifyMoveCar(
      item.value.id, 
      notifyPhone.value || null
    )
    
    if (result.success) {
      notifySuccess.value = true
      notifyPhone.value = '' // 清空输入框
      
      // 3秒后自动隐藏成功提示
      setTimeout(() => {
        notifySuccess.value = false
      }, 3000)
    } else {
      notifyError.value = result.error || '发送挪车通知失败'
    }
  } catch (err) {
    console.error("Error sending move car notification:", err)
    notifyError.value = err.message || '发送挪车通知时出现错误'
  } finally {
    notifyLoading.value = false
  }
}

// 组件挂载时执行
onMounted(() => {
  // 从URL获取物品的key
  const urlParams = new URLSearchParams(window.location.search)
  key.value = urlParams.get('key')
  
  if (key.value) {
    // 尝试先从缓存获取数据
    loadFromCacheAndFetch()
  } else {
    loading.value = false
    error.value = "缺少物品标识，无法获取信息"
  }
})
</script>

<template>
  <v-container class="found-container">
    <v-row justify="center">
      <v-col cols="12" md="8" lg="6">
        <!-- 加载状态显示 -->
        <v-skeleton-loader
          v-if="loading && !item"
          type="card, article"
          class="mx-auto"
        ></v-skeleton-loader>

        <!-- 错误信息显示 -->
        <v-alert
          v-if="error"
          type="error"
          variant="tonal"
          closable
          class="mb-4"
        >
          {{ error }}
        </v-alert>

        <!-- 物品详情卡片 -->
        <v-card v-if="item" class="found-item-card">
          <v-card-title class="text-h4 d-flex align-center">
            <v-icon :icon="item.icon || 'mdi-tag'" size="large" class="mr-2"></v-icon>
            {{ item.name || '未命名物品' }}
          </v-card-title>

          <v-card-subtitle class="text-subtitle-1">
            ID: {{ item.id || '未知' }}
            <v-chip v-if="isFromCache" size="x-small" color="grey" class="ml-2" variant="outlined">缓存</v-chip>
          </v-card-subtitle>

          <v-divider class="my-2"></v-divider>

          <v-card-text>
            <div class="d-flex align-center mb-4">
              <v-chip
                :color="getStatusColor(item.status)"
                class="mr-2"
                variant="elevated"
              >
                {{ getStatusText(item.status) }}
              </v-chip>
              <span class="text-caption">最后更新: {{ formatDate(item.updated_at) }}</span>
            </div>

            <!-- 物品描述或者丢失上下文 - 只在丢失状态下显示 -->
            <div v-if="item.status === 'lost' && item.lost_description" class="mb-4">
              <v-alert variant="tonal" color="error" class="context-box">
                <div class="text-subtitle-1 font-weight-bold mb-2">丢失信息</div>
                <div>{{ item.lost_description }}</div>
              </v-alert>
            </div>

            <!-- 挪车通知 - 只在物品类型为车辆时显示 -->
            <v-card v-if="item.type === 'car'" variant="outlined" class="mb-4">
              <v-card-title class="text-h6">
                <v-icon icon="mdi-car" class="mr-2"></v-icon>
                挪车通知
              </v-card-title>
              <v-card-text>
                <v-text-field
                  v-model="notifyPhone"
                  label="联系电话（可选）"
                  placeholder="输入您的联系电话"
                  variant="outlined"
                  density="comfortable"
                  class="mb-3"
                ></v-text-field>
                <v-btn
                  :loading="notifyLoading"
                  :disabled="notifyLoading"
                  color="primary"
                  block
                  @click="sendMoveCarNotification"
                >
                  <v-icon icon="mdi-bell-ring" class="mr-2"></v-icon>
                  发送挪车通知
                </v-btn>
                <v-alert
                  v-if="notifySuccess"
                  type="success"
                  variant="tonal"
                  class="mt-3"
                >
                  通知已成功发送！车主将收到挪车提醒。
                </v-alert>
                <v-alert
                  v-if="notifyError"
                  type="error"
                  variant="tonal"
                  class="mt-3"
                  closable
                  @click:close="notifyError = null"
                >
                  {{ notifyError }}
                </v-alert>
              </v-card-text>
            </v-card>

            <!-- 创建者/主人信息 -->
            <v-card variant="outlined" class="mb-4">
              <v-card-title class="text-h6">
                <v-icon icon="mdi-account" class="mr-2"></v-icon>
                联系信息
              </v-card-title>
              <v-card-text>
                <v-list>
                  <v-list-item v-if="item.phone">
                    <template v-slot:prepend>
                      <v-icon icon="mdi-phone"></v-icon>
                    </template>
                    <v-list-item-title>{{ item.phone }}</v-list-item-title>
                  </v-list-item>
                </v-list>
              </v-card-text>
            </v-card>

            <!-- 仍在加载更新数据时显示 -->
            <div v-if="loading && isFromCache" class="text-center my-3">
              <v-progress-circular
                indeterminate 
                size="24"
                width="2"
                color="primary"
                class="mr-2"
              ></v-progress-circular>
              <span class="text-caption">正在获取最新数据...</span>
            </div>
          </v-card-text>

          <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn
              color="primary"
              variant="tonal"
              prepend-icon="mdi-phone"
              v-if="item.phone"
              :href="`tel:${item.phone}`"
            >
              联系失主
            </v-btn>
          </v-card-actions>
        </v-card>

        <!-- 未找到物品信息时显示 -->
        <v-alert
          v-if="!loading && !error && !item"
          type="warning"
          variant="tonal"
          class="mt-4"
        >
          未找到物品信息，请检查链接是否正确。
        </v-alert>
      </v-col>
    </v-row>
    
  </v-container>
</template>

<style scoped>
.found-container {
  padding-top: 20px;
  padding-bottom: 40px;
}

.found-item-card {
  border-radius: 12px;
  overflow: hidden;
}

.context-box {
  border-left: 4px solid;
}
</style>