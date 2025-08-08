<template>
  <!-- 路線・乗車地 選択 -->
  <div class="grid">
    <div>
      <label for="routeSelect">路線選択</label>
      <select id="routeSelect" v-model="selectedRoute" @change="updateStations">
        <option value="">-- 路線を選択 --</option>
        <option v-for="route in routes" :key="route.id" :value="route.id">
          {{ route.name }}
        </option>
      </select>
    </div>
    <div>
      <label for="stationSelect">乗車地選択</label>
      <select
        id="stationSelect"
        v-model="selectedStation"
        @change="selectStation"
        :disabled="!selectedRoute"
      >
        <option value="">-- 乗車地を選択 --</option>
        <option v-for="station in availableStations" :key="station.id" :value="station.id">
          {{ station.name }}
        </option>
      </select>
    </div>
  </div>

  <!-- 乗車地・ダイヤ -->
  <div v-if="stationData && status == 'ready'" class="info-area">
    <h4>{{ stationData.station }}</h4>
    <div v-html="diaInfo"></div>
  </div>

  <!-- 出力 -->
  <div v-if="status == 'loading'">
    <p>読み込み中...</p>
  </div>

  <div v-if="status == 'error'">
    <p>サーバとの通信に失敗しました。再度選択してください。</p>
  </div>

  <div v-if="stationData && status == 'ready'">
    <TripCard
      v-for="trip in trips"
      :routeName="trip.routeName"
      :dest="trip.dest"
      :depTime="trip.depTime"
      :timer="trip.timer"
    />
  </div>

  <div v-else>
    <p>路線と乗車地を選択してください。</p>
  </div>
</template>

<script setup>
import { ref, computed, onUnmounted, onMounted, watch } from 'vue'
import TripCard from './components/TripCard.vue'

const API_URL = 'https://bustimer.orangeliner.net/apiv1.php?sta='

const routes = [{ id: 'showa', name: 'からつ・いまり号' }]

const allStations = {
  showa: [
    { id: 'tenjin', name: '🏙 天神' },
    { id: 'hakata', name: '🚅 博多' },
    { id: 'fukdom', name: '✈ 福岡空港' },
  ],
}

const selectedRoute = ref('')
const selectedStation = ref('')
const availableStations = ref([])

// localStorageから前回の選択を復元
const loadSavedSelections = () => {
  const savedRoute = localStorage.getItem('bustimer-selected-route')
  const savedStation = localStorage.getItem('bustimer-selected-station')

  if (savedRoute && routes.some((r) => r.id === savedRoute)) {
    selectedRoute.value = savedRoute
    updateStations()

    if (savedStation && availableStations.value.some((s) => s.id === savedStation)) {
      selectedStation.value = savedStation
    }
  }
}

// 選択をlocalStorageに保存
const saveSelections = () => {
  if (selectedRoute.value) {
    localStorage.setItem('bustimer-selected-route', selectedRoute.value)
  }
  if (selectedStation.value) {
    localStorage.setItem('bustimer-selected-station', selectedStation.value)
  }
}

// 路線選択時に使用可能な乗車地を更新
const updateStations = () => {
  if (selectedRoute.value && allStations[selectedRoute.value]) {
    availableStations.value = allStations[selectedRoute.value]
  } else {
    availableStations.value = []
  }
  // 路線変更時は乗車地選択をリセット
  selectedStation.value = ''
  saveSelections()
}

const status = ref(null)
const stationData = ref(null)
const trips = ref([])
const depTimes = ref([])
let timerInterval = null

const diaInfo = computed(() => {
  if (!stationData.value) return ''

  const { diaType, diaTypeName } = stationData.value
  const badgeStyles = {
    weekday: 'color: var(--pico-color-green-500);',
    saturday: 'color: var(--pico-color-blue-500);',
    holiday: 'color: var(--pico-color-red-500);',
    newyear: 'color: var(--pico-color-red-500);',
  }

  return `<mark style="${badgeStyles[diaType] || ''}">${diaTypeName} ダイヤ</mark>`
})

const clearTimer = () => {
  if (timerInterval) {
    clearInterval(timerInterval)
    timerInterval = null
  }
}

const selectStation = async () => {
  if (!selectedStation.value) return

  // 選択を保存
  saveSelections()

  status.value = 'loading'
  trips.value = []
  depTimes.value = []
  clearTimer()

  try {
    const response = await fetch(`${API_URL}${selectedStation.value}`)
    if (!response.ok) throw new Error('Network response was not ok')

    const data = await response.json()
    stationData.value = data
    processTrips(data.trips)

    timerInterval = setInterval(calculateTime, 1000)
  } catch (err) {
    status.value = 'error'
  } finally {
    status.value = 'ready'
  }
}

const processTrips = (tripsData) => {
  const newTrips = []
  const newDepTimes = []
  let hasNoData = false

  for (let i = 0; i < 5; i++) {
    if (tripsData[i]) {
      const trip = tripsData[i]
      const depTime = new Date(trip.depTime)
      const hours = depTime.getHours()
      const minutes = depTime.getMinutes().toString().padStart(2, '0')

      newTrips.push({
        routeName: trip.routeName,
        routeColor: '',
        dest: trip.dest,
        depTime: `${hours} : ${minutes} 発`,
        timer: '読み込み中...',
      })
      newDepTimes[i] = trip.depTime
    } else if (!hasNoData) {
      newTrips.push({
        dest: 'データがありません',
        depTime: '-- : --',
        timer: '',
      })
      hasNoData = true
    }
  }

  trips.value = newTrips
  depTimes.value = newDepTimes
}

const calculateTime = () => {
  const currentTime = new Date()

  depTimes.value.forEach((depTime, i) => {
    if (!depTime || !trips.value[i]) return

    const departureTime = new Date(depTime)
    const remainTime = departureTime - currentTime

    if (remainTime > 0) {
      const h = Math.floor(remainTime / (60 * 60 * 1000))
      const m = Math.floor((remainTime % (60 * 60 * 1000)) / (60 * 1000))
      const s = Math.floor((remainTime % (60 * 1000)) / 1000)

      trips.value[i].timer = h === 0 ? `あと ${m} 分 ${s} 秒` : `あと ${h} 時間 ${m} 分 ${s} 秒`
    } else {
      trips.value[i].timer = '出発済み'
    }
  })
}

// アプリ起動時に前回の選択を復元
onMounted(() => {
  loadSavedSelections()
  // 保存された乗車地がある場合は自動でデータを読み込み
  if (selectedStation.value) {
    selectStation()
  }
})

watch(selectedStation, selectStation)

onUnmounted(clearTimer)
</script>

<style scoped>
.info-area {
  text-align: center;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  flex-wrap: wrap;
  margin-bottom: 1rem;
}
.info-area h4 {
  margin: 0;
}
</style>
