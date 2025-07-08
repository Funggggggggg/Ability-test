<template>
  <v-container>
    <div class="ma-8">
      <div class="text-h4 text-lg-h3 d-flex justify-center"> 良知新聞行事曆 </div>
      <v-row class="mt-4">
        <v-col>
          <v-card
            class="mx-auto"
            :subtitle="newsSubtitle"
            width="400"
          >
            <!-- <template #title> -->
            <!-- <span class="font-weight-black d-flex justify-center"> -->
            <!-- {{ formatDate(yesterday) }} -->
            <!-- 新聞焦點 -->
            <!-- {{ dailyNews.length > 0
                  ? dailyNews[0].title.slice(0, 30) + (dailyNews[0].title.length > 30 ? '...' : '')
                  : (category === '全部' ? '請選擇分類': `載入${category}新聞中`) }} -->
            <!-- </span> -->
            <!-- </template> -->

            <v-card-text class="bg-surface-light pt-4">
              <div v-if="dailyNews.length > 0">
                <div v-for="news in dailyNews" :key="news.id" class="mb-2">
                  <P class="text-h6 mb-2"><strong>{{ news.title }}</strong></P>
                  <p class="text-caption mb-2">
                    {{ news.content && news.content.length > 100
                      ? news.content.slice(0, 100) + '...'
                      : news.content }}
                  </p>
                  <div class="text-xs text-grey-600 mt-1 d-flex flex-row-reverse">
                    <span>發布日期: {{ news.post_date }}</span>
                    <!-- <span class="ml-2">ID: {{ news.id }}</span> -->
                    <!-- <span class="ml-2">分類ID: {{ news.keyword_group_id }}</span> -->
                  </div>
                </div>
              </div>
              <div v-else>
                沒有找到昨日新聞
              </div>
            </v-card-text>
          </v-card>
        </v-col>

        <!-- 分類區 -->
        <v-col class="d-flex justify-center mt-10" cols="12">
          <v-btn
            v-for="cate in categories"
            :key="cate"
            class="mx-2 text-body-2 ms-0"
            :class="{ 'v-btn--active': category === cate }"
            style="max-width: 500px; line-height: 5px;"
            variant="tonal"
            @click="category = cate"
          >
            {{ cate }}
          </v-btn>
        </v-col>
      </v-row>
    </div>

    <!-- 行事曆區-->
    <v-row class="">
      <v-col>
        <v-sheet height="600">
          <v-calendar
            ref="calendar"
            v-model="value"
            :events="events"
            :view-mode="type"
            @click:event="handleEventClick"
          />
        </v-sheet>
      </v-col>
    </v-row>

  </v-container>
</template>

<script setup>
  import axios from 'axios'
  import { computed, onMounted, ref, watch } from 'vue'
  import { useDate } from 'vuetify'

  // 🔥 測試用基本變數
  const category = ref('全部')
  const categories = ref(['全部', '生技醫藥', '資訊安全', '國際金融', '數位資產', '人工智慧'])

  // 測試用：昨日新聞變數
  const newsSubtitle = ref('載入中...')
  const dailyNews = ref([])

  // 昨天的日期
  const yesterday = computed(() => {
    const date = new Date()
    date.setDate(date.getDate() - 1)
    return date
  })
  // 行事曆變數
  const type = ref('month')
  const value = ref([new Date()])
  const events = ref([])
  const colors = ref(['blue', 'indigo', 'deep-purple', 'cyan', 'green', 'orange', 'grey darken-1'])
  const adapter = useDate()

  // API 設定
  const apiUrl = '/api/get_post'

  const categoryMapping = {
    生技醫藥: 1,
    資訊安全: 2,
    國際金融: 3,
    數位資產: 4,
    人工智慧: 5,
  }

  // 日期格式化
  const formatDate = date => {
    return date.toISOString().split('T')[0]
  }

  // 單次 API 呼叫
  const fetchSingleDayNews = async (keywordGroupId, date) => {
    if (!keywordGroupId) return []

    try {
      const params = {
        date: formatDate(date),
        keyword_group_id: keywordGroupId,
      }

      console.log(`📰 載入 ${formatDate(date)} 的 ${keywordGroupId} 新聞`)

      const response = await axios.get(apiUrl, { params })

      // 檢查 API 回應格式
      if (!response.data) return []
      return Array.isArray(response.data) ? response.data : [response.data]
    } catch (error) {
      console.error(`❌ ${formatDate(date)} API 錯誤:`, error.message)
      return []
    }
  }

  // 載入昨日新聞
  const loadYesterdayNews = async (categoryName = '生技醫藥') => {
    const keywordGroupId = categoryMapping[categoryName]

    if (!keywordGroupId) {
      newsSubtitle.value = ''
      dailyNews.value = []
      return
    }

    try {
      newsSubtitle.value = ''
      const newsData = await fetchSingleNews(keywordGroupId, yesterday.value)

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

  // 監聽分類變化
  watch(category, async newCategory => {
    console.log('🔄 分類變更為:', newCategory)

    if (newCategory === '全部') {
      newsSubtitle.value = '請選擇特定分類'
      dailyNews.value = []
    } else {
      await loadYesterdayNews(newCategory)
    }
  })

  // 頁面載入
  onMounted(async () => {
    console.log('🚀 開始測試 API')
    await loadYesterdayNews('生技醫藥') // 預設載入生技醫藥
  })

  // 新增：按月載入行事曆資料 (避免過載)
  const generateMonthlyEvents = async ({ start, end }) => {
    console.log('開始載入月度行事曆')

    // 如果沒選擇特定分類，不載入資料
    if (category.value === '全部') {
      events.value = []
      console.log('⚠️ 請選擇特定分類')
      return
    }

    const eventList = []
    const startDate = new Date(start)
    const endDate = new Date(end)
    const keywordGroupId = categoryMapping[category.value]

    console.log(`載入範圍: ${formatDate(startDate)} 到 ${formatDate(endDate)}`)
    console.log(`當前分類: ${category.value} (ID: ${keywordGroupId})`)

    // 🔥 重要：限制載入天數，避免過載
    const maxDays = 30 // 每次最多載入 30 天
    let dayCount = 0

    for (let date = new Date(startDate); date <= endDate; date.setDate(date.getDate() + 1)) {
      const currentDate = new Date(date)

      // 🔥 使用測試成功的 API 呼叫邏輯
      const newsData = await fetchSingleDayNews(keywordGroupId, currentDate)

      if (newsData && newsData.length > 0) {
        // 為每天的每則新聞建立事件
        for (const [index, news] of newsData.entries()) {
          eventList.push({
            id: news.id || `${formatDate(currentDate)}-${index}`,
            title: news.title || `${category.value} 新聞`,
            start: currentDate,
            end: currentDate,
            color: colors.value[index % colors.value.length],
            allDay: true,
            content: news.content || news.description,
            category: category.value,
            postDate: news.post_date,
          })
        }
        console.log(`✅ ${formatDate(currentDate)}: 找到 ${newsData.length} 則新聞`)
      } else {
        console.log(`⚪ ${formatDate(currentDate)}: 無新聞資料`)
      }
    }
    dayCount++

    // 🔥 加入延遲避免 API 過載
    await new Promise(resolve => setTimeout(resolve, 200)) // 延遲 200ms
  }

  // 事件點擊處理
  const handleEventClick = event => {
    console.log('📰 點擊事件:', event)
    // 可以在這裡顯示新聞詳情
    alert(`新聞標題: ${event.title}\n發布日期: ${event.postDate}`)
  }

  // 🔥 監聽日期變化：載入新月份的資料
  watch(value, async newCategory => {
    console.log('🔄 分類變更為:', newCategory)

    if (newCategory === '全部') {
      newsSubtitle.value = ''
      dailyNews.value = []
      events.value = [] // 清空行事曆
    } else {
      // 更新新聞卡片
      await loadYesterdayNews(newCategory[0])

      // 更新行事曆
      if (value.value && value.value[0]) {
        await generateMonthlyEvents({
          start: adapter.startOfDay(adapter.startOfMonth(newCategory[0])),
          end: adapter.endOfDay(adapter.endOfMonth(newCategory[0])),
        })
      }
    }
  })

</script>

<style scoped>
</style>

<route lang="yaml">
meta:
  login: false
  admin: false
  title: '首頁'
</route>
