<template>
  <v-container>
    <div class="ma-8">
      <div class="text-h4 text-lg-h3 d-flex justify-center"> 良知新聞行事曆 </div>
      <v-row>
        <v-col cols="12">
          <!-- 分類區 -->
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
            :attributes="attrs"
            color=""
            :events="events"
            type="month"
            :view-mode="type"
            :weekdays="weekday"
          />
        </v-sheet>
      </v-col>
    </v-row>
    <!-- 錯誤提示 -->
    <!-- <v-alert
          v-if="error"
          class="ma-4"
          type="error"
          variant="tonal"
        >
          {{ error }}
        </v-alert> -->

  </v-container>
</template>

<script setup>
  import { computed, onMounted, ref, watch } from 'vue'
  import { useDate } from 'vuetify'

  // 日期與分類資料
  const type = ref('month')
  const types = ref(['month', 'week', 'day'])
  const weekday = ref([0, 1, 2, 3, 4, 5, 6])
  const value = ref([new Date()])
  const events = ref([])
  const colors = ref(['blue', 'indigo', 'deep-purple', 'cyan', 'green', 'orange', 'grey darken-1'])
  const titles = ref(['生技醫藥', '資訊安全', '國際金融', '數位資產', '人工智慧'])

  // 取得日期適配器
  const adapter = useDate()

  // FIXME 方法 => API 抓資料
  const rnd = (a, b) => {
    return Math.floor((b - a + 1) * Math.random()) + a
  }

  // FIXME 新聞不用有時間的計算
  const getEvents = ({ start, end }) => {
    const eventList = []

    const min = start
    const max = end
    const days = (max.getTime() - min.getTime()) / 86_400_000
    const eventCount = rnd(days, days + 20)

    for (let i = 0; i < eventCount; i++) {
      const allDay = rnd(0, 3) === 0
      const firstTimestamp = rnd(min.getTime(), max.getTime())
      const first = new Date(firstTimestamp - (firstTimestamp % 900_000))
      const secondTimestamp = rnd(2, allDay ? 288 : 8) * 900_000
      const second = new Date(first.getTime() + secondTimestamp)

      eventList.push({
        title: titles.value[rnd(0, titles.value.length - 1)],
        start: first,
        end: second,
        color: colors.value[rnd(0, colors.value.length - 1)],
        allDay: !allDay,
      })
    }

    events.value = eventList
  }

  const _getEventColor = event => {
    return event.color
  }

  // 生命週期鉤子
  onMounted(() => {
    getEvents({
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
