<template>
  <v-container>
    <div class="ma-8">
      <div class="text-h4 text-lg-h3 d-flex justify-center"> 良知新聞行事曆 </div>

      <!-- 新聞分類篩選區 -->
      <v-row class="mt-4">
        <v-col class="d-flex justify-center mt-10" cols="12">
          <v-card>
            <v-card-title class="mb-4 text-h5">
              <div> 新聞分類 </div>
            </v-card-title>
            <v-card-text>
              <v-row>
                <v-col
                  v-for="cate in availableCategories"
                  :key="cate"
                  class="CategoriesStyle"
                  cols="12"
                  lg="2"
                  md="4"
                  sm="6"
                >
                  <v-checkbox
                    v-if="cate === '全選/取消'"
                    class="toggle-all-checkbox"
                    color="primary"
                    hide-details
                    :label="allSelected ? '取消全選' : '全選'"
                    :model-value="allSelected"
                    readonly
                    @click="toggleAllCategories"
                  />
                  <v-checkbox
                    v-else
                    v-model="selectedCategories"
                    color="primary"
                    hide-details
                    :label="cate"
                    :value="cate"
                  />
                </v-col>
              </v-row>
            </v-card-text>
          </v-card>
        </v-col>
      </v-row>
    </div>

    <!-- 行事曆區-->
    <v-row class="mt-4">
      <v-col>
        <v-sheet class="calenderStyle">
          <!-- 🔥 新增：載入覆蓋層 -->
          <v-overlay
            v-model="isLoading"
            class="d-flex align-center justify-center"
            contained
            style="background-color: rgba(255, 255, 255, 0.8);"
          >
            <span v-if="isLoading">載入中...</span>
          </v-overlay>
          <v-calendar
            ref="calendar"
            v-model="value"
            :events="events"
            @click:event="handleEventClick"
          />
        </v-sheet>
      </v-col>
    </v-row>

    <!-- 彈跳視窗 -->
    <v-dialog v-model="dialog" max-width="500">
      <v-card>
        <v-card-title class="d-flex justify-space-between align-center">
          <span>{{ selectedEvent?.category }}</span>
          <v-btn icon="mdi-close" size="small" variant="text" @click="dialog = false" />
        </v-card-title>

        <v-card-text class="pa-4">
          <!-- 標題 -->
          <h3 class="mb-3">{{ selectedEvent?.title }}</h3>

          <!-- 內容區域 -->
          <div class="mb-3">
            <h4 class="text-subtitle-1 mb-2" />

            <!-- 載入中狀態 -->
            <div v-if="!selectedEvent?.content" class="text-center py-4">
              <v-progress-circular
                class="mb-2"
                color="primary"
                indeterminate
                size="24"
              />
              <p class="text-body-2 text-grey-600">載入內容中...</p>
            </div>

            <!-- 顯示內容 -->
            <div v-else class="content-display">
              <p class="text-body-1">{{ selectedEvent.content }}</p>
            </div>
          </div>

          <!-- 發布日期 -->
          <p class="d-flex justify-end">發布日期：{{ selectedEvent?.postDate }}</p>

        </v-card-text>

        <v-card-actions>
          <v-spacer />
          <v-btn variant="text" @click="dialog = false">關閉</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-container>
</template>
<script setup>
  import axios from 'axios'
  import { computed, onMounted, ref, watch } from 'vue'
  import { useDate } from 'vuetify'

  // ==================
  // 1. 基本變數定義
  // ==================

  // 載入控制變數（載入速度優化）
  let currentLoadingId = 0
  let debounceTimer = null

  // 分類
  const categories = ref(['生技醫藥', '資訊安全', '國際金融', '數位資產', '人工智慧'])
  const selectedCategories = ref(['生技醫藥']) // 預設只選擇一個分類
  // 顯示用的項目清單（包含按鈕）
  const availableCategories = computed(() => ['全選/取消', ...categories.value])

  // 昨日新聞變數測試用
  const newsSubtitle = ref('載入中...')
  const dailyNews = ref([])

  // 行事曆變數
  const value = ref([new Date()])
  const events = ref([])
  const colors = ref(['blue', 'indigo', 'deep-purple', 'cyan', 'green', 'orange', 'grey darken-1'])
  const adapter = useDate()

  // 彈跳視窗變數
  const dialog = ref(false)
  const selectedEvent = ref(null)
  // const newsContent = ref('')
  // const loadingContent = ref(false)

  // ==================
  // 2. 計算屬性
  // ==================

  const allSelected = computed(() =>
    selectedCategories.value.length === categories.value.length,
  )
  // ==================
  // 3. API 設定
  // ==================

  const apiUrl = '/api/get_post'
  const categoryMapping = {
    生技醫藥: 1,
    資訊安全: 2,
    國際金融: 3,
    數位資產: 4,
    人工智慧: 5,
  }

  // ===================
  // 4. 工具函數
  // ===================

  // 日期格式化
  // 使用 toISOString() 會轉換為 UTC 時間 => 要加 8 小時
  // 台灣時間：2025-07-01 08:00 // UTC 時間：2025-07-01 00:00
  const formatDate = date => {
    // 🔥 類型檢查和錯誤處理
    if (!date) {
      console.error('❌ formatDate: date 是 null 或 undefined')
      return new Date().toISOString().split('T')[0]
    }

    // 🔥 如果已經是字串，直接返回（避免重複格式化）
    if (typeof date === 'string') {
      console.warn('⚠️ formatDate: 接收到字串，直接返回', date)
      return date
    }

    // 🔥 確保是 Date 物件
    const dateObj = date instanceof Date ? date : new Date(date)

    if (Number.isNaN(dateObj.getTime())) {
      console.error('❌ formatDate: 無效的日期', date)
      return new Date().toISOString().split('T')[0]
    }

    const year = date.getFullYear()
    const month = String(date.getMonth() + 1).padStart(2, '0')
    const day = String(date.getDate()).padStart(2, '0')
    return `${year}-${month}-${day}`
  }

  // ===================
  // 5. API 相關函數
  // ===================

  // 在 API 層面就過濾未來日期
  const fetchSingleDayNewsTitle = async (keywordGroupId, date) => {
    if (!keywordGroupId) return []

    // 檢查是否為未來日期
    const today = new Date()
    const requestDate = new Date(date)

    today.setHours(23, 59, 59, 999) // 設定為今天的 23:59:59
    requestDate.setHours(0, 0, 0, 0)

    if (requestDate > today) {
      console.log(`⚠️ 跳過未來日期 API 請求: ${formatDate(date)}`)
      return []
    }

    console.log(`📰 載入 ${formatDate(date)} 的 ${keywordGroupId} 新聞標題 (有效日期)`)

    try {
      const params = {
        date: formatDate(date),
        keyword_group_id: keywordGroupId,
      }

      const response = await axios.get(apiUrl, { params })

      if (!response.data) return []

      // 只保留必要的欄位，減少資料量
      const newsData = Array.isArray(response.data) ? response.data : [response.data]
      return newsData.map(news => ({
        id: news.id,
        title: news.title,
        // post_date: news.post_date, // => 移除減少資料量和載入時間
        // 不載入 content，節省載入時間
      }))
    } catch (error) {
      if (error.response?.status === 404) {
        console.log(`📅 ${formatDate(date)} 沒有 ${keywordGroupId} 分類的新聞`)
      } else {
        console.error(`❌ ${formatDate(date)} API 錯誤:`, error.message)
      }
      return []
    }
  }

  // 🔥 載入多分類標題的優化版本
  const fetchMultipleCategoryNewsTitle = async date => {
    const allNews = []

    for (const categoryName of selectedCategories.value) {
      const keywordGroupId = categoryMapping[categoryName]
      if (keywordGroupId) {
        const newsData = await fetchSingleDayNewsTitle(keywordGroupId, date)

        // 🔥 只處理標題相關資料
        const categorizedNews = newsData.map(news => ({
          id: news.id,
          title: news.title,
          // post_date: news.post_date,
          categoryName: categoryName,
          displayTitle: `[${categoryName}] ${news.title}`,
        }))

        allNews.push(...categorizedNews)
      }
    }

    return allNews
  }

  // 載入單日單分類新聞 => 詳情頁使用
  const fetchSingleDayNews = async (keywordGroupId, date) => {
    if (!keywordGroupId) return []

    try {
      const params = {
        date: formatDate(date),
        keyword_group_id: keywordGroupId,
      }

      console.log(`📰 載入 ${formatDate(date)} 的 ${keywordGroupId} 新聞`)

      const response = await axios.get(apiUrl, { params })

      if (!response.data) return []
      return Array.isArray(response.data) ? response.data : [response.data]
    } catch (error) {
      console.error(`❌ ${formatDate(date)} API 錯誤:`, error.message)
      return []
    }
  }

  // ===================
  // 6. 主要功能函數
  // ===================

  // 按月載入行事曆資料
  const isLoading = ref(false)// 載入狀態

  const generateMonthlyEvents = async ({ start, end }) => {
    const loadingId = ++currentLoadingId // 速度優化
    console.log(`🆔 開始載入 ID: ${loadingId}`)
    isLoading.value = true // 開始載入

    try {
      console.log('📅 載入行事曆（智能月份過濾）')

      // 立即清除舊資料，提升使用者體驗
      events.value = []

      if (selectedCategories.value.length === 0) {
        return
      }

      const startDate = new Date(start)
      const endDate = new Date(end)
      const today = new Date() // 確保不載入未來日期
      today.setHours(23, 59, 59, 999) // 今天結束時間

      console.log(`📅 今天日期: ${formatDate(today)}`)

      // 🔥 取得當前顯示的月份
      const displayMonth = value.value[0] ? value.value[0].getMonth() : new Date().getMonth()
      const displayYear = value.value[0] ? value.value[0].getFullYear() : new Date().getFullYear()
      console.log(`📅 載入月份: ${displayYear}-${displayMonth + 1}`)

      // 建立日期陣列
      const dates = []
      const maxDays = 35

      for (let date = new Date(startDate), dayCount = 0;
           date <= endDate && dayCount < maxDays;
           date.setDate(date.getDate() + 1), dayCount++) {
             // 重要修正：建立純日期物件，避免時區問題
             const currentDate = new Date(date.getFullYear(), date.getMonth(), date.getDate())

             // 條件：1. 屬於顯示的月份 2. 不是未來日期 3. API 有資料的日期
             const isDisplayMonth = currentDate.getMonth() === displayMonth
               && currentDate.getFullYear() === displayYear
             const isNotFuture = currentDate <= today

             if (isDisplayMonth && isNotFuture) {
               dates.push(currentDate)
               console.log(`📅 加入: ${formatDate(currentDate)}`) // ← 直接用 currentDate
             } else if (!isDisplayMonth) {
               console.log(`⚠️ 跳過其他月份: ${formatDate(currentDate)}`) // ← 直接用 currentDate
             } else if (!isNotFuture) {
               console.log(`⚠️ 跳過未來日期: ${formatDate(currentDate)} (今天: ${formatDate(today)})`) // ← 直接用 currentDate 和 today
             }
           }

      console.log(`📅 有效日期數量: ${dates.length}`)

      // 🔥 顯示所有有效日期
      console.log(`📅 有效日期列表: ${dates.map(d => formatDate(d)).join(', ')}`)

      // 如果沒有有效日期，直接返回
      if (dates.length === 0) {
        console.log('📅 本月沒有有效日期，不載入任何資料')
        return
      }

      // 🔥 加入載入進度提示
      console.log(`🚀 開始並行載入 ${dates.length} 天的新聞...`)

      // 並行載入所有日期
      const results = await Promise.allSettled(
        dates.map(date => fetchMultipleCategoryNewsTitle(date)),
      )

      // 檢查是否還是最新請求 => 速度優化
      if (loadingId !== currentLoadingId) {
        console.log(`⚠️ 載入 ${loadingId} 已過期，捨棄結果`)
        return
      }

      // 處理結果
      const eventList = []
      for (const [index, result] of results.entries()) {
        if (result.status === 'fulfilled') {
          const allDayNews = result.value
          const currentDate = dates[index]

          if (allDayNews.length > 0) {
            for (const [newsIndex, news] of allDayNews.entries()) {
              eventList.push({
                id: news.id || `${formatDate(currentDate)}-${newsIndex}`,
                title: news.displayTitle || news.title,
                start: currentDate,
                end: currentDate,
                color: colors.value[categoryMapping[news.categoryName] % colors.value.length],
                allDay: true,
                category: news.categoryName,
                postDate: formatDate(currentDate), // 使用已知的日期，不另外取
                // postDate: news.post_date,
                needsContent: true,
              })
            }
            console.log(`✅ ${formatDate(currentDate)}: 找到 ${allDayNews.length} 則新聞`)
          }
        } else {
          console.error(`❌ ${formatDate(dates[index])} 載入失敗:`, result.reason)
        }
      }

      events.value = eventList
      console.log(`🎯 並行載入完成: ${eventList.length} 個事件`)

      if (loadingId === currentLoadingId) {
        events.value = eventList
        console.log(`🎯 載入完成: ${eventList.length} 事件 (ID: ${loadingId})`)
      }
    } catch (error) {
      console.error('❌ 載入失敗:', error)
      if (loadingId === currentLoadingId) {
        events.value = []
      }
    } finally {
      if (loadingId === currentLoadingId) {
        isLoading.value = false // 關閉載入狀態
      }
    }
  }

  const loadYesterdayNews = async (categoryName = '生技醫藥') => {
    const keywordGroupId = categoryMapping[categoryName]

    if (!keywordGroupId) return

    try {
      const yesterday = new Date()
      yesterday.setDate(yesterday.getDate() - 1)

      const newsData = await fetchSingleDayNews(keywordGroupId, yesterday)

      if (newsData && newsData.length > 0) {
        dailyNews.value = newsData
        newsSubtitle.value = ``
      } else {
        dailyNews.value = []
        newsSubtitle.value = `${categoryName} 昨日暫無新聞`
      }
    } catch {
      newsSubtitle.value = ''
      dailyNews.value = []
    }
  }

  // =====================
  // 7. UI 互動函數
  // =====================

  // 全選/取消全選功能
  const toggleAllCategories = () => {
    selectedCategories.value = allSelected.value ? [] : [...categories.value]
  }

  // FIXME 點擊時才載入完整內容 => 原生 DOM 事件轉日曆事件 => 解構賦值
  const handleEventClick = async (_nativeEvent, eventWrapper) => {
    console.log('📰 點擊事件:', eventWrapper?.event)

    const event = eventWrapper?.event

    if (!event || !event.id) {
      console.error('❌ 無效的日曆事件')
      return
    }

    console.log('✅ 成功取得事件:', event)
    // console.log('✅ 事件ID:', event.id)
    // console.log('✅ 事件標題:', event.title)
    // console.log('✅ 事件分類:', event.category)

    selectedEvent.value = event
    dialog.value = true

    // 🔥 動態載入內容（如果還沒有的話）
    if (event.content) {
      console.log('✅ 已有內容，直接顯示')
    } else {
      console.log('🔍 開始動態載入新聞內容...')

      try {
        const keywordGroupId = categoryMapping[event.category]
        const fullNews = await fetchSingleDayNews(keywordGroupId, event.start)
        const newsDetail = fullNews.find(news => news.id === event.id)

        // 🔥 直接更新 selectedEvent 的內容
        if (newsDetail?.content) {
          selectedEvent.value = {
            ...selectedEvent.value,
            content: newsDetail.content,
          }
          console.log('✅ 內容載入成功')
        } else {
          selectedEvent.value = {
            ...selectedEvent.value,
            content: '此新聞暫無詳細內容',
          }
          console.log('⚠️ 沒有找到新聞內容')
        }
      } catch (error) {
        console.error('❌ 載入內容失敗:', error)
        selectedEvent.value = {
          ...selectedEvent.value,
          content: '載入內容時發生錯誤，請稍後再試',
        }
      }
    }
  }

  // 監聽分類變化
  watch(selectedCategories, async () => {
    console.log('🔄 分類變更為:', selectedCategories.value)

    // 立即清除舊資料
    events.value = []
    isLoading.value = true

    // 清除之前的防抖計時器
    if (debounceTimer) {
      clearTimeout(debounceTimer)
    }

    // 🔥 防抖：300ms 後才真正載入
    debounceTimer = setTimeout(async () => {
      if (selectedCategories.value.length > 0 && value.value && value.value[0]) {
        await generateMonthlyEvents({
          start: adapter.startOfDay(adapter.startOfMonth(value.value[0])),
          end: adapter.endOfDay(adapter.endOfMonth(value.value[0])),
        })
      } else {
        // 🔥 沒有選擇分類時，清除載入狀態
        isLoading.value = false
        console.log('📅 沒有選擇分類，清除事件')
      }
    }, 300) // 300ms 防抖
  }, { deep: true })

  // 監聽日期變化，清除快取
  watch(value, async newValue => {
    console.log('📅 日期變更為:', newValue)

    // 月份變更時清除快取和舊資料
    const newMonth = newValue?.[0]?.getMonth()
    const newYear = newValue?.[0]?.getFullYear()

    if (newMonth !== undefined && newYear !== undefined) {
      console.log(`📅 切換到 ${newYear}-${newMonth + 1} 月`)
      events.value = [] // 立即清除舊資料

      if (selectedCategories.value.length > 0) {
        await generateMonthlyEvents({
          start: adapter.startOfDay(adapter.startOfMonth(newValue[0])),
          end: adapter.endOfDay(adapter.endOfMonth(newValue[0])),
        })
      }
    }
  })

  // ====================
  // 9. 頁面載入
  // ====================

  onMounted(async () => {
    console.log('🚀 開始載入頁面')

    // 載入昨日新聞測試
    await loadYesterdayNews('生技醫藥')

    console.log('📅 onMounted - 準備載入行事曆')
    console.log('📅 selectedCategories.value:', selectedCategories.value)
    console.log('📅 value.value:', value.value)

    if (selectedCategories.value.length > 0) {
      console.log('📅 onMounted - 開始載入行事曆')
      await generateMonthlyEvents({
        start: adapter.startOfDay(adapter.startOfMonth(new Date())),
        end: adapter.endOfDay(adapter.endOfMonth(new Date())),
      })
    }
  })

</script>

<style>
  .calenderStyle {
    max-width: 100%;
    overflow: visible;
  }
  .CategoriesStyle{
    white-space: nowrap !important;
    min-width: 140px; /* 🔥 最小寬度 */
    min-height: 48px; /* 🔥 最小高度 */
    /* border: 1px solid #ccc; */
  }
</style>

<style scoped>
</style>

<route lang="yaml">
meta:
  login: false
  admin: false
  title: '首頁'
</route>
