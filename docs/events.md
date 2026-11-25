---
aside: false
outline: false
---

<script setup>
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'

const timelineScroller = ref(null)
const timelineProgress = ref(null)
const timelineThumbWidth = ref(100)
const timelineThumbLeft = ref(0)
const timelineIsScrollable = ref(false)
const timelineCanScrollLeft = ref(false)
const timelineCanScrollRight = ref(false)

const archiveQuery = ref('')
const archiveSelectedIndex = ref(0)
const archiveItems = [
  {
    id: 'recent',
    title: '待补充',
    date: '待补充',
    summary: '待补充',
    keywords: '待补充',
    priority: 30
  },
  {
    id: 'history',
    title: '待补充',
    date: '待补充',
    summary: '待补充',
    keywords: '待补充',
    priority: 20
  },
  {
    id: 'materials',
    title: '待补充',
    date: '待补充',
    summary: '待补充',
    keywords: '待补充',
    priority: 10
  }
]

let resizeObserver
let dragStartX = 0
let dragStartScroll = 0
let dragScale = 1
let archiveWheelLocked = false
let archiveWheelTimer

function syncTimelineScrollbar() {
  const scroller = timelineScroller.value
  if (!scroller) return

  const maxScroll = Math.max(scroller.scrollWidth - scroller.clientWidth, 0)
  timelineIsScrollable.value = maxScroll > 1
  timelineCanScrollLeft.value = scroller.scrollLeft > 2
  timelineCanScrollRight.value = maxScroll - scroller.scrollLeft > 2

  if (!timelineIsScrollable.value) {
    timelineThumbWidth.value = 100
    timelineThumbLeft.value = 0
    return
  }

  timelineThumbWidth.value = Math.min(
    100,
    Math.max((scroller.clientWidth / scroller.scrollWidth) * 100, 12)
  )
  timelineThumbLeft.value =
    (scroller.scrollLeft / maxScroll) * (100 - timelineThumbWidth.value)
}

function scrollTimeline(direction) {
  const scroller = timelineScroller.value
  if (!scroller) return

  scroller.scrollBy({
    left: direction * Math.max(scroller.clientWidth * 0.72, 260),
    behavior: window.matchMedia('(prefers-reduced-motion: reduce)').matches
      ? 'auto'
      : 'smooth'
  })
}

function stopTimelineDrag() {
  window.removeEventListener('pointermove', dragTimeline)
  window.removeEventListener('pointerup', stopTimelineDrag)
}

function dragTimeline(event) {
  const scroller = timelineScroller.value
  const track = timelineProgress.value
  if (!scroller || !track) return

  const maxScroll = Math.max(scroller.scrollWidth - scroller.clientWidth, 0)
  scroller.scrollLeft = Math.min(
    maxScroll,
    Math.max(0, dragStartScroll + (event.clientX - dragStartX) * dragScale)
  )
}

function startTimelineDrag(event) {
  const scroller = timelineScroller.value
  const track = timelineProgress.value
  if (!scroller || !track || !timelineIsScrollable.value) return

  const thumbWidth = track.clientWidth * (timelineThumbWidth.value / 100)
  const draggableWidth = Math.max(track.clientWidth - thumbWidth, 1)
  const maxScroll = Math.max(scroller.scrollWidth - scroller.clientWidth, 0)

  dragStartX = event.clientX
  dragStartScroll = scroller.scrollLeft
  dragScale = maxScroll / draggableWidth

  window.addEventListener('pointermove', dragTimeline)
  window.addEventListener('pointerup', stopTimelineDrag, { once: true })
}

function normalizeArchiveQuery(value) {
  return value.toLocaleLowerCase('zh-CN').replace(/\s+/g, ' ').trim()
}

function archiveScore(item, query) {
  const title = item.title.toLocaleLowerCase('zh-CN')
  const summary = item.summary.toLocaleLowerCase('zh-CN')
  const keywords = item.keywords.toLocaleLowerCase('zh-CN')

  let score = 0
  if (title === query) score += 120
  else if (title.includes(query)) score += 90
  if (summary.includes(query)) score += 50
  if (keywords.includes(query)) score += 40
  return score
}

const archiveMatches = computed(() => {
  const query = normalizeArchiveQuery(archiveQuery.value)
  const matches = archiveItems.map((item) => ({
    item,
    score: query ? archiveScore(item, query) : 0
  }))

  if (!query) {
    return matches
      .map(({ item }) => item)
      .sort((left, right) => right.priority - left.priority)
  }

  return matches
    .filter(({ score }) => score > 0)
    .sort(
      (left, right) =>
        right.score - left.score || right.item.priority - left.item.priority
    )
    .map(({ item }) => item)
})

const archiveWheelCards = computed(() => {
  const matches = archiveMatches.value
  const index = archiveSelectedIndex.value
  if (!matches.length) return []

  return matches
    .map((item, itemIndex) => {
      let offset = itemIndex - index
      if (offset > matches.length / 2) offset -= matches.length
      if (offset < -matches.length / 2) offset += matches.length
      return { item, offset }
    })
    .filter(({ offset }) => Math.abs(offset) <= 1)
})

function stepArchive(direction) {
  const length = archiveMatches.value.length
  if (!length) return
  archiveSelectedIndex.value =
    (archiveSelectedIndex.value + direction + length) % length
}

function selectArchiveOffset(offset) {
  if (!offset) return
  stepArchive(offset)
}

function handleArchiveWheel(event) {
  if (
    archiveMatches.value.length <= 1 ||
    Math.abs(event.deltaY) < 4
  ) {
    return
  }

  event.preventDefault()
  if (archiveWheelLocked) return

  archiveWheelLocked = true
  stepArchive(event.deltaY > 0 ? 1 : -1)
  archiveWheelTimer = window.setTimeout(() => {
    archiveWheelLocked = false
  }, 520)
}

watch(archiveQuery, () => {
  archiveSelectedIndex.value = 0
})

onMounted(() => {
  syncTimelineScrollbar()
  resizeObserver = new ResizeObserver(syncTimelineScrollbar)
  if (timelineScroller.value) resizeObserver.observe(timelineScroller.value)
  if (timelineProgress.value) resizeObserver.observe(timelineProgress.value)
  window.addEventListener('resize', syncTimelineScrollbar)
})

onBeforeUnmount(() => {
  resizeObserver?.disconnect()
  window.removeEventListener('resize', syncTimelineScrollbar)
  window.clearTimeout(archiveWheelTimer)
  stopTimelineDrag()
})
</script>

<div class="events-page">

<section class="events-upcoming" aria-labelledby="upcoming-title">
  <div class="events-upcoming-copy">
    <span class="events-upcoming-kicker">UPCOMING EVENT</span>
    <h1 id="upcoming-title">活动公告</h1>
  </div>

  <article class="events-feature-card">
    <div class="events-feature-side">
      <h2>最新公告</h2>
    </div>
    <div class="events-feature-content">
      <div class="events-placeholder-lines">
        <p>待补充</p>
        <p>待补充</p>
        <p>待补充</p>
      </div>
    </div>
  </article>
</section>

<div class="events-dark-flow">

<section id="calendar" class="events-calendar" aria-labelledby="calendar-title">
  <div class="events-calendar-head">
    <div>
      <span class="events-kicker">ACTIVITY CALENDAR</span>
      <h2 id="calendar-title">活动日历</h2>
    </div>
    <div class="events-calendar-tools">
      <p>活动日期确认后将按时间顺序展示，方便提前安排与报名。</p>
      <div class="events-timeline-nav" aria-label="活动月份导航">
        <button
          type="button"
          aria-label="查看更早的活动"
          :disabled="!timelineCanScrollLeft"
          @click="scrollTimeline(-1)"
        >
          <svg viewBox="0 0 20 20" aria-hidden="true">
            <path d="m12.5 4.5-5 5.5 5 5.5"></path>
          </svg>
        </button>
        <button
          type="button"
          aria-label="查看更晚的活动"
          :disabled="!timelineCanScrollRight"
          @click="scrollTimeline(1)"
        >
          <svg viewBox="0 0 20 20" aria-hidden="true">
            <path d="m7.5 4.5 5 5.5-5 5.5"></path>
          </svg>
        </button>
      </div>
    </div>
  </div>

  <div class="events-timeline" aria-label="活动月份分组">
    <div
      ref="timelineScroller"
      class="events-timeline-scroller"
      tabindex="0"
      aria-label="左右滑动查看更多活动"
      @scroll="syncTimelineScrollbar"
    >
      <div class="events-timeline-track">
        <section class="events-month-column">
          <div class="events-month-header">
            <div>
              <span class="events-month-code">01</span>
              <h3>月份待补充</h3>
            </div>
            <span aria-hidden="true">•••</span>
          </div>
          <div class="events-month-cards">
            <article class="events-event-card">
              <span class="events-card-date">日期待补充</span>
              <h4>活动名称待补充</h4>
              <p>活动时间、地点与参与方式待补充。</p>
              <span class="events-card-status">即将公布</span>
            </article>
            <article class="events-event-card">
              <span class="events-card-date">日期待补充</span>
              <h4>活动名称待补充</h4>
              <p>活动内容与报名信息待补充。</p>
              <span class="events-card-status">筹备中</span>
            </article>
          </div>
        </section>
        <section class="events-month-column">
          <div class="events-month-header">
            <div>
              <span class="events-month-code">02</span>
              <h3>月份待补充</h3>
            </div>
            <span aria-hidden="true">•••</span>
          </div>
          <div class="events-month-cards">
            <article class="events-event-card">
              <span class="events-card-date">日期待补充</span>
              <h4>活动名称待补充</h4>
              <p>活动详情与参与方式待补充。</p>
              <span class="events-card-status">等待通知</span>
            </article>
          </div>
        </section>
      </div>
    </div>
    <div class="events-scroll-control">
      <span class="events-scroll-label">左右拖动</span>
      <div
        ref="timelineProgress"
        class="events-timeline-scrollbar"
        :class="{ 'is-scrollable': timelineIsScrollable }"
        role="scrollbar"
        aria-label="左右拖动查看活动"
        :aria-valuemin="0"
        :aria-valuemax="100"
        :aria-valuenow="Math.round(timelineThumbLeft)"
      >
        <span
          class="events-timeline-thumb"
          :style="{
            width: `${timelineThumbWidth}%`,
            left: `${timelineThumbLeft}%`
          }"
          @pointerdown.prevent="startTimelineDrag"
        >
          <span class="events-drag-grip" aria-hidden="true"></span>
        </span>
      </div>
    </div>
  </div>
</section>

<section id="archive" class="events-archive" aria-labelledby="history-title">
  <div class="events-archive-head">
    <div>
      <span class="events-kicker">ARCHIVE</span>
      <h2 id="history-title">历史公告</h2>
    </div>
    <div class="events-archive-tools">
      <label class="events-archive-search">
        <svg viewBox="0 0 20 20" aria-hidden="true">
          <circle cx="8.5" cy="8.5" r="5"></circle>
          <path d="m12.2 12.2 4 4"></path>
        </svg>
        <input
          v-model.trim="archiveQuery"
          type="search"
          placeholder="搜索活动名称或关键词"
          aria-label="搜索历史公告"
        />
      </label>
    </div>
  </div>

  <div
    v-if="archiveMatches.length"
    class="events-archive-wheel"
    aria-label="历史公告滚轮，可使用鼠标滚轮翻页"
    aria-live="polite"
    @wheel="handleArchiveWheel"
  >
    <div class="events-archive-stage">
      <article
        v-for="card in archiveWheelCards"
        :key="card.item.id"
        class="events-archive-card"
        :class="{
          'is-active': card.offset === 0,
          'is-previous': card.offset === -1,
          'is-next': card.offset === 1
        }"
        :tabindex="card.offset === 0 ? 0 : -1"
        @click="selectArchiveOffset(card.offset)"
      >
        <span class="events-archive-date">{{ card.item.date }}</span>
        <h3>{{ card.item.title }}</h3>
        <p>{{ card.item.summary }}</p>
        <span v-if="card.offset === 0" class="events-archive-current">
          正在展示
        </span>
      </article>
    </div>
  </div>
  <p v-else class="events-archive-empty">
    没有找到匹配的历史公告，请尝试其他关键词。
  </p>
</section>

</div>

</div>

<style scoped>
.events-page {
  --events-orange: #f5a623;
  --events-teal: var(--vp-c-brand-2);
  --events-teal-deep: #07363a;
  --events-border: var(--vp-c-divider);
  --events-surface: var(--vp-c-bg-soft);
  --events-card: var(--vp-c-bg);
  --events-text: var(--vp-c-text-1);
  --events-muted: var(--vp-c-text-2);
  width: 100%;
  margin: 0 0 0;
  color: var(--events-text);
}

:global(html:has(.events-page)) {
  background: #edf8ef;
  color-scheme: light;
}

:global(body:has(.events-page)) {
  --vp-c-bg: #f8fcf8;
  --vp-c-bg-alt: #edf7ef;
  --vp-c-bg-soft: #f2faf3;
  --vp-c-text-1: #1f2922;
  --vp-c-text-2: #617066;
  --vp-c-text-3: #859188;
  --vp-c-divider: rgba(37, 73, 48, 0.13);
  --vp-c-gutter: #edf8ef;
  --vp-c-brand-1: #007f85;
  --vp-c-brand-2: #00979d;
  --vp-c-brand-3: #00aab1;
  --vp-c-brand-soft: rgba(0, 151, 157, 0.12);
  --vp-nav-bg-color: #eef8f0;
  --vp-nav-screen-bg-color: #eef8f0;
  --vp-local-nav-bg-color: #eef8f0;
  background:
    radial-gradient(circle at 15% 4%, rgba(0, 151, 157, 0.09), transparent 30%),
    linear-gradient(180deg, #e9f6ec 0%, #f8fcf8 52%, #ffffff 100%);
  color: #1f2922;
  overflow-x: hidden;
}

@media (min-width: 960px) {
  :global(.VPDoc:not(.has-sidebar) .container:has(.events-page)) {
    max-width: none !important;
  }

  :global(.VPDoc:not(.has-sidebar) .content:has(.events-page)) {
    max-width: none !important;
    padding-right: clamp(24px, 3vw, 48px);
    padding-left: clamp(24px, 3vw, 48px);
  }
}

:global(.vp-doc._events h1) {
  color: #005f66;
  background: none;
  -webkit-text-fill-color: #005f66;
  font-size: clamp(48px, 4.8vw, 64px);
  line-height: 1.2;
  letter-spacing: -0.045em;
  animation: events-title-in 0.8s cubic-bezier(0.22, 1, 0.36, 1) both;
}

.events-upcoming {
  padding: clamp(0px, 1vw, 16px) 0 clamp(72px, 8vw, 112px);
  text-align: center;
}

.events-upcoming-copy {
  position: relative;
  top: -32px;
  max-width: 980px;
  margin: 0 auto;
}

.events-upcoming-kicker {
  color: #00979d;
  font-size: 12px;
  font-weight: 700;
  letter-spacing: 0.12em;
}

.events-upcoming-copy h1 {
  max-width: 980px;
  margin: 16px auto 0;
  font-weight: 600;
}

.events-feature-card {
  display: grid;
  grid-template-columns: minmax(240px, 0.32fr) minmax(0, 0.68fr);
  width: min(100%, 980px);
  margin: clamp(24px, 3vw, 36px) auto 0;
  overflow: hidden;
  border: 1px solid rgba(0, 95, 80, 0.07);
  border-radius: 20px;
  background: #ffffff;
  box-shadow: 0 18px 48px rgba(38, 75, 51, 0.08);
  text-align: left;
}

.events-feature-side {
  display: flex;
  align-items: center;
  justify-content: center;
  border-right: 1px solid rgba(0, 151, 157, 0.08);
  padding: clamp(30px, 4vw, 48px);
  background: #f3faf4;
}

.events-feature-side h2 {
  margin: 0;
  color: #202124;
  font-size: clamp(28px, 3vw, 42px);
  font-weight: 500;
  line-height: 1.3;
  letter-spacing: -0.035em;
}

.events-feature-content {
  display: flex;
  flex-direction: column;
  justify-content: center;
  padding: clamp(34px, 5vw, 58px);
}

.events-placeholder-lines {
  border-top: 1px solid rgba(46, 58, 49, 0.1);
}

.events-placeholder-lines p {
  margin: 0;
  border-bottom: 1px solid rgba(46, 58, 49, 0.1);
  padding: 22px 4px;
  color: #556158;
  font-size: 18px;
}

@keyframes events-title-in {
  from {
    opacity: 0;
    transform: translateY(18px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.events-rule {
  position: relative;
  width: 100%;
  height: 1px;
  margin: 16px 0 20px;
  background: linear-gradient(
    90deg,
    rgba(0, 151, 157, 0.95),
    rgba(0, 151, 157, 0.2) 58%,
    rgba(245, 166, 35, 0.8)
  );
  box-shadow: 0 0 14px rgba(0, 151, 157, 0.16);
  transform: scaleX(0);
  transform-origin: left center;
  animation: events-rule-draw 1.1s cubic-bezier(0.22, 1, 0.36, 1) 0.15s both;
}

.events-rule::before {
  content: '';
  position: absolute;
  top: -5px;
  left: -12%;
  width: 120px;
  height: 11px;
  border-radius: 999px;
  background: linear-gradient(90deg, transparent, rgba(139, 233, 236, 0.58), transparent);
  filter: blur(2px);
  animation: events-rule-scan 5.5s ease-in-out 1.2s infinite;
}

.events-rule::after {
  content: '';
  position: absolute;
  top: 50%;
  right: 0;
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: #f5a623;
  box-shadow: 0 0 12px rgba(245, 166, 35, 0.72);
  opacity: 0;
  transform: translate(50%, -50%);
  animation: events-rule-spark 0.4s ease 0.9s forwards;
}

.events-rule--section {
  margin: clamp(44px, 5vw, 68px) 0 28px;
  animation-delay: 0.2s;
}

@keyframes events-rule-draw {
  to {
    transform: scaleX(1);
  }
}

@keyframes events-rule-scan {
  0%,
  18% {
    left: -12%;
    opacity: 0;
  }

  32% {
    opacity: 0.9;
  }

  72% {
    opacity: 0.55;
  }

  100% {
    left: 100%;
    opacity: 0;
  }
}

@keyframes events-rule-spark {
  to {
    opacity: 1;
  }
}

.events-page h2,
.events-page h3,
.events-page p {
  margin: 0;
}

.events-page h2 {
  margin: 0;
  border-top: 0;
  padding-top: 0;
  font-size: clamp(30px, 2.7vw, 38px);
  line-height: 1.2;
  letter-spacing: -0.025em;
}

.events-page h3 {
  font-size: 16px;
  line-height: 1.45;
}

.events-lead {
  max-width: 720px;
  margin: 0 0 28px !important;
  color: var(--events-muted);
  font-size: 16.5px;
  line-height: 1.8;
}

@media (min-width: 960px) {
  .events-opening {
    display: flex;
    min-height: calc(100vh - var(--vp-nav-height) - 140px);
    flex-direction: column;
    justify-content: center;
  }
}

.events-hero-layout {
  display: grid;
  grid-template-columns: minmax(0, 1fr) minmax(300px, 380px);
  align-items: stretch;
  gap: clamp(24px, 4vw, 56px);
  margin-bottom: 20px;
}

.events-hero-card {
  position: relative;
  display: flex;
  min-height: 300px;
  flex-direction: column;
  justify-content: center;
  overflow: visible;
  border: 0;
  padding: 0;
  background: transparent;
  box-shadow: none;
}

.events-heading {
  display: flex;
  align-items: center;
  gap: 14px;
  margin-bottom: 24px;
}

.events-heading--compact {
  margin-bottom: 0;
}

.events-caret {
  width: 0;
  height: 0;
  flex: 0 0 auto;
  border-top: 9px solid transparent;
  border-right: 15px solid #70c879;
  border-bottom: 9px solid transparent;
  filter: drop-shadow(0 4px 8px rgba(112, 200, 121, 0.2));
}

.events-kicker {
  display: block;
  margin-bottom: 4px;
  color: var(--events-teal);
  font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
  font-size: 11px;
  font-weight: 700;
  letter-spacing: 0.14em;
}

.events-latest {
  display: grid;
  grid-template-columns: auto minmax(0, 1fr);
  align-items: start;
  gap: clamp(36px, 6vw, 96px);
}

.events-hero-card .events-heading {
  margin-bottom: 0;
}

.events-heading--section {
  display: grid;
  grid-template-columns: 16px minmax(0, 1fr);
  grid-template-rows: auto auto;
  align-items: center;
  column-gap: 14px;
  row-gap: 6px;
}

.events-heading--section .events-caret {
  grid-column: 1;
  grid-row: 2;
  align-self: center;
}

.events-heading--latest .events-caret {
  border-right: 0;
  border-left: 15px solid #70c879;
}

.events-heading--section .events-kicker {
  grid-column: 2;
  grid-row: 1;
  margin-bottom: 0;
}

.events-heading--section h2 {
  grid-column: 2;
  grid-row: 2;
}

.events-hero-card .events-caret {
  animation: events-caret-turn 1.1s cubic-bezier(0.22, 1, 0.36, 1) both;
}

@keyframes events-caret-turn {
  from {
    opacity: 0;
    transform: rotate(180deg);
  }

  to {
    opacity: 1;
    transform: rotate(0deg);
  }
}

.events-latest-list {
  margin: 0;
  padding: 4px 0 0;
  list-style: none;
}

.events-latest-list li {
  padding: 8px 0;
  color: var(--events-text);
  font-size: clamp(18px, 1.7vw, 22px);
  font-weight: 600;
  line-height: 1.45;
  transition:
    color 0.2s ease,
    transform 0.2s ease;
}

.events-latest-list li:hover {
  transform: translateX(4px);
  color: var(--vp-c-brand-1);
}

.events-visual {
  position: relative;
  display: flex;
  height: 100%;
  min-height: 300px;
  align-items: center;
  justify-content: center;
  overflow: hidden;
}

.events-visual::before {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 280px;
  height: 280px;
  border-radius: 50%;
  background: radial-gradient(
    circle,
    rgba(0, 151, 157, 0.18),
    rgba(245, 166, 35, 0.08) 48%,
    transparent 72%
  );
  filter: blur(10px);
  pointer-events: none;
  transform: translate(-50%, -50%);
}

.events-visual img {
  position: absolute;
  top: 50%;
  left: 50%;
  z-index: 1;
  width: 560px;
  max-width: none;
  height: auto;
  filter: drop-shadow(0 18px 26px rgba(0, 63, 68, 0.2));
  transform: translate(-50%, -50%);
  animation: events-chip-breathe 6s ease-in-out infinite;
}

@keyframes events-chip-breathe {
  0%,
  100% {
    transform: translate(-50%, -50%) scale(1) rotate(0deg);
  }

  50% {
    transform: translate(-50%, -50%) scale(1.025) rotate(0.35deg);
  }
}

.events-toolbar {
  display: grid;
  grid-template-columns: auto minmax(0, 1fr) auto;
  align-items: center;
  gap: 24px;
  margin: 28px 0 0;
  padding: 0 0 18px;
}

.events-filters {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.events-filters a,
.events-search {
  border: 1px solid var(--events-border);
  border-radius: 999px;
  color: var(--events-muted);
  text-decoration: none;
  transition:
    border-color 0.2s ease,
    color 0.2s ease,
    background-color 0.2s ease;
}

.events-filters a {
  padding: 7px 14px;
  font-size: 13px;
}

.events-filters a:hover,
.events-filters a.is-active {
  border-color: var(--events-teal);
  background: var(--vp-c-brand-soft);
  color: var(--vp-c-brand-1);
}

.events-search {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 7px 14px;
  font-size: 13px;
  font-weight: 600;
}

.events-search:hover {
  border-color: var(--events-orange);
  color: var(--events-text);
}

.events-search svg {
  width: 16px;
  height: 16px;
  fill: none;
  stroke: currentColor;
  stroke-linecap: round;
  stroke-width: 1.8;
}

.events-dark-flow {
  position: relative;
  overflow: visible;
  width: 100%;
  max-width: 100%;
  margin: 0;
  border: 0;
  padding: clamp(56px, 7vw, 96px) 0;
  background: transparent;
  color: #25241f;
}

.events-dark-flow::before {
  display: none;
}

.events-dark-flow::after {
  display: none;
}

.events-dark-flow > * {
  position: relative;
  z-index: 1;
}

.events-dark-flow .events-kicker {
  color: #00979d;
}

.events-dark-flow h2,
.events-dark-flow h3 {
  color: #202124;
}

.events-dark-flow > section + section {
  margin-top: clamp(72px, 9vw, 120px);
}

@keyframes events-ambient {
  from {
    opacity: 0.55;
    transform: translate3d(-5%, -3%, 0) scale(0.94);
  }

  to {
    opacity: 1;
    transform: translate3d(4%, 5%, 0) scale(1.08);
  }
}

@keyframes events-card-reveal {
  from {
    opacity: 0;
    transform: translateY(22px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.events-month-column {
  animation: events-card-reveal 0.65s cubic-bezier(0.22, 1, 0.36, 1) both;
}

.events-month-column:nth-child(2) {
  animation-delay: 0.08s;
}

.events-month-column:nth-child(3) {
  animation-delay: 0.16s;
}

@supports (animation-timeline: view()) {
  .events-month-column {
    animation: events-card-reveal linear both;
    animation-timeline: view();
    animation-range: entry 8% cover 32%;
  }
}

.events-calendar {
  position: relative;
  min-width: 0;
  overflow: clip;
  margin: 0;
  border: 0;
  border-radius: 0;
  padding: 0;
  background: transparent;
  color: #25241f;
}

.events-calendar-head,
.events-timeline {
  position: relative;
  z-index: 1;
}

.events-calendar-head {
  display: flex;
  align-items: end;
  justify-content: space-between;
  gap: 24px;
  margin-bottom: clamp(88px, 10vw, 136px);
}

.events-calendar .events-kicker {
  color: #00979d;
}

.events-calendar h2 {
  color: #202124;
  font-size: clamp(34px, 3.2vw, 46px);
}

.events-calendar-head > div {
  padding-left: 0;
}

.events-calendar-head p {
  max-width: 380px;
  color: #666876;
  font-size: 14px;
  line-height: 1.75;
  text-align: right;
}

.events-calendar-tools {
  display: flex;
  align-items: center;
  gap: 20px;
}

.events-timeline-nav {
  display: flex;
  flex: 0 0 auto;
  gap: 8px;
}

.events-timeline-nav button {
  display: inline-flex;
  width: 36px;
  height: 36px;
  align-items: center;
  justify-content: center;
  border: 1px solid rgba(45, 127, 118, 0.18);
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.42);
  color: #2d7f76;
  cursor: pointer;
  transition:
    border-color 0.2s ease,
    background-color 0.2s ease,
    color 0.2s ease,
    transform 0.2s ease;
}

.events-timeline-nav button:hover:not(:disabled) {
  border-color: rgba(45, 127, 118, 0.42);
  background: rgba(255, 255, 255, 0.72);
  color: #1f665e;
  transform: translateY(-2px);
}

.events-timeline-nav button:focus-visible {
  outline: 2px solid rgba(45, 127, 118, 0.52);
  outline-offset: 3px;
}

.events-timeline-nav button:disabled {
  opacity: 0.3;
  cursor: default;
}

.events-timeline-nav svg {
  width: 18px;
  height: 18px;
  fill: none;
  stroke: currentColor;
  stroke-linecap: round;
  stroke-linejoin: round;
  stroke-width: 1.8;
}

.events-timeline {
  min-width: 0;
  max-width: 100%;
  padding-top: 24px;
}

.events-timeline-scroller {
  overflow-x: auto;
  padding-bottom: 6px;
  scrollbar-width: none;
  scroll-snap-type: x proximity;
  overscroll-behavior-inline: contain;
}

.events-timeline-scroller::-webkit-scrollbar {
  display: none;
}

.events-timeline-scroller:focus-visible {
  border-radius: 12px;
  outline: 2px solid rgba(139, 233, 236, 0.72);
  outline-offset: 5px;
}

.events-timeline-track {
  position: relative;
  display: flex;
  width: max-content;
  min-width: 100%;
  align-items: flex-start;
  gap: 20px;
  padding: 0 0 10px;
}

.events-scroll-control {
  display: flex;
  align-items: center;
  gap: 14px;
  margin-top: 22px;
}

.events-scroll-label {
  flex: 0 0 auto;
  color: #647267;
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.08em;
}

.events-timeline-scrollbar {
  position: relative;
  flex: 1;
  overflow: hidden;
  height: 12px;
  margin-top: 0;
  border-radius: 999px;
  background: rgba(72, 101, 77, 0.12);
  box-shadow: inset 0 1px 2px rgba(40, 62, 44, 0.08);
  opacity: 1;
}

.events-timeline-scrollbar.is-scrollable {
  height: 12px;
  opacity: 1;
  pointer-events: auto;
}

.events-timeline-thumb {
  position: absolute;
  top: 0;
  bottom: 0;
  min-width: 12%;
  border-radius: 999px;
  background: linear-gradient(90deg, #315f51, #4c8674);
  box-shadow: 0 2px 7px rgba(40, 76, 59, 0.22);
  cursor: grab;
  touch-action: none;
  transition:
    width 0.22s ease,
    left 0.1s linear;
}

.events-timeline-scrollbar.is-scrollable .events-timeline-thumb {
  cursor: grab;
}

.events-timeline-scrollbar.is-scrollable .events-timeline-thumb:active {
  cursor: grabbing;
}

.events-timeline-thumb:active {
  cursor: grabbing;
}

.events-drag-grip {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 26px;
  height: 2px;
  border-radius: 999px;
  background: rgba(239, 249, 242, 0.68);
  box-shadow:
    0 -4px 0 rgba(239, 249, 242, 0.52),
    0 4px 0 rgba(239, 249, 242, 0.52);
  transform: translate(-50%, -50%);
  pointer-events: none;
}

.events-month-column {
  display: flex;
  flex: 0 0 clamp(300px, 30vw, 360px);
  flex-direction: column;
  gap: 10px;
  scroll-snap-align: start;
}

.events-month-header {
  display: flex;
  min-height: 66px;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  border: 1px solid rgba(0, 151, 157, 0.08);
  border-radius: 10px;
  padding: 12px 20px;
  background: #ffffff;
  color: #202124;
  box-shadow: 0 12px 28px rgba(37, 73, 48, 0.07);
}

.events-month-header > div {
  display: flex;
  align-items: baseline;
  gap: 10px;
}

.events-month-header h3 {
  color: #202124;
  font-size: clamp(24px, 2.2vw, 31px);
  letter-spacing: -0.03em;
}

.events-month-header > span:last-child {
  color: #858795;
  font-size: 15px;
  letter-spacing: 0.08em;
}

.events-month-code {
  color: #00979d;
  font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
  font-size: 10px;
  font-weight: 700;
}

.events-month-cards {
  display: grid;
  gap: 12px;
}

.events-event-card {
  position: relative;
  display: flex;
  flex-direction: column;
  margin-top: 0;
  border: 1px solid rgba(0, 95, 80, 0.08);
  border-radius: 9px;
  padding: 20px;
  background: #ffffff;
  color: #202124;
  box-shadow: 0 10px 24px rgba(38, 75, 51, 0.07);
  transform-origin: top center;
  transition:
    transform 0.25s ease,
    box-shadow 0.25s ease,
    border-color 0.25s ease;
}

.events-event-card:hover {
  transform: translateY(-3px);
  box-shadow:
    0 0 0 1px rgba(45, 116, 98, 0.12),
    0 16px 30px rgba(48, 70, 51, 0.13);
}

.events-card-date {
  color: #777988;
  font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.06em;
}

.events-event-card h4 {
  margin: 14px 0 0;
  font-size: 20px;
  line-height: 1.35;
  letter-spacing: -0.02em;
}

.events-event-card p {
  margin: 8px 0 0;
  color: #686a78;
  font-size: 13px;
  line-height: 1.7;
}

.events-card-status {
  margin-top: auto;
  padding-top: 18px;
  color: #288b69;
  font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
  font-size: 10px;
  font-weight: 700;
  letter-spacing: 0.1em;
}

.events-archive {
  margin: 0;
}

.events-archive-head {
  display: flex;
  align-items: end;
  justify-content: space-between;
  gap: 20px;
  margin-bottom: clamp(52px, 6vw, 80px);
}

.events-archive-tools {
  display: flex;
  align-items: center;
  gap: 14px;
}

.events-archive-search {
  display: flex;
  min-width: min(320px, 34vw);
  align-items: center;
  gap: 9px;
  border: 1px solid rgba(0, 95, 80, 0.1);
  border-radius: 999px;
  padding: 9px 14px;
  background: #ffffff;
  transition:
    border-color 0.2s ease,
    background-color 0.2s ease,
    box-shadow 0.2s ease;
}

.events-archive-search:focus-within {
  border-color: rgba(0, 151, 157, 0.34);
  background: #ffffff;
  box-shadow: 0 0 0 4px rgba(0, 151, 157, 0.06);
}

.events-archive-search svg {
  width: 17px;
  height: 17px;
  flex: 0 0 auto;
  fill: none;
  stroke: #686a78;
  stroke-linecap: round;
  stroke-width: 1.7;
}

.events-archive-search input {
  min-width: 0;
  width: 100%;
  border: 0;
  outline: 0;
  background: transparent;
  color: #25241f;
  font: inherit;
  font-size: 13px;
}

.events-archive-search input::placeholder {
  color: #9294a2;
}

.events-archive-wheel {
  position: relative;
  perspective: 1300px;
}

.events-archive-wheel::before {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: min(78%, 760px);
  height: 250px;
  border-radius: 28px;
  background: radial-gradient(
    ellipse,
    rgba(45, 127, 118, 0.11),
    rgba(216, 187, 112, 0.09) 54%,
    transparent 74%
  );
  filter: blur(20px);
  pointer-events: none;
  transform: translate(-50%, -50%);
}

.events-archive-stage {
  position: relative;
  height: 430px;
  overflow: hidden;
  -webkit-mask-image: linear-gradient(
    to bottom,
    transparent 0%,
    #000 12%,
    #000 88%,
    transparent 100%
  );
  mask-image: linear-gradient(
    to bottom,
    transparent 0%,
    #000 12%,
    #000 88%,
    transparent 100%
  );
  transform-style: preserve-3d;
}

.events-archive-card {
  position: absolute;
  top: 50%;
  left: 50%;
  display: flex;
  width: min(76%, 720px);
  height: 230px;
  flex-direction: column;
  border: 1px solid rgba(0, 95, 80, 0.08);
  padding: 28px 32px;
  background: #ffffff;
  color: #202124;
  box-shadow: 0 20px 46px rgba(38, 75, 51, 0.1);
  transform-style: preserve-3d;
  transition:
    transform 0.55s cubic-bezier(0.22, 1, 0.36, 1),
    opacity 0.4s ease,
    filter 0.4s ease,
    border-color 0.3s ease;
  will-change: transform;
}

.events-archive-card.is-active {
  z-index: 3;
  border-color: rgba(0, 151, 157, 0.2);
  opacity: 1;
  transform: translate(-50%, -50%) translateZ(110px) scale(1.02);
  box-shadow:
    0 0 0 1px rgba(0, 151, 157, 0.06),
    0 0 38px rgba(0, 151, 157, 0.1),
    0 28px 58px rgba(38, 75, 51, 0.16);
}

.events-archive-card.is-previous {
  z-index: 2;
  opacity: 0.5;
  filter: blur(0.7px) saturate(0.82);
  transform: translate(-50%, -116%) translateZ(-170px) rotateX(44deg) scale(0.82);
  cursor: pointer;
}

.events-archive-card.is-next {
  z-index: 2;
  opacity: 0.5;
  filter: blur(0.7px) saturate(0.82);
  transform: translate(-50%, 16%) translateZ(-170px) rotateX(-44deg) scale(0.82);
  cursor: pointer;
}

.events-archive-card:focus-visible {
  outline: 2px solid rgba(139, 233, 236, 0.72);
  outline-offset: 4px;
}

.events-archive-date {
  color: #00979d;
  font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
  font-size: 11px;
  font-weight: 700;
  letter-spacing: 0.08em;
}

.events-archive-card h3 {
  margin-top: 16px;
  font-size: clamp(24px, 2.4vw, 34px);
}

.events-archive-card p {
  max-width: 560px;
  margin-top: 10px;
  color: #686a78;
  font-size: 14px;
  line-height: 1.75;
}

.events-archive-current {
  align-self: flex-start;
  margin-top: auto;
  border: 1px solid rgba(45, 127, 118, 0.16);
  border-radius: 999px;
  padding: 5px 10px;
  color: #2d7f76;
  font-size: 11px;
  font-weight: 700;
}

.events-archive-empty {
  border: 1px dashed rgba(45, 127, 118, 0.2);
  padding: 48px 24px;
  color: #7b7465;
  font-size: 14px;
  text-align: center;
}

@media (max-width: 820px) {
  .events-upcoming {
    padding-top: 8px;
  }

  .events-feature-card {
    grid-template-columns: 1fr;
  }

  .events-feature-side {
    border-right: 0;
    border-bottom: 1px solid rgba(0, 151, 157, 0.08);
  }

  .events-dark-flow {
    padding: 48px 0 64px;
  }

  .events-hero-layout {
    grid-template-columns: 1fr;
    gap: 16px;
  }

  .events-visual {
    height: 270px;
    min-height: 270px;
    width: 100%;
  }

  .events-visual img {
    width: min(100%, 420px);
  }

  .events-toolbar {
    grid-template-columns: 1fr auto;
  }

  .events-filters {
    grid-column: 1 / -1;
    grid-row: 2;
  }

  .events-calendar-head {
    align-items: flex-start;
    flex-direction: column;
  }

  .events-calendar-head p {
    max-width: 560px;
    text-align: left;
  }

  .events-calendar-tools {
    width: 100%;
    justify-content: space-between;
  }

  .events-event-card {
    min-height: 0;
  }

  .events-archive-head {
    align-items: flex-start;
    flex-direction: column;
  }

  .events-archive-tools {
    width: 100%;
    justify-content: space-between;
  }

  .events-archive-search {
    min-width: 0;
    flex: 1;
  }

  .events-archive-stage {
    height: 390px;
  }

  .events-archive-card {
    width: min(88%, 680px);
    height: 220px;
  }
}

@media (max-width: 560px) {
  .events-page {
    margin-top: 0;
  }

  .events-hero-card {
    border-radius: 22px;
    padding: 22px 18px;
  }

  .events-heading {
    gap: 10px;
  }

  .events-upcoming-copy {
    top: -20px;
  }

  .events-feature-content,
  .events-feature-side {
    padding: 24px 20px;
  }

  .events-latest {
    grid-template-columns: 1fr;
    gap: 16px;
  }

  .events-latest-list li {
    font-size: 17px;
    padding: 15px 2px;
  }

  .events-visual {
    height: 240px;
    min-height: 240px;
  }

  .events-visual img {
    width: min(100%, 360px);
  }

  .events-toolbar {
    grid-template-columns: 1fr;
    gap: 18px;
  }

  .events-calendar-tools {
    align-items: flex-start;
    flex-direction: column;
    gap: 14px;
  }

  .events-timeline-nav {
    align-self: flex-end;
  }

  .events-filters {
    grid-column: auto;
    grid-row: auto;
  }

  .events-search {
    justify-self: start;
  }

  .events-archive-head {
    align-items: flex-start;
    flex-direction: column;
    gap: 8px;
  }

  .events-archive-tools {
    align-items: stretch;
    flex-direction: column;
  }

  .events-archive-stage {
    height: 370px;
  }

  .events-archive-card {
    width: 96%;
    height: 230px;
    padding: 22px;
  }
}

:global(html.dark:has(.events-page)) {
  background: #071b1f;
  color-scheme: dark;
}

:global(.dark body:has(.events-page)) {
  --vp-c-bg: #071b1f;
  --vp-c-bg-alt: #0b262a;
  --vp-c-bg-soft: #102f33;
  --vp-c-text-1: #f0ffff;
  --vp-c-text-2: rgba(226, 255, 255, 0.68);
  --vp-c-text-3: rgba(226, 255, 255, 0.42);
  --vp-c-divider: rgba(139, 233, 236, 0.14);
  --vp-c-gutter: #071b1f;
  --vp-c-brand-1: #6ee5e8;
  --vp-c-brand-2: #43cbd0;
  --vp-c-brand-3: #23b8be;
  --vp-c-brand-soft: rgba(67, 203, 208, 0.14);
  --vp-nav-bg-color: #071b1f;
  --vp-nav-screen-bg-color: #071b1f;
  --vp-local-nav-bg-color: #071b1f;
  background:
    radial-gradient(circle at 15% 4%, rgba(0, 151, 157, 0.17), transparent 30%),
    linear-gradient(180deg, #081f23 0%, #0a282d 52%, #071b1f 100%);
  color: #f0ffff;
}

.dark .events-page {
  --events-border: rgba(139, 233, 236, 0.14);
  --events-surface: #102f33;
  --events-card: #0c262a;
  --events-text: #f0ffff;
  --events-muted: rgba(226, 255, 255, 0.66);
}

.dark .events-upcoming-kicker,
.dark .events-feature-label,
.dark .events-dark-flow .events-kicker,
.dark .events-calendar .events-kicker,
.dark .events-month-code,
.dark .events-archive-date {
  color: #8be9ec;
}

.dark .events-upcoming h1 {
  color: #6ee5e8;
  -webkit-text-fill-color: #6ee5e8;
}

.dark .events-feature-card {
  border-color: rgba(139, 233, 236, 0.12);
  background: #0c262a;
  box-shadow: 0 18px 48px rgba(0, 0, 0, 0.28);
}

.dark .events-feature-side {
  border-color: rgba(139, 233, 236, 0.1);
  background: #102f33;
}

.dark .events-feature-side h2,
.dark .events-placeholder-lines p,
.dark .events-calendar h2,
.dark .events-dark-flow h2,
.dark .events-dark-flow h3,
.dark .events-month-header h3,
.dark .events-archive h2 {
  color: #f0ffff;
}

.dark .events-placeholder-lines {
  border-color: rgba(139, 233, 236, 0.1);
}

.dark .events-placeholder-lines p {
  border-color: rgba(139, 233, 236, 0.1);
  color: rgba(226, 255, 255, 0.7);
}

.dark .events-calendar-head p,
.dark .events-scroll-label {
  color: rgba(226, 255, 255, 0.58);
}

.dark .events-timeline-nav button {
  border-color: rgba(139, 233, 236, 0.16);
  background: rgba(255, 255, 255, 0.04);
  color: #8be9ec;
}

.dark .events-timeline-nav button:hover:not(:disabled) {
  border-color: rgba(139, 233, 236, 0.36);
  background: rgba(255, 255, 255, 0.08);
  color: #ffffff;
}

.dark .events-month-header {
  border-color: rgba(139, 233, 236, 0.1);
  background: #102f33;
  color: #f0ffff;
  box-shadow: 0 12px 28px rgba(0, 0, 0, 0.16);
}

.dark .events-month-header > span:last-child {
  color: rgba(226, 255, 255, 0.45);
}

.dark .events-event-card {
  border-color: rgba(139, 233, 236, 0.1);
  background: #0c262a;
  color: #f0ffff;
  box-shadow: 0 10px 24px rgba(0, 0, 0, 0.18);
}

.dark .events-event-card:hover {
  box-shadow:
    0 0 0 1px rgba(139, 233, 236, 0.12),
    0 16px 30px rgba(0, 0, 0, 0.24);
}

.dark .events-card-date {
  color: rgba(226, 255, 255, 0.5);
}

.dark .events-event-card p {
  color: rgba(226, 255, 255, 0.62);
}

.dark .events-card-status {
  color: #8be9ec;
}

.dark .events-timeline-scrollbar {
  background: rgba(255, 255, 255, 0.07);
}

.dark .events-timeline-thumb {
  background: linear-gradient(90deg, #2a766b, #4f9c84);
}

.dark .events-archive-search {
  border-color: rgba(139, 233, 236, 0.12);
  background: #0c262a;
}

.dark .events-archive-search input {
  color: #f0ffff;
}

.dark .events-archive-search input::placeholder {
  color: rgba(226, 255, 255, 0.4);
}

.dark .events-archive-search svg {
  stroke: #8be9ec;
}

.dark .events-archive-card {
  border-color: rgba(139, 233, 236, 0.1);
  background: #0c262a;
  color: #f0ffff;
  box-shadow: 0 20px 46px rgba(0, 0, 0, 0.28);
}

.dark .events-archive-card.is-active {
  border-color: rgba(139, 233, 236, 0.3);
  box-shadow:
    0 0 0 1px rgba(139, 233, 236, 0.08),
    0 0 38px rgba(69, 208, 213, 0.12),
    0 28px 58px rgba(0, 0, 0, 0.32);
}

.dark .events-archive-card p {
  color: rgba(226, 255, 255, 0.62);
}

.dark .events-archive-current {
  border-color: rgba(139, 233, 236, 0.14);
  color: #8be9ec;
}

@media (prefers-reduced-motion: reduce) {
  :global(.vp-doc._events h1),
  .events-rule,
  .events-rule::before,
  .events-rule::after,
  .events-visual img,
  .events-hero-card .events-caret {
    animation: none;
  }

  .events-rule {
    transform: scaleX(1);
  }

  .events-rule::after {
    opacity: 1;
  }

  .events-dark-flow::after,
  .events-month-column {
    animation: none;
  }

  .events-latest-list li,
  .events-event-card,
  .events-archive-card {
    transition: none;
  }

  .events-latest-list li:hover,
  .events-event-card:hover {
    transform: none;
  }
}
</style>
