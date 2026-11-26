# 项目展示

<script setup>
import { ref, computed } from 'vue'

const active = ref(null)
const order = ref('desc')

const teams = {
  sw: { label: '软件组', en: 'SOFTWARE', },
  hw: { label: '硬件组', en: 'HARDWARE', }
}

const projects = {
  sw: [
    { id: 'sw-1', name: '华南理工大学维基百科', time: '2026届', brief: '简介【待补充】' },
  ],
  hw: [
    { id: 'hw-1', name: '电车实验箱', time: '2026届', brief: '简介【待补充】' },
  ]
}

const sorted = computed(() => {
  const list = [...projects[active.value || 'sw']]
  return list.sort((a, b) =>
    order.value === 'desc' ? b.time.localeCompare(a.time) : a.time.localeCompare(b.time)
  )
})

function pick(key) {
  active.value = active.value === key ? null : key
}

function toggleOrder() {
  order.value = order.value === 'desc' ? 'asc' : 'desc'
}
</script>

<div class="board" :class="active ? 'is-open is-' + active : ''">
  <button
    v-show="active !== 'hw'"
    class="team team-sw"
    :class="{ 'is-active': active === 'sw' }"
    type="button"
    @click="pick('sw')">
    <span class="team-en">{{ teams.sw.en }}</span>
    <span class="team-name">{{ teams.sw.label }}</span>
    <span class="team-desc">{{ teams.sw.desc }}</span>
    <span class="team-cta">{{ active === 'sw' ? '点击返回' : '查看项目 →' }}</span>
  </button>
  <div v-show="active" class="list">
    <div class="list-head">
      <span class="list-title">{{ active === 'sw' ? '软件组项目' : '硬件组项目' }}</span>
      <button
        class="sort"
        type="button"
        @click="toggleOrder"
        :aria-label="order === 'desc' ? '按时间倒序，点击切换为正序' : '按时间正序，点击切换为倒序'">
        <span class="sort-tri">{{ order === 'desc' ? '▼' : '▲' }}</span>
        <span>时间</span>
      </button>
    </div>
    <div class="bars" :key="active + order">
      <div v-for="(p, i) in sorted" :key="p.id" class="bar" :style="{ '--i': i }" tabindex="0">
        <div class="bar-inner">
          <div class="bar-face bar-front">
            <span class="bar-name">{{ p.name }}</span>
            <span class="bar-time">{{ p.time }}</span>
            <span class="bar-mobile-brief">{{ p.brief }}</span>
          </div>
          <div class="bar-face bar-back">
            <span class="bar-brief">{{ p.brief }}</span>
          </div>
        </div>
      </div>
    </div>
  </div>
  <button
    v-show="active !== 'sw'"
    class="team team-hw"
    :class="{ 'is-active': active === 'hw' }"
    type="button"
    @click="pick('hw')">
    <span class="team-en">{{ teams.hw.en }}</span>
    <span class="team-name">{{ teams.hw.label }}</span>
    <span class="team-desc">{{ teams.hw.desc }}</span>
    <span class="team-cta">{{ active === 'hw' ? '点击返回' : '查看项目 →' }}</span>
  </button>
</div>

##### 更多

- [加入联系](/contact)
- [学习路线](/learn)
- [活动公告](/events)

<style scoped>
.board-lead {
  margin: -4px 0 24px;
  color: var(--vp-c-text-2);
}

.board {
  /* 左边缘与正文（含标题）对齐，向右铺到距屏幕右缘约 32px 处 */
  width: 100%;
  /* 面板向右伸出后会压到右侧目录栏，那里有个 fixed 的空遮罩会吞掉点击，
     抬高层级让它盖在遮罩之上（仍低于导航栏的 30 和侧边栏的 25） */
  position: relative;
  z-index: 11;
  display: flex;
  gap: 20px;
  align-items: stretch;
  min-height: clamp(440px, 62vh, 660px);
  margin-top: 8px;
  margin-bottom: 48px;
}

/* 960~1279px：没有右侧目录栏，正文两侧各留 32px 内边距 */
@media (min-width: 960px) {
  .board {
    width: calc(100% + 50vw - 512px);
  }
}

/* ≥1280px：右侧出现 256px 目录栏，正文左移，需再放宽 128px */
@media (min-width: 1280px) {
  .board {
    width: calc(100% + 50vw - 384px);
  }
}

.team {
  flex: 1 1 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 38px;
  min-width: 0;
  border: 1px solid var(--vp-c-divider);
  border-radius: 20px;
  /* 与首页功能卡片同一套渐变，另叠一层极淡的品牌色 */
  background:
    radial-gradient(circle at 18% 10%, rgba(0, 151, 157, 0.08), transparent 46%),
    linear-gradient(155deg, var(--vp-c-bg-soft), var(--vp-c-bg));
  font-family: inherit;
  text-align: center;
  cursor: pointer;
  transition:
    border-color 0.25s ease,
    transform 0.25s ease,
    flex-basis 0.45s cubic-bezier(0.4, 0.2, 0.2, 1);
}

/* 硬件组用首页主色渐变的另一端（橙），两屏之间呼应 */
.team-hw {
  background:
    radial-gradient(circle at 82% 10%, rgba(245, 166, 35, 0.08), transparent 46%),
    linear-gradient(155deg, var(--vp-c-bg-soft), var(--vp-c-bg));
}

.team:hover {
  border-color: var(--vp-c-brand-2);
  transform: translateY(-3px);
}

.team-sw.is-active {
  order: 1;
  flex: 0 0 110px;
  align-items: center;
  justify-content: center;
  padding: 28px 14px;
  background: var(--vp-c-brand-soft);
  border-color: var(--vp-c-brand-2);
}

.team-hw.is-active {
  order: 2;
  flex: 0 0 110px;
  align-items: center;
  justify-content: center;
  padding: 28px 14px;
  background: var(--vp-c-brand-soft);
  border-color: var(--vp-c-brand-2);
}

.board.is-sw .list {
  order: 2;
}

.board.is-hw .list {
  order: 1;
}

.team-en {
  font-size: 12px;
  font-weight: 500;
  letter-spacing: 0.2em;
  color: var(--vp-c-text-3);
}

.team-name {
  font-size: clamp(40px, 4vw, 80px);
  line-height: 1.1;
  font-weight: 600;
  letter-spacing: 0.04em;
  /* 与首页主标题同一支品牌绿，暗色模式自动切到亮青 */
  color: var(--vp-c-brand-1);
}

.team-desc {
  margin-top: 8px;
  font-size: 15px;
  line-height: 1.6;
  color: var(--vp-c-text-2);
}

.team-cta {
  margin-top: 22px;
  font-size: 15px;
  font-weight: 500;
  color: var(--vp-c-brand-1);
}

.team.is-active .team-en,
.team.is-active .team-desc {
  display: none;
}

.team.is-active .team-name {
  writing-mode: vertical-rl;
  letter-spacing: 0.3em;
  font-size: 22px;
}

.team.is-active .team-cta {
  margin-top: 16px;
  font-size: 12px;
  color: var(--vp-c-text-3);
  writing-mode: vertical-rl;
  letter-spacing: 0.2em;
}

.team.is-active:hover {
  transform: none;
  background: var(--vp-c-brand-1);
}

.team.is-active:hover .team-name,
.team.is-active:hover .team-cta {
  color: #fff;
}

.list {
  flex: 1 1 0;
  min-width: 0;
  display: flex;
  flex-direction: column;
}

.list-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  margin-bottom: 18px;
}

.list-title {
  font-size: 14px;
  font-weight: 500;
  color: var(--vp-c-text-2);
}

.sort {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 6px 10px;
  border: none;
  border-radius: 6px;
  background: transparent;
  font-family: inherit;
  font-size: 13px;
  color: var(--vp-c-text-3);
  cursor: pointer;
  transition: color 0.2s ease, background-color 0.2s ease;
}

.sort:hover {
  color: var(--vp-c-brand-1);
  background: var(--vp-c-brand-soft);
}

.sort-tri {
  font-size: 10px;
  line-height: 1;
}

.bars {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.bar {
  height: 108px;
  perspective: 900px;
  outline: none;
  animation: rise 0.45s cubic-bezier(0.2, 0.7, 0.3, 1) both;
  animation-delay: calc(var(--i) * 70ms);
}

@keyframes rise {
  from {
    opacity: 0;
    transform: translateY(16px) scale(0.985);
  }
  to {
    opacity: 1;
    transform: none;
  }
}

.bar-inner {
  position: relative;
  height: 100%;
  transform-style: preserve-3d;
  transition: transform 0.55s cubic-bezier(0.4, 0.2, 0.2, 1);
}

.bar:hover .bar-inner,
.bar:focus-visible .bar-inner {
  transform: rotateY(180deg);
}

.bar-face {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  padding: 0 28px;
  border: 1px solid var(--vp-c-divider);
  border-radius: 14px;
  background: var(--vp-c-bg-soft);
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
}

.bar-front {
  justify-content: space-between;
  gap: 16px;
}

.bar:hover .bar-front {
  border-color: var(--vp-c-brand-2);
}

.bar-name {
  font-size: 17px;
  font-weight: 500;
  color: var(--vp-c-text-1);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.bar-time {
  flex: none;
  font-size: 13px;
  color: var(--vp-c-text-3);
  font-variant-numeric: tabular-nums;
}

.bar-back {
  transform: rotateY(180deg);
  border-color: var(--vp-c-brand-2);
  background: var(--vp-c-brand-soft);
}

.bar-brief {
  display: block;
  font-size: 14px;
  line-height: 1.6;
  color: var(--vp-c-text-1);
  overflow: hidden;
}

.bar-mobile-brief {
  display: none;
}

@media (hover: none) {
  .bar {
    height: auto;
  }

  .bar-inner {
    transform: none !important;
  }

  .bar-face {
    position: static;
    backface-visibility: visible;
  }

  .bar-front {
    flex-wrap: wrap;
    padding: 14px 18px;
  }

  .bar-mobile-brief {
    display: block;
    width: 100%;
    margin-top: 6px;
    font-size: 12px;
    line-height: 1.6;
    color: var(--vp-c-text-2);
  }

  .bar-back {
    display: none;
  }
}

@media (max-width: 767px) {
  .board {
    width: 100%;
    flex-direction: column;
    gap: 14px;
    min-height: 0;
  }

  .team {
    min-height: 168px;
    padding: 28px;
  }

  .team-name {
    font-size: 44px;
  }

  .team-sw.is-active,
  .team-hw.is-active {
    flex: 0 0 auto;
    min-height: 0;
    padding: 14px 18px;
    align-items: flex-start;
    justify-content: center;
  }

  .team.is-active .team-name {
    writing-mode: horizontal-tb;
    letter-spacing: 0;
    font-size: 15px;
  }

  .team.is-active .team-cta {
    margin-top: 2px;
    writing-mode: horizontal-tb;
  }
}

@media (prefers-reduced-motion: reduce) {
  .bar {
    animation: none;
  }

  .bar-inner,
  .team {
    transition: none;
  }
}
</style>
