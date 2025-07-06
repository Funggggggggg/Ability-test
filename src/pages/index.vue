<template>
  <v-container>
    <div class="ma-8">
      <div class="text-h4 text-lg-h3 d-flex justify-center"> 良知新聞行事曆 </div>
      <v-row>
        <v-col>
          <div> 昨日新聞測試 {{ formatDate(yesterday) }}</div>
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
    <!-- 行事曆區 -->
    <v-row class="">
      <v-col>
        <v-sheet
          class="d-flex justify-center"
          height="54"
          tile
          variant="flat"
        >
          <v-select
            v-model="type"
            class="ma-2"
            density="compact"
            hide-details
            :items="types"
            label="View Mode"
            variant="outlined"
          />
        </v-sheet>
        <v-sheet height="600">
          <v-calendar
            ref="calendar"
            v-model="value"
            :events="events"
            type="month"
            :view-mode="type"
            :weekdays="weekday"
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

  // 日期與分類資料
  const type = ref('month')
  const types = ref(['month', 'week', 'day'])
  const weekday = ref([0, 1, 2, 3, 4, 5, 6])
  const value = ref([new Date()])
  const events = ref([])
  const colors = ref(['blue', 'indigo', 'deep-purple', 'cyan', 'green', 'orange', 'grey darken-1'])

  const category = ref('全部') // 分類變數，預設為'全部'
  const categories = ref(['全部', '生技醫藥', '資訊安全', '國際金融', '數位資產', '人工智慧']) // 分類選項

  // 新增：昨天的日期測試
  const yesterday = computed(() => {
    const date = new Date()
    date.setDate(date.getDate() - 1)
    return date
  })

  // API 設定
  const apiUrl = 'https://eunomics.net/get_post'

  //  分類對應的 keyword_group_id
  const categoryMapping = {
    // 全部: 0, =>「全部」的對應，API 不支援
    生技醫藥: 1,
    資訊安全: 2,
    國際金融: 3,
    數位資產: 4,
    人工智慧: 5,
  }

  // 取得日期適配器
  const adapter = useDate()

  // API 相關方法
  const fetchNewsData = async (keywordGroupId, date) => {
    //  async 函式 非同步呼叫 API
    try {
      // 準備 API 參數
      const params = {
        date: formatDate(date),
      }
      if (keywordGroupId) {
        params.keyword_group_id = keywordGroupId
      }

      console.log(`哈哈呼叫 API: ${apiUrl}?keyword_group_id=${keywordGroupId}&date=${formatDate(date)}`)

      // 發送請求
      const response = await axios.get(apiUrl, { params })

      console.log('API 回應:', response.data)
      return response.data || []
    } catch (error) {
      if (error.code === 'ECONNABORTED') {
        console.error('API 超時:', error.message)
      } else if (error.response) {
        console.error('API 錯誤:', error.response.status, error.response.data)
      } else {
        console.error('API 錯誤:', error.message)
      }
      return []
    }
  }

  // 日期格式化用 formatDate 函式轉成 YYYY-MM-DD。
  const formatDate = date => {
    return date.toISOString().split('T')[0]
  }

  // 從 API 資料生成事件
  const generateEventsFromAPI = async ({ start, end }) => {
    console.log('=== 開始生成事件 ===')
    const eventList = []
    const startDate = new Date(start)
    const endDate = new Date(end)

    console.log('日期範圍:', formatDate(startDate), '到', formatDate(endDate))
    console.log('當前分類:', category.value)

    // 遍歷日期範圍
    for (let date = new Date(startDate); date <= endDate; date.setDate(date.getDate() + 1)) {
      const currentDate = new Date(date)

      // 根據當前選擇的分類取得資料
      const keywordGroupId = categoryMapping[category.value]// 修正：定義 keywordGroupId
      const newsData = await fetchNewsData(keywordGroupId, currentDate)

      // 將 API 資料轉換為事件
      if (newsData && newsData.length > 0) {
        for (const [index, news] of newsData.entries()) {
          eventList.push({
            id: news.id || `${formatDate(currentDate)}-${index}`,
            title: news.title || category.value,
            start: currentDate,
            end: currentDate,
            color: colors.value[index % colors.value.length],
            allDay: true,
            url: news.url || news.link, // 新聞連結
            content: news.content || news.description,
            category: category.value,
          })
        }
      }
    }

    events.value = eventList
  }

  // 處理事件點擊
  const handleEventClick = event => {
    if (event.url) {
      // 開啟新分頁跳轉到新聞內容
      window.open(event.url, '_blank')
    } else {
      // 如果沒有 URL，可以顯示詳細資訊或其他處理
      console.log('事件詳情:', event)
    }
  }

  // 監聽分類變化，重新載入資料
  watch(category, async () => {
    await generateEventsFromAPI({
      start: adapter.startOfDay(adapter.startOfMonth(value.value[0] || new Date())),
      end: adapter.endOfDay(adapter.endOfMonth(value.value[0] || new Date())),
    })
  })

  // 監聽日期變化，重新載入資料
  watch(value, async newValue => {
    if (newValue && newValue[0]) {
      await generateEventsFromAPI({
        start: adapter.startOfDay(adapter.startOfMonth(newValue[0])),
        end: adapter.endOfDay(adapter.endOfMonth(newValue[0])),
      })
    }
  })

  // 生命週期鉤子
  onMounted(async () => {
    await generateEventsFromAPI({
      start: adapter.startOfDay(adapter.startOfMonth(new Date())),
      end: adapter.endOfDay(adapter.endOfMonth(new Date())),
    })
  })

</script>

<style scoped>

</style>

<route lang="
      yaml"
    >
      meta:
      login: false
      admin: false
      title: '首頁'
</route>
