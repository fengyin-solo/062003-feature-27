<template>
  <div class="relation-panel card">
    <div class="panel-header">
      <h3>🤝 练习生关系</h3>
      <div class="tabs">
        <button
          v-for="tab in tabs"
          :key="tab.key"
          class="tab-btn"
          :class="{ active: activeTab === tab.key }"
          @click="activeTab = tab.key"
        >
          {{ tab.label }}
          <span v-if="tab.count > 0" class="badge">{{ tab.count }}</span>
        </button>
      </div>
    </div>

    <div v-if="activeTab === 'all'" class="rel-list">
      <div
        v-for="pair in sortedPairs"
        :key="pair.key"
        class="rel-row"
        :class="{
          highlight: pair.isHighlight,
          critical: pair.isCritical,
          'has-change': pair.recentChange !== 0,
        }"
        @mouseenter="hoveredPair = pair.key"
        @mouseleave="hoveredPair = null"
      >
        <div class="rel-left">
          <span class="names">{{ pair.a }} ↔ {{ pair.b }}</span>
          <div v-if="hoveredPair === pair.key && pair.history.length > 0" class="history-tip">
            <div v-for="h in pair.history.slice(-3)" :key="h.day" class="history-item">
              <span class="history-icon">{{ h.sourceInfo.icon }}</span>
              <span class="history-label">{{ h.sourceInfo.label }}</span>
              <span class="history-delta" :class="h.delta > 0 ? 'up' : 'down'">
                {{ h.delta > 0 ? '+' : '' }}{{ h.delta }}
              </span>
              <span class="history-day">第{{ h.day }}天</span>
            </div>
          </div>
        </div>
        <div class="rel-bar-wrap">
          <div class="rel-bar">
            <div
              class="rel-fill"
              :class="pair.type"
              :style="{ width: pair.barWidth + '%', marginLeft: pair.barOffset + '%' }"
            ></div>
          </div>
          <div class="rel-val-wrap">
            <span class="rel-val" :class="pair.type">{{ pair.value }}</span>
            <span
              v-if="pair.recentChange !== 0"
              class="rel-change"
              :class="pair.recentChange > 0 ? 'up' : 'down'"
            >
              {{ pair.recentChange > 0 ? '▲' : '▼' }}{{ Math.abs(pair.recentChange) }}
            </span>
          </div>
        </div>
        <div class="rel-right">
          <span class="rel-tag" :class="pair.type">{{ pair.tag }}</span>
          <span v-if="pair.isHighlight" class="highlight-icon">⭐</span>
          <span v-if="pair.isCritical" class="critical-icon">⚠️</span>
        </div>
      </div>
    </div>

    <div v-else-if="activeTab === 'suggestions'" class="suggestions-section">
      <p v-if="debutSuggestions.length === 0" class="empty-tip">
        暂无达标练习生，继续培养吧！
      </p>
      <div v-else class="sug-list">
        <div
          v-for="(sug, idx) in debutSuggestions"
          :key="idx"
          class="sug-card"
          :class="'grade-' + sug.grade.toLowerCase()"
        >
          <div class="sug-header">
            <span class="sug-grade" :class="'grade-' + sug.grade.toLowerCase()">{{ sug.grade }}</span>
            <span class="sug-size">{{ sug.size }}人组</span>
            <span class="sug-synergy" :title="`平均关系值: ${sug.synergyAvg}`">
              默契 {{ sug.synergyAvg }}
            </span>
          </div>
          <div class="sug-members">
            <span
              v-for="m in sug.members"
              :key="m.id"
              class="sug-member"
              :title="`综合评分: ${m.score}`"
            >
              {{ m.name }}
              <span class="member-score">{{ m.score }}</span>
            </span>
          </div>
          <div class="sug-stats">
            <span>均分 {{ sug.avgScore }}</span>
            <span>最低关系 {{ sug.synergyMin }}</span>
          </div>
          <button class="btn primary sm" @click="$emit('quick-debut', sug.members.map(m => m.id))">
            🚀 快速出道
          </button>
        </div>
      </div>
    </div>

    <div v-if="activeTab === 'all' && (highlightPairs.length > 0 || criticalPairs.length > 0)" class="legend">
      <div v-if="highlightPairs.length > 0" class="legend-item highlight">
        <span class="legend-icon">⭐</span>
        <span>重点培养 (≥{{ CFG.highlightThreshold }})</span>
      </div>
      <div v-if="criticalPairs.length > 0" class="legend-item critical">
        <span class="legend-icon">⚠️</span>
        <span>关系危险 (≤{{ CFG.criticalThreshold }})</span>
      </div>
      <div class="legend-item">
        <span class="legend-icon">▲▼</span>
        <span>近期变化</span>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import { GAME_CONFIG } from '../config/gameConfig'
import { pairKey } from '../utils/random'

const props = defineProps({
  trainees: Array,
  relationships: Object,
  relationshipHistory: Object,
  debutSuggestions: Array,
  getRelHistory: Function,
})

defineEmits(['quick-debut'])

const CFG = GAME_CONFIG.relationships
const activeTab = ref('all')
const hoveredPair = ref(null)

const tabs = computed(() => [
  { key: 'all', label: '全部关系', count: 0 },
  { key: 'suggestions', label: '出道建议', count: props.debutSuggestions?.length || 0 },
])

const pairs = computed(() => {
  const active = props.trainees.filter((t) => t.status !== 'left')
  const result = []
  for (let i = 0; i < active.length; i++) {
    for (let j = i + 1; j < active.length; j++) {
      const idA = active[i].id
      const idB = active[j].id
      const value = props.relationships[pairKey(idA, idB)] ?? 0
      let type = 'neutral'
      let tag = '普通'
      if (value >= CFG.synergyThreshold) { type = 'synergy'; tag = '默契' }
      else if (value <= CFG.competitionThreshold) { type = 'competition'; tag = '竞争' }

      const normalized = (value - CFG.min) / (CFG.max - CFG.min) * 100
      const barWidth = Math.abs(value) / CFG.max * 50
      const barOffset = value < 0 ? 50 - barWidth : 50

      const history = props.getRelHistory
        ? props.getRelHistory(idA, idB)
        : (props.relationshipHistory?.[pairKey(idA, idB)] || [])

      const recentChange = history.length > 0
        ? history.slice(-Math.min(3, history.length)).reduce((s, h) => s + h.delta, 0)
        : 0

      const historyWithSource = history.map((h) => ({
        ...h,
        sourceInfo: CFG.sources[h.source] || { label: h.source, icon: '•' },
      }))

      result.push({
        key: pairKey(idA, idB),
        a: active[i].name,
        b: active[j].name,
        value,
        type,
        tag,
        barWidth: Math.max(barWidth, 2),
        barOffset,
        normalized,
        isHighlight: value >= CFG.highlightThreshold,
        isCritical: value <= CFG.criticalThreshold,
        recentChange,
        history: historyWithSource,
      })
    }
  }
  return result
})

const sortedPairs = computed(() => {
  return [...pairs.value].sort((a, b) => {
    if (a.isHighlight !== b.isHighlight) return a.isHighlight ? -1 : 1
    if (a.isCritical !== b.isCritical) return a.isCritical ? -1 : 1
    return Math.abs(b.value) - Math.abs(a.value)
  })
})

const highlightPairs = computed(() => pairs.value.filter((p) => p.isHighlight))
const criticalPairs = computed(() => pairs.value.filter((p) => p.isCritical))
</script>

<style scoped>
.relation-panel h3 { margin-bottom: 0.75rem; }

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.panel-header h3 { margin: 0; }

.tabs {
  display: flex;
  gap: 0.25rem;
  background: var(--bg-secondary);
  padding: 0.2rem;
  border-radius: 8px;
}

.tab-btn {
  padding: 0.35rem 0.7rem;
  border: none;
  background: transparent;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.8rem;
  color: var(--text-secondary);
  display: flex;
  align-items: center;
  gap: 0.3rem;
  transition: all 0.2s;
}

.tab-btn.active {
  background: var(--accent);
  color: white;
}

.tab-btn .badge {
  background: rgba(255, 255, 255, 0.2);
  padding: 0.05rem 0.4rem;
  border-radius: 10px;
  font-size: 0.7rem;
}

.rel-list {
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
}

.rel-row {
  display: grid;
  grid-template-columns: 130px 1fr auto;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.85rem;
  padding: 0.4rem;
  border-radius: 8px;
  transition: all 0.2s;
  position: relative;
}

.rel-row:hover {
  background: var(--bg-secondary);
}

.rel-row.highlight {
  background: linear-gradient(90deg, var(--success-soft) 0%, transparent 100%);
  border-left: 3px solid var(--success);
}

.rel-row.critical {
  background: linear-gradient(90deg, var(--danger-soft) 0%, transparent 100%);
  border-left: 3px solid var(--danger);
}

.rel-row.has-change .rel-val-wrap {
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.7; }
}

.rel-left {
  position: relative;
}

.names { color: var(--text-secondary); }

.history-tip {
  position: absolute;
  left: 0;
  top: 100%;
  z-index: 10;
  background: var(--bg-card);
  border: 1px solid var(--border);
  border-radius: 8px;
  padding: 0.5rem;
  box-shadow: var(--shadow);
  min-width: 200px;
  margin-top: 0.25rem;
}

.history-item {
  display: flex;
  align-items: center;
  gap: 0.4rem;
  padding: 0.25rem 0;
  font-size: 0.75rem;
}

.history-icon { font-size: 0.9rem; }
.history-label { flex: 1; color: var(--text-secondary); }
.history-delta { font-weight: 600; }
.history-delta.up { color: var(--success); }
.history-delta.down { color: var(--danger); }
.history-day { color: var(--text-muted); font-size: 0.7rem; }

.rel-bar-wrap {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.rel-bar {
  flex: 1;
  height: 8px;
  background: var(--bg-secondary);
  border-radius: 4px;
  position: relative;
  overflow: hidden;
}

.rel-bar::after {
  content: '';
  position: absolute;
  left: 50%;
  top: 0;
  bottom: 0;
  width: 1px;
  background: var(--border);
}

.rel-fill {
  height: 100%;
  border-radius: 4px;
  transition: width 0.3s;
}

.rel-fill.synergy { background: var(--success); }
.rel-fill.competition { background: var(--danger); }
.rel-fill.neutral { background: var(--text-muted); }

.rel-val-wrap {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  min-width: 50px;
}

.rel-val { text-align: right; font-weight: 600; }
.rel-val.synergy { color: var(--success); }
.rel-val.competition { color: var(--danger); }

.rel-change {
  font-size: 0.7rem;
  font-weight: 600;
}
.rel-change.up { color: var(--success); }
.rel-change.down { color: var(--danger); }

.rel-right {
  display: flex;
  align-items: center;
  gap: 0.3rem;
}

.rel-tag {
  font-size: 0.7rem;
  padding: 0.1rem 0.4rem;
  border-radius: 4px;
  background: var(--bg-secondary);
}

.rel-tag.synergy { background: var(--success-soft); color: var(--success); }
.rel-tag.competition { background: var(--danger-soft); color: var(--danger); }

.highlight-icon, .critical-icon {
  font-size: 0.85rem;
  animation: bounce 1s infinite;
}

@keyframes bounce {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-2px); }
}

.legend {
  display: flex;
  gap: 1rem;
  margin-top: 0.75rem;
  padding-top: 0.75rem;
  border-top: 1px solid var(--border);
  flex-wrap: wrap;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 0.3rem;
  font-size: 0.75rem;
  color: var(--text-secondary);
}

.legend-icon { font-size: 0.9rem; }

.suggestions-section {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.empty-tip {
  text-align: center;
  color: var(--text-muted);
  padding: 2rem 1rem;
  font-size: 0.9rem;
}

.sug-list {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.sug-card {
  padding: 0.75rem;
  border-radius: 10px;
  border: 1px solid var(--border);
  background: var(--bg-secondary);
  transition: all 0.2s;
}

.sug-card:hover {
  transform: translateY(-2px);
  box-shadow: var(--shadow);
}

.sug-card.grade-s {
  border-color: var(--success);
  background: linear-gradient(135deg, var(--success-soft) 0%, var(--bg-secondary) 100%);
}

.sug-card.grade-a {
  border-color: var(--accent);
  background: linear-gradient(135deg, var(--accent-soft) 0%, var(--bg-secondary) 100%);
}

.sug-card.grade-b {
  border-color: #f59e0b;
}

.sug-header {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin-bottom: 0.5rem;
}

.sug-grade {
  font-size: 1.1rem;
  font-weight: 800;
  padding: 0.1rem 0.5rem;
  border-radius: 6px;
  background: var(--bg-card);
}

.sug-grade.grade-s { color: var(--success); }
.sug-grade.grade-a { color: var(--accent); }
.sug-grade.grade-b { color: #f59e0b; }
.sug-grade.grade-c { color: var(--text-muted); }

.sug-size {
  font-size: 0.8rem;
  color: var(--text-secondary);
  background: var(--bg-card);
  padding: 0.15rem 0.4rem;
  border-radius: 4px;
}

.sug-synergy {
  margin-left: auto;
  font-size: 0.8rem;
  color: var(--success);
  font-weight: 600;
}

.sug-members {
  display: flex;
  flex-wrap: wrap;
  gap: 0.4rem;
  margin-bottom: 0.5rem;
}

.sug-member {
  display: flex;
  align-items: center;
  gap: 0.25rem;
  padding: 0.25rem 0.5rem;
  background: var(--bg-card);
  border-radius: 6px;
  font-size: 0.8rem;
}

.member-score {
  font-size: 0.7rem;
  color: var(--text-muted);
  font-weight: 600;
}

.sug-stats {
  display: flex;
  gap: 1rem;
  font-size: 0.75rem;
  color: var(--text-secondary);
  margin-bottom: 0.5rem;
}

.btn.sm {
  width: 100%;
  padding: 0.4rem 0.75rem;
  font-size: 0.85rem;
}

@media (max-width: 600px) {
  .rel-row {
    grid-template-columns: 110px 1fr auto;
  }
  .names { font-size: 0.75rem; }
  .history-tip {
    position: fixed;
    left: 1rem;
    right: 1rem;
    max-width: none;
  }
}
</style>
