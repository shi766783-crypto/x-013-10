<script setup>
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import { daysFromToday, normalRanges, todayISO } from '../data'

const props = defineProps({ records: { type: Array, default: () => [] }, type: { type: String, default: 'weight' } })
const canvas = ref(null)
const normalRange = computed(() => normalRanges[props.type])

const rangeOptions = [
  { id: '7d', label: '近7天', days: 7 },
  { id: '30d', label: '近30天', days: 30 },
  { id: 'all', label: '全部', days: 0 },
]
const range = ref('7d')

const filteredRecords = computed(() => {
  if (range.value === 'all') return props.records
  const days = rangeOptions.find((item) => item.id === range.value).days
  return props.records.filter((item) => {
    const offset = daysFromToday(item.date)
    return offset <= 0 && offset > -days
  })
})

const emptyText = computed(() => {
  if (!props.records.length) return '还没有记录，添加一次健康数据吧'
  const option = rangeOptions.find((item) => item.id === range.value)
  return `${option.label}没有记录，试试其他时间范围`
})

const dayTime = (iso) => new Date(`${iso}T00:00:00`).getTime()
function addDays(time, days) {
  const date = new Date(time)
  date.setDate(date.getDate() + days)
  return date.getTime()
}

// 目标视图：横轴时间范围、纵轴数值域和数据点都由它推导，切换范围时在两个视图之间插值
const targetView = computed(() => {
  const list = filteredRecords.value
  const today = dayTime(todayISO())
  let domainStart
  let domainEnd
  if (range.value === 'all') {
    if (list.length) {
      const times = list.map((item) => dayTime(item.date))
      domainStart = Math.min(...times)
      domainEnd = Math.max(...times)
    } else {
      domainStart = addDays(today, -6)
      domainEnd = today
    }
    if (domainStart === domainEnd) {
      domainStart = addDays(domainStart, -3)
      domainEnd = addDays(domainEnd, 3)
    }
  } else {
    const days = rangeOptions.find((item) => item.id === range.value).days
    domainEnd = today
    domainStart = addDays(today, -(days - 1))
  }
  const values = list.map((item) => Number(item.value))
  const padding = (normalRange.value.max - normalRange.value.min) * 0.2
  return {
    domainStart,
    domainEnd,
    yMin: Math.min(...values, normalRange.value.min) - padding,
    yMax: Math.max(...values, normalRange.value.max) + padding,
    points: list
      .map((item) => ({ id: item.id, time: dayTime(item.date), value: Number(item.value), alpha: 1 }))
      .sort((a, b) => a.time - b.time),
  }
})

let currentView = null
let fromView = null
let toView = null
let animStart = 0
let rafId = 0
const DURATION = 360

const lerp = (a, b, t) => a + (b - a) * t

function interpolateView(from, to, t) {
  const fromPoints = new Map(from.points.map((point) => [point.id, point]))
  const toPoints = new Map(to.points.map((point) => [point.id, point]))
  const points = []
  to.points.forEach((point) => {
    const start = fromPoints.get(point.id) || { ...point, alpha: 0 }
    points.push({ id: point.id, time: lerp(start.time, point.time, t), value: lerp(start.value, point.value, t), alpha: lerp(start.alpha, point.alpha, t) })
  })
  from.points.forEach((point) => {
    if (!toPoints.has(point.id)) points.push({ id: point.id, time: point.time, value: point.value, alpha: lerp(point.alpha, 0, t) })
  })
  points.sort((a, b) => a.time - b.time)
  return {
    domainStart: lerp(from.domainStart, to.domainStart, t),
    domainEnd: lerp(from.domainEnd, to.domainEnd, t),
    yMin: lerp(from.yMin, to.yMin, t),
    yMax: lerp(from.yMax, to.yMax, t),
    points,
  }
}

function tickFrame(now) {
  const progress = Math.min(1, (now - animStart) / DURATION)
  const eased = 1 - Math.pow(1 - progress, 3)
  currentView = interpolateView(fromView, toView, eased)
  draw()
  if (progress < 1) rafId = requestAnimationFrame(tickFrame)
  else currentView = toView
}

function startTransition(to) {
  if (!currentView) {
    currentView = to
    draw()
    return
  }
  fromView = currentView
  toView = to
  cancelAnimationFrame(rafId)
  animStart = performance.now()
  rafId = requestAnimationFrame(tickFrame)
}

function makeTicks(start, end, width) {
  const spanDays = Math.max(1, Math.round((end - start) / 86400000))
  const maxTicks = Math.max(2, Math.min(6, Math.floor(width / 64)))
  const step = Math.max(1, Math.ceil(spanDays / (maxTicks - 1)))
  const days = []
  for (let d = 0; d <= spanDays; d += step) days.push(d)
  const last = days[days.length - 1]
  if (last < spanDays) {
    if (spanDays - last >= step * 0.6) days.push(spanDays)
    else days[days.length - 1] = spanDays
  }
  return days.map((d) => addDays(start, d))
}

function tickLabel(time, domainEnd) {
  const date = new Date(time)
  const label = `${date.getMonth() + 1}/${date.getDate()}`
  return date.getFullYear() === new Date(domainEnd).getFullYear() ? label : `${date.getFullYear()}/${label}`
}

function draw() {
  const element = canvas.value
  if (!element || !currentView) return
  const rect = element.getBoundingClientRect()
  if (!rect.width || !rect.height) return
  const ratio = window.devicePixelRatio || 1
  element.width = rect.width * ratio
  element.height = rect.height * ratio
  const ctx = element.getContext('2d')
  ctx.scale(ratio, ratio)
  const width = rect.width
  const height = rect.height
  const view = currentView
  const padX = 14
  const padTop = 8
  const padBottom = 18
  const plotWidth = width - padX * 2
  const plotHeight = height - padTop - padBottom
  const x = (time) => padX + ((time - view.domainStart) / (view.domainEnd - view.domainStart || 1)) * plotWidth
  const y = (value) => padTop + (1 - (value - view.yMin) / (view.yMax - view.yMin || 1)) * plotHeight
  ctx.clearRect(0, 0, width, height)
  ctx.strokeStyle = '#e7ebe4'
  ctx.lineWidth = 1
  ;[0.25, 0.5, 0.75].forEach((step) => {
    ctx.beginPath(); ctx.moveTo(0, padTop + plotHeight * step); ctx.lineTo(width, padTop + plotHeight * step); ctx.stroke()
  })
  ctx.fillStyle = '#f0f7dd'
  ctx.fillRect(0, Math.max(0, y(normalRange.value.max)), width, Math.max(0, y(normalRange.value.min) - y(normalRange.value.max)))
  ctx.font = '10px "DM Sans", "Noto Sans SC", sans-serif'
  ctx.fillStyle = '#a4ada0'
  ctx.textAlign = 'center'
  makeTicks(view.domainStart, view.domainEnd, width).forEach((tick) => {
    const tx = Math.min(Math.max(x(tick), 20), width - 20)
    ctx.fillText(tickLabel(tick, view.domainEnd), tx, height - 5)
  })
  ctx.textAlign = 'left'
  // 只有一条记录时不连线，用虚线标出数值位置，避免撑满横轴的假趋势
  if (view.points.length === 1) {
    const point = view.points[0]
    ctx.save()
    ctx.globalAlpha = point.alpha
    ctx.setLineDash([3, 4])
    ctx.strokeStyle = '#dde4d2'
    ctx.beginPath(); ctx.moveTo(padX, y(point.value)); ctx.lineTo(width - padX, y(point.value)); ctx.stroke()
    ctx.restore()
  }
  // 折线按段绘制，透明度跟随端点，切换范围时数据点进出场更平滑
  if (view.points.length >= 2) {
    ctx.lineWidth = 2.5
    ctx.strokeStyle = '#b5cf3e'
    ctx.lineJoin = 'round'
    ctx.lineCap = 'round'
    for (let index = 1; index < view.points.length; index += 1) {
      const prev = view.points[index - 1]
      const next = view.points[index]
      ctx.globalAlpha = Math.min(prev.alpha, next.alpha)
      ctx.beginPath(); ctx.moveTo(x(prev.time), y(prev.value)); ctx.lineTo(x(next.time), y(next.value)); ctx.stroke()
    }
    ctx.globalAlpha = 1
  }
  view.points.forEach((point) => {
    ctx.globalAlpha = point.alpha
    ctx.beginPath(); ctx.arc(x(point.time), y(point.value), 4, 0, Math.PI * 2); ctx.fillStyle = '#ffffff'; ctx.fill(); ctx.strokeStyle = '#87a52f'; ctx.lineWidth = 2; ctx.stroke()
  })
  ctx.globalAlpha = 1
  ctx.font = '11px Inter, sans-serif'
  ctx.fillStyle = '#8b9686'
  ctx.fillText(`${normalRange.value.max}`, 6, Math.max(12, y(normalRange.value.max) - 6))
  ctx.fillText(`${normalRange.value.min}`, 6, Math.min(height - padBottom - 2, y(normalRange.value.min) + 13))
}

watch(targetView, (to) => startTransition(to))
onMounted(() => {
  currentView = targetView.value
  draw()
  window.addEventListener('resize', draw)
})
onBeforeUnmount(() => {
  cancelAnimationFrame(rafId)
  window.removeEventListener('resize', draw)
})
</script>

<template>
  <div class="trend-chart-wrap">
    <div class="chart-toolbar">
      <div class="segmented">
        <button v-for="option in rangeOptions" :key="option.id" type="button" :class="{ active: range === option.id }" @click="range = option.id">{{ option.label }}</button>
      </div>
    </div>
    <div class="trend-chart">
      <canvas ref="canvas"></canvas>
      <div v-if="!filteredRecords.length" class="chart-empty">{{ emptyText }}</div>
      <div v-else-if="filteredRecords.length === 1" class="chart-hint">只有 1 条记录，还看不出趋势</div>
    </div>
  </div>
</template>
