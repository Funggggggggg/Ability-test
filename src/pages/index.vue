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
            <template #title>
              <span class="font-weight-black d-flex justify-center">昨日新聞測試 {{ formatDate(yesterday) }}</span>
            </template>

            <v-card-text class="bg-surface-light pt-4">
              <div v-if="dailyNews.length > 0">
                <div v-for="news in dailyNews" :key="news.id" class="mb-2">
                  <strong>{{ news.title }}</strong>
                  <p class="text-caption">{{ news.content }}</p>
                </div>
                <div class="text-xs text-grey-600 mt-1">
                  <span>發布日期: {{ news.post_date }}</span>
                  <span class="ml-2"> {{ news.id }}</span>
                  <span class="ml-2"> {{ news.keyword_group_id }}</span>
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

    <!-- 行事曆區 - 暫時註解掉 -->
    <!--
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
    -->
  </v-container>
</template>

<script setup>
  import axios from 'axios'
  import { computed, onMounted, ref, watch } from 'vue'
  import { useDate } from 'vuetify'

  // 🔥 測試用基本變數
  const category = ref('全部')
  const categories = ref(['全部', '生技醫藥', '資訊安全', '國際金融', '數位資產', '人工智慧'])

  // 🔥 測試用：昨日新聞變數
  const newsSubtitle = ref('載入中...')
  const dailyNews = ref([])

  // 🔥 昨天的日期
  const yesterday = computed(() => {
    const date = new Date()
    date.setDate(date.getDate() - 1)
    return date
  })

  // 🔥 API 設定
  const apiUrl = 'https://eunomics.net/get_post'

  const categoryMapping = {
    生技醫藥: 1,
    資訊安全: 2,
    國際金融: 3,
    數位資產: 4,
    人工智慧: 5,
  }

  // 🔥 日期格式化
  const formatDate = date => {
    return date.toISOString().split('T')[0]
  }

  // 🔥 單次 API 呼叫（僅用於昨日新聞測試）
  const fetchSingleNews = async (keywordGroupId, date) => {
    if (!keywordGroupId) return []

    try {
      const params = {
        date: formatDate(date),
        keyword_group_id: keywordGroupId,
      }

      console.log(`📰 測試 API: ${apiUrl}?keyword_group_id=${keywordGroupId}&date=${formatDate(date)}`)

      const response = await axios.get(apiUrl, { params })

      console.log('✅ API 回應:', response.data)
      return response.data || []
    } catch (error) {
      console.error('❌ API 錯誤:', error.message)
      return []
    }
  }

  // 🔥 載入昨日新聞
  const loadYesterdayNews = async (categoryName = '生技醫藥') => {
    const keywordGroupId = categoryMapping[categoryName]

    if (!keywordGroupId) {
      newsSubtitle.value = '請選擇有效分類'
      dailyNews.value = []
      return
    }

    try {
      newsSubtitle.value = '載入中...'
      const newsData = await fetchSingleNews(keywordGroupId, yesterday.value)

      if (newsData && newsData.length > 0) {
        dailyNews.value = newsData
        newsSubtitle.value = `✅ 找到 ${newsData.length} 筆 ${categoryName} 昨日新聞`
      } else {
        dailyNews.value = []
        newsSubtitle.value = `${categoryName} 昨日暫無新聞`
      }
    } catch {
      newsSubtitle.value = '❌ 載入失敗'
      dailyNews.value = []
    }
  }

  // 🔥 監聽分類變化
  watch(category, async newCategory => {
    console.log('🔄 分類變更為:', newCategory)

    if (newCategory === '全部') {
      newsSubtitle.value = '請選擇特定分類'
      dailyNews.value = []
    } else {
      await loadYesterdayNews(newCategory)
    }
  })

  // 🔥 頁面載入
  onMounted(async () => {
    console.log('🚀 開始測試 API')
    await loadYesterdayNews('生技醫藥') // 預設載入生技醫藥
  })

// 註解掉的程式碼區域
/*
// 以下程式碼暫時註解，避免過載問題

const type = ref('month')
const value = ref([new Date()])
const events = ref([])
const colors = ref(['blue', 'indigo', 'deep-purple', 'cyan', 'green', 'orange', 'grey darken-1'])
const adapter = useDate()

const fetchNewsData = async (keywordGroupId, date) => {
  try {
    const params = {
      date: formatDate(date),
    }
    if (keywordGroupId) {
      params.keyword_group_id = keywordGroupId
    }

    console.log(`哈哈呼叫 API: ${apiUrl}?keyword_group_id=${keywordGroupId}&date=${formatDate(date)}`)

    // const response = await axios.get(apiUrl, { params })

    console.log('API 回應:', response.data)
    return response.data || []
  } catch (error) {
    console.error('API 錯誤:', error.message)
    return []
  }
}

const generateEventsFromAPI = async ({ start, end }) => {
  console.log('=== 開始生成事件 ===')
  const eventList = []
  const startDate = new Date(start)
  const endDate = new Date(end)

  console.log('日期範圍:', formatDate(startDate), '到', formatDate(endDate))
  console.log('當前分類:', category.value)

  for (let date = new Date(startDate); date <= endDate; date.setDate(date.getDate() + 1)) {
    const currentDate = new Date(date)
    const keywordGroupId = categoryMapping[category.value]
    const newsData = await fetchNewsData(keywordGroupId, currentDate)

    if (newsData && newsData.length > 0) {
      for (const [index, news] of newsData.entries()) {
        eventList.push({
          id: news.id || `${formatDate(currentDate)}-${index}`,
          title: news.title || category.value,
          start: currentDate,
          end: currentDate,
          color: colors.value[index % colors.value.length],
          allDay: true,
          url: news.url || news.link,
          content: news.content || news.description,
          category: category.value,
        })
      }
    }
  }

  events.value = eventList
}

const handleEventClick = event => {
  if (event.url) {
    window.open(event.url, '_blank')
  } else {
    console.log('事件詳情:', event)
  }
}

watch(value, async newValue => {
  if (newValue && newValue[0]) {
    await generateEventsFromAPI({
      start: adapter.startOfDay(adapter.startOfMonth(newValue[0])),
      end: adapter.endOfDay(adapter.endOfMonth(newValue[0])),
    })
  }
})
*/

</script>

<style scoped>
</style>

<route lang="yaml">
meta:
  login: false
  admin: false
  title: '首頁'
</route>
