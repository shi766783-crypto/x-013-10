<script setup>
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import { dateRanges, filterRecordsByRange, normalRanges, todayISO } from '../data'

const props = defineProps({
  records: { type: Array, default: () => [] },
  type: { type: String, default: 'weight' },
  modelValue: { type: String, default: '7d' },
})
const emit = defineEmits(['update:modelValue'])

const canvas = ref(null)
const range = computed(() => normalRanges[props.type])
const rangeOptions = dateRanges
const visibleRecords = computed(() => filterRecordsByRange(props.records, props.modelValue))
const activeRangeLabel = computed(() => dateRanges.find((item) => item.key === props.modelValue)?.label || '')
const emptyText = computed(() => (
  props.modelValue === 'all'
    ? '还没有记录，添加一次健康数据吧'
    : `${activeRangeLabel.value}内还没有记录，可以切换到「全部」看看`
))

const PAD_L = 28
const PAD_R = 14
const PAD_T = 16
const PAD_B = 24
const ANIMATION_MS = 320

function dayValue(dateString) {
  const [year, month, day] = dateString.split('-').map(Number)
  return Math.floor(Date.UTC(year, month - 1, day) / 86400000)
}

function tickLabel(day) {
  const date = new Date(day * 86400000)
  const month = date.getUTCMonth() + 1
  const d = date.getUTCDate()
  const iso = `${date.getUTCFullYear()}-${String(month).padStart(2, '0')}-${String(d).padStart(2, '0')}`
  if (iso === todayISO()) return '今天'
  return `${month}/${d}`
}

function buildTicks(start, end) {
  const span = end - start
  const choices = [1, 2, 5, 7, 10, 14, 30, 60, 90]
  const step = choices.find((candidate) => span / candidate <= 8) || Math.ceil((span / 8) / 30) * 30
  const first = Math.ceil(start / step) * step
  const ticks = []
  for (let day = first; day <= end + 0.001; day += step) ticks.push(day)
  return ticks
}

// 场景 = 某个范围 + 数据集下图表的完整坐标系；切换范围时在两个场景之间插值
function buildScene(list) {
  const config = dateRanges.find((item) => item.key === props.modelValue)
  const today = dayValue(todayISO())
  let start
  let end
  if (config.days) {
    start = today - (config.days - 1)
    end = today
  } else if (list.length) {
    const days = list.map((item) => dayValue(item.date))
    start = Math.min(...days)
    end = Math.max(...days)
  } else {
    start = today - 6
    end = today
  }
  // 只有一条记录时（“全部”范围没有固定时间窗）向两侧留白，
  // 避免单点被硬撑满整条横轴；固定窗口内保留真实日期位置，由提示文案说明
  if (list.length === 1 && !config.days) {
    const only = dayValue(list[0].date)
    start = only - 3
    end = only + 3
  }
  if (end - start < 1) {
    start -= 1
    end += 1
  }
  const values = list.map((item) => Number(item.value))
  const span = range.value.max - range.value.min
  const vMin = (values.length ? Math.min(...values, range.value.min) : range.value.min) - span * 0.2
  const vMax = (values.length ? Math.max(...values, range.value.max) : range.value.max) + span * 0.2
  return { list, start, end, vMin, vMax, ticks: buildTicks(start, end) }
}

let prevScene = null
let nextScene = null
let progress = 1
let frameId = 0

const ease = (t) => (t < 0.5 ? 4 * t * t * t : 1 - ((-2 * t + 2) ** 3) / 2)
const lerp = (a, b, t) => a + (b - a) * t

function render() {
  const element = canvas.value
  if (!element || !prevScene || !nextScene) return
  const rect = element.getBoundingClientRect()
  if (!rect.width || !rect.height) return
  const ratio = window.devicePixelRatio || 1
  const pixelWidth = Math.round(rect.width * ratio)
  const pixelHeight = Math.round(rect.height * ratio)
  if (element.width !== pixelWidth) element.width = pixelWidth
  if (element.height !== pixelHeight) element.height = pixelHeight
  const ctx = element.getContext('2d')
  ctx.setTransform(ratio, 0, 0, ratio, 0, 0)
  const width = rect.width
  const height = rect.height
  const plotHeight = height - PAD_T - PAD_B
  ctx.clearRect(0, 0, width, height)
  ctx.font = '10px Inter, "Noto Sans SC", sans-serif'

  const t = ease(progress)
  const start = lerp(prevScene.start, nextScene.start, t)
  const end = lerp(prevScene.end, nextScene.end, t)
  const vMin = lerp(prevScene.vMin, nextScene.vMin, t)
  const vMax = lerp(prevScene.vMax, nextScene.vMax, t)
  const x = (day) => PAD_L + ((day - start) / (end - start)) * (width - PAD_L - PAD_R)
  const y = (value) => PAD_T + (1 - (value - vMin) / (vMax - vMin)) * plotHeight

  // 水平网格线
  ctx.strokeStyle = '#e7ebe4'
  ctx.lineWidth = 1
  ;[0.25, 0.5, 0.75].forEach((step) => {
    ctx.beginPath()
    ctx.moveTo(PAD_L, PAD_T + plotHeight * step)
    ctx.lineTo(width - PAD_R, PAD_T + plotHeight * step)
    ctx.stroke()
  })

  // 正常范围色带（坐标随场景插值，自然伸缩）
  ctx.fillStyle = '#f0f7dd'
  const bandTop = Math.max(PAD_T, y(range.value.max))
  const bandBottom = Math.min(PAD_T + plotHeight, y(range.value.min))
  if (bandBottom > bandTop) ctx.fillRect(PAD_L, bandTop, width - PAD_L - PAD_R, bandBottom - bandTop)

  // 纵向日期刻度：新旧两套刻度交叉淡入淡出
  const tickMap = new Map()
  prevScene.ticks.forEach((day) => tickMap.set(day, { inPrev: true }))
  nextScene.ticks.forEach((day) => {
    const existing = tickMap.get(day)
    if (existing) existing.inNext = true
    else tickMap.set(day, { inPrev: false, inNext: true })
  })
  ctx.textAlign = 'center'
  tickMap.forEach((flags, day) => {
    const alpha = flags.inPrev && flags.inNext ? 1 : flags.inNext ? t : 1 - t
    const xx = x(day)
    if (xx < PAD_L - 1 || xx > width - PAD_R + 1) return
    ctx.globalAlpha = alpha * 0.7
    ctx.strokeStyle = '#eef1ec'
    ctx.beginPath()
    ctx.moveTo(xx, PAD_T)
    ctx.lineTo(xx, PAD_T + plotHeight)
    ctx.stroke()
    ctx.globalAlpha = alpha
    const label = tickLabel(day)
    ctx.font = label === '今天' ? 'bold 10px Inter, "Noto Sans SC", sans-serif' : '10px Inter, "Noto Sans SC", sans-serif'
    ctx.fillStyle = label === '今天' ? '#718c24' : '#9aa493'
    ctx.fillText(label, xx, height - 7)
  })
  ctx.globalAlpha = 1

  // 合并新旧场景的数据点：按 id 匹配，移动中的点做位置插值
  const merged = new Map()
  prevScene.list.forEach((item) => {
    merged.set(item.id, {
      item,
      prev: { x: x(dayValue(item.date)), y: y(Number(item.value)) },
      next: null,
    })
  })
  nextScene.list.forEach((item) => {
    const existing = merged.get(item.id)
    const point = { x: x(dayValue(item.date)), y: y(Number(item.value)) }
    if (existing) existing.next = point
    else merged.set(item.id, { item, prev: null, next: point })
  })
  const points = [...merged.values()]
    .map((entry) => {
      if (entry.prev && entry.next) return { ...entry, x: lerp(entry.prev.x, entry.next.x, t), y: lerp(entry.prev.y, entry.next.y, t), alpha: 1 }
      if (entry.next) return { ...entry, x: entry.next.x, y: entry.next.y, alpha: t }
      return { ...entry, x: entry.prev.x, y: entry.prev.y, alpha: 1 - t }
    })
    .sort((a, b) => a.item.date.localeCompare(b.item.date))

  // 折线：同一场景内相邻才连接，避免不同场景的点之间拉出奇怪的过渡线
  ctx.strokeStyle = '#b5cf3e'
  ctx.lineWidth = 2.5
  ctx.lineJoin = 'round'
  for (let i = 1; i < points.length; i += 1) {
    const a = points[i - 1]
    const b = points[i]
    let alpha = 0
    if (a.prev && b.prev && a.next && b.next) alpha = 1
    else if (a.prev && b.prev && !a.next && !b.next) alpha = 1 - t
    else if (a.next && b.next && !a.prev && !b.prev) alpha = t
    if (!alpha) continue
    ctx.globalAlpha = alpha
    ctx.beginPath()
    ctx.moveTo(a.x, a.y)
    ctx.lineTo(b.x, b.y)
    ctx.stroke()
  }

  // 数据点
  points.forEach((point) => {
    ctx.globalAlpha = point.alpha
    ctx.beginPath()
    ctx.arc(point.x, point.y, 4, 0, Math.PI * 2)
    ctx.fillStyle = '#ffffff'
    ctx.fill()
    ctx.strokeStyle = '#87a52f'
    ctx.lineWidth = 2
    ctx.stroke()
  })
  ctx.globalAlpha = 1

  // 正常范围数值标注
  ctx.textAlign = 'left'
  ctx.fillStyle = '#8b9686'
  ctx.fillText(`${range.value.max}`, 4, Math.max(PAD_T + 4, y(range.value.max) - 6))
  ctx.fillText(`${range.value.min}`, 4, Math.min(PAD_T + plotHeight, y(range.value.min) + 13))
}

function startTransition() {
  const scene = buildScene(visibleRecords.value)
  if (!prevScene) prevScene = scene
  nextScene = scene
  progress = 0
  cancelAnimationFrame(frameId)
  const startedAt = performance.now()
  const step = (now) => {
    progress = Math.min(1, (now - startedAt) / ANIMATION_MS)
    render()
    if (progress < 1) {
      frameId = requestAnimationFrame(step)
    } else {
      prevScene = nextScene
      frameId = 0
    }
  }
  frameId = requestAnimationFrame(step)
}

function handleResize() {
  if (progress >= 1) {
    prevScene = buildScene(visibleRecords.value)
    nextScene = prevScene
  }
  render()
}

watch([visibleRecords, () => props.type], () => startTransition())
onMounted(() => {
  startTransition()
  window.addEventListener('resize', handleResize)
})
onBeforeUnmount(() => {
  cancelAnimationFrame(frameId)
  window.removeEventListener('resize', handleResize)
})
</script>

<template>
  <div class="chart-card">
    <div class="chart-toolbar">
      <span class="chart-count">{{ visibleRecords.length }} 条记录</span>
      <div class="segmented chart-range">
        <button
          v-for="option in rangeOptions"
          :key="option.key"
          :class="{ active: modelValue === option.key }"
          @click="emit('update:modelValue', option.key)"
        >{{ option.label }}</button>
      </div>
    </div>
    <div class="trend-chart">
      <canvas ref="canvas"></canvas>
      <div v-if="!visibleRecords.length" class="chart-empty">{{ emptyText }}</div>
      <div v-else-if="visibleRecords.length === 1" class="chart-hint">仅有 1 条记录，暂时看不出趋势</div>
    </div>
  </div>
</template>
