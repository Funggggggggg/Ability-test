<template>
  <v-container>
    <div class="ma-8">
      <div class="text-h4 text-lg-h3 d-flex justify-center"> 良知新聞行事曆 </div>

      <!-- <v-col> -->
      <!-- <v-card> -->
      <!-- <template #title> -->
      <!-- <span class="font-weight-black d-flex justify-center"> -->
      <!-- {{ formatDate(yesterday) }} -->
      <!-- 新聞焦點 -->
      <!-- {{ dailyNews.length > 0
                  ? dailyNews[0].title.slice(0, 30) + (dailyNews[0].title.length > 30 ? '...' : '')
                  : (category === '全部' ? '請選擇分類': `載入${category}新聞中`) }} -->
      <!-- </span> -->
      <!-- </template> -->

      <!-- <v-card-text class="bg-surface-light pt-4">
              <div v-if="dailyNews.length > 0">
                <div v-for="news in dailyNews" :key="news.id" class="mb-2">
                  <p class="text-h6 mb-2"><strong>{{ news.title }}</strong></p>
                  <p class="text-caption mb-2">
                    {{ news.content && news.content.length > 100
                      ? news.content.slice(0, 100) + '...'
                      : news.content }}
                  </p>
                  <div class="text-xs text-grey-600 mt-1 d-flex flex-row-reverse">
                    <span>發布日期: {{ news.post_date }}</span>
                  </div>
                </div>
              </div>
              <div v-else>
                沒有找到昨日新聞
              </div>
            </v-card-text> -->
      <!-- </v-card> -->
      <!-- </v-col> -->

      <!-- 新聞分類篩選區 -->
      <v-row class="mt-4">
        <v-col class="d-flex justify-center mt-10" cols="12">
          <v-card>
            <v-card-title>
              <span> 新聞分類篩選</span>
              <v-spacer />
              <v-btn
                size="small"
                variant="text"
                @click="toggleAllCategories"
              >
                {{ allSelected ? '取消全選' : '全選' }}
              </v-btn>
            </v-card-title>
            <v-card-text>
              <v-row>
                <v-col
                  v-for="cate in availableCategories"
                  :key="cate"
                  cols="12"
                  md="4"
                  sm="6"
                >
                  <v-checkbox
                    v-model="selectedCategories"
                    color="primary"
                    hide-details
                    :label="cate"
                    :value="cate"
                  /></v-col>
              </v-row>
            </v-card-text>
          </v-card>
        </v-col>
      </v-row>
    </div>

    <!-- 除錯：顯示 events 陣列狀態 -->
    <!-- <v-row class="mt-4">
      <v-col>
        <v-card>
          <v-card-title>除錯資訊</v-card-title>
          <v-card-text>
            <div>當前分類: {{ category }}</div>
            <div>events 長度: {{ events.length }}</div>
            <div>events 內容: {{ events.slice(0, 3) }}</div>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row> -->

    <!-- 行事曆區-->
    <v-row class="mt-4">
      <v-col>
        <v-sheet class="calenderStyle">
          <v-calendar
            ref="calendar"
            v-model="value"
            :events="events"
            @click:event="handleEventClick"
          />
          <!-- :event-height="40" -->
          <!-- :view-mode="type" -->
        </v-sheet>
      </v-col>
    </v-row>
  </v-container>
</template>
<script setup>
  import axios from 'axios'
  import { computed, onMounted, ref, watch } from 'vue'
  import { useDate } from 'vuetify'

  // ==================
  // 1. 基本變數定義
  // ==================

  // 分類
  const availableCategories = ref(['生技醫藥', '資訊安全', '國際金融', '數位資產', '人工智慧'])
  const selectedCategories = ref(['生技醫藥']) // 預設只選擇一個分類

  // 昨日新聞變數測試用
  const newsSubtitle = ref('載入中...')
  const dailyNews = ref([])

  // 行事曆變數
  const value = ref([new Date()])
  const events = ref([])
  const colors = ref(['blue', 'indigo', 'deep-purple', 'cyan', 'green', 'orange', 'grey darken-1'])
  const adapter = useDate()

  // ==================
  // 2. 計算屬性
  // ==================

  const allSelected = computed(() =>
    selectedCategories.value.length === availableCategories.value.length,
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
  const formatDate = date => {
    return date.toISOString().split('T')[0]
  }

  // ===================
  // 5. API 相關函數
  // ===================

  const fetchSingleDayNewsTitle = async (keywordGroupId, date) => {
    if (!keywordGroupId) return []

    try {
      const params = {
        date: formatDate(date),
        keyword_group_id: keywordGroupId,
      }
      console.log(`📰 載入 ${formatDate(date)} 的 ${keywordGroupId} 新聞標題`)

      const response = await axios.get(apiUrl, { params })

      if (!response.data) return []

      // 只保留必要的欄位，減少資料量
      const newsData = Array.isArray(response.data) ? response.data : [response.data]
      return newsData.map(news => ({
        id: news.id,
        title: news.title,
        post_date: news.post_date,
      // 不載入 content，節省載入時間
      }))
    } catch (error) {
      console.error(`❌ ${formatDate(date)} API 錯誤:`, error.message)
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
          post_date: news.post_date,
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

  // 按月載入行事曆資料 (避免過載)
  // 🔥 修正：只保留一個正確的 generateMonthlyEvents
  const generateMonthlyEvents = async ({ start, end }) => {
    console.log('📅 超級優化載入行事曆')

    if (selectedCategories.value.length === 0) {
      events.value = []
      return
    }

    const startDate = new Date(start)
    const endDate = new Date(end)

    // 建立日期陣列
    const dates = []
    const maxDays = 30

    for (let date = new Date(startDate), dayCount = 0; date <= endDate && dayCount < maxDays; date.setDate(date.getDate() + 1), dayCount++) {
      dates.push(new Date(date))
    }

    console.log(`📅 準備並行載入 ${dates.length} 天的資料`)

    // 並行載入所有日期
    const results = await Promise.allSettled(
      dates.map(date => fetchMultipleCategoryNewsTitle(date)),
    )

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
              postDate: news.post_date,
              needsContent: true, // 🔥 修正2：統一加入此屬性
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
    selectedCategories.value = allSelected.value ? [] : [...availableCategories.value]
  }

  // 點擊時才載入完整內容
  const handleEventClick = async event => {
    console.log('📰 點擊事件:', event)

    if (event.needsContent) {
      console.log('📰 載入完整新聞內容...')

      try {
        // 載入完整新聞內容
        const keywordGroupId = categoryMapping[event.category]
        const fullNews = await fetchSingleDayNews(keywordGroupId, event.start)

        // 找到對應的新聞
        const newsDetail = fullNews.find(news => news.id === event.id)

        if (newsDetail && newsDetail.content) {
          alert(`新聞標題: ${event.title}\n分類: ${event.category}\n發布日期: ${event.postDate}\n\n內容: ${newsDetail.content}`)
        } else {
          alert(`新聞標題: ${event.title}\n分類: ${event.category}\n發布日期: ${event.postDate}\n\n(無詳細內容)`)
        }
      } catch (error) {
        console.error('載入新聞詳情失敗:', error)
        alert(`新聞標題: ${event.title}\n分類: ${event.category}\n發布日期: ${event.postDate}\n\n(載入內容時發生錯誤)`)
      }
    } else {
      alert(`新聞標題: ${event.title}\n分類: ${event.category}\n發布日期: ${event.postDate}`)
    }
  }

  // ====================
  // 8. 監聽器
  // ====================

  // 載入昨日新聞

  // 昨天的日期
  // const yesterday = computed(() => {
  //   const date = new Date()
  //   date.setDate(date.getDate() - 1)
  //   return date
  // })

  // 監聽分類變化
  watch(selectedCategories, async () => {
    console.log('🔄 分類變更為:', selectedCategories.value)

    if (selectedCategories.value.length > 0 && value.value && value.value[0]) {
      await generateMonthlyEvents({
        start: adapter.startOfDay(adapter.startOfMonth(value.value[0])),
        end: adapter.endOfDay(adapter.endOfMonth(value.value[0])),
      })
    } else {
      events.value = []
    }
  }, { deep: true })

  // 監聽日期變化
  watch(value, async newValue => {
    console.log('📅 日期變更為:', newValue)

    if (newValue && newValue[0] && selectedCategories.value.length > 0) {
      await generateMonthlyEvents({
        start: adapter.startOfDay(adapter.startOfMonth(newValue[0])),
        end: adapter.endOfDay(adapter.endOfMonth(newValue[0])),
      })
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
    // 載入當月行事曆
    // await generateMonthlyEvents({
    //   start: adapter.startOfDay(adapter.startOfMonth(new Date())),
    //   end: adapter.endOfDay(adapter.endOfMonth(new Date())),
    // })
  })

</script>

<style>
  .calenderStyle {
    height: 600px;
    max-width: 100%;
    overflow: auto;
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
