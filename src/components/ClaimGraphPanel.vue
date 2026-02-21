<script setup lang="ts">

import { onMounted, onBeforeUnmount, ref, computed } from 'vue'
import { Network } from 'vis-network'
import { useAgentStore } from '../store/agentStore'
import type { ExtractedRelation } from '../types/ExtractedRelation'


const lastDecisionType = ref<string | null>(null)

const flashColor = ref<string | null>(null)
const pulseScale = ref<number>(1)
const glowWidth = ref<number>(2)
const trailWidth = ref(2)
const trailOpacity = ref(1)

const container = ref()
const store = useAgentStore()
// 🔥 DominantDecision 响应式代理（避免 undefined）
const decision = computed(() => store.decision ?? null)

let network: any = null



function shakeAndFade(nodeId:string){

  if(!network) return

  // ⭐ 抖动动画
  network.focus(nodeId,{
    scale:1.35,
    animation:{
      duration:120,
      easingFunction:'easeInOutQuad'
    }
  })

  // ⭐ 再缩回
  setTimeout(()=>{
    network.focus(nodeId,{
      scale:0.9,
      animation:{
        duration:280,
        easingFunction:'easeInOutQuad'
      }
    })
  },150)

  // ⭐ 最终恢复正常
  setTimeout(()=>{
    network.focus(nodeId,{
      scale:1,
      animation:{
        duration:300,
        easingFunction:'easeInOutQuad'
      }
    })
  },450)
}




function focusDominant(nodeId:string){

  if(!network) return

  // v3: 先把目标节点“吸”到视图中心（position），再做一次轻微 focus（scale）
  const posMap = network.getPositions([nodeId]) as any
  const pos = posMap?.[nodeId]
  if (pos) {
    network.moveTo({
      position: pos,
      scale: 1.15,
      animation: {
        duration: 900,
        easingFunction: 'easeInOutQuad'
      }
    })
  }

  // 再轻微 focus，让它成为视觉中心
  network.focus(nodeId,{
    scale:1.25,
    animation:{
      duration:600,
      easingFunction:'easeInOutQuad'
    }
  })
}




type DecisionType = 'CONFIRMED' | 'OVERRIDDEN' | 'UNCERTAIN'
type DecisionLike = { type?: DecisionType | string }

// ===== Graph Motion Engine v2 (low-intrusion) =====
let rafId: number | null = null
let animStart = 0
const animDurationMs = 520 // ~0.5s 光轨拖尾
let activeColor: string | null = null

function startRaf() {
  if (rafId != null) return
  rafId = requestAnimationFrame(tick)
}

function stopRaf() {
  if (rafId == null) return
  cancelAnimationFrame(rafId)
  rafId = null
}

function lerp(a: number, b: number, t: number) {
  return a + (b - a) * t
}

// easeOutQuad: 前快后慢，像“脉冲退场”
function easeOut(t: number) {
  return 1 - (1 - t) * (1 - t)
}

function beginTrail(color: string) {
  activeColor = color
  animStart = performance.now()
  startRaf()
}

function endTrail() {
  activeColor = null
  flashColor.value = null
  trailWidth.value = 2
  trailOpacity.value = 1
  pulseScale.value = 1
}

function tick(now: number) {
  // 可能组件已卸载
  if (!container.value) {
    stopRaf()
    return
  }

  if (activeColor) {
    const tRaw = Math.min(1, (now - animStart) / animDurationMs)
    const t = easeOut(tRaw)

    // 光轨：宽度从 7 -> 2；透明度从 1 -> 0
    flashColor.value = activeColor
    trailWidth.value = lerp(7, 2, t)
    trailOpacity.value = lerp(1, 0, t)

    // Dominant pulse：1.35 -> 1.0
    pulseScale.value = lerp(1.35, 1.0, t)

    // 每帧只更新数据，不重建 network
    renderGraph()

    if (tRaw >= 1) {
      endTrail()
      renderGraph()
      stopRaf()
      return
    }

    rafId = requestAnimationFrame(tick)
    return
  }

  // 没有动画就停掉 RAF
  stopRaf()
}

function applyTransition(d: DecisionLike | null | undefined) {
  const type = (d?.type ?? null) as DecisionType | null
  if (!type) return

  // 状态没变，不触发动画
  if (lastDecisionType.value === type) return
  lastDecisionType.value = type

  // 颜色映射
  if (type === 'CONFIRMED') beginTrail('#3b82f6')
  else if (type === 'OVERRIDDEN') beginTrail('#f97316')
  else if (type === 'UNCERTAIN') beginTrail('#9ca3af')

  // 目标节点：当前 demo 只有 subject/object，两者里选 subject 作为 Dominant
  const sid = store.relation?.subjectId
  if (sid) {
    focusDominant(sid)
    shakeAndFade(sid)
  }
}


function renderGraph() {
  if (!container.value) return

  // ⭐ 从 store 读取 relation（后端可能返回 null，不要销毁 network）
  const r = store.relation as ExtractedRelation | null

  // 调试日志：联调阶段用于确认后端是否真的返回 relation
  console.log('[GraphPanel] relation =', r, 'decision =', decision.value)

  // 如果本次没有抽取到 relation，只跳过渲染，不 destroy graph
  if (!r) {
    return
  }

  const dominantId = r.subjectId
  const dimOthers = !!flashColor.value // 只有在迁移动画期间才弱化其它节点

  const nodes = [
    {
      id: r.subjectId,
      label: r.subjectId,
      size: decision.value?.type === 'CONFIRMED'
        ? 30 * pulseScale.value
        : 20,
      color: {
        background: '#bfdbfe',
        border: '#3b82f6'
      },
      font: {
        color: '#111827'
      }
    },
    {
      id: r.objectId,
      label: r.objectId,
      size: 20,
      color: dimOthers
        ? { background: '#f3f4f6', border: '#e5e7eb' }
        : { background: '#dbeafe', border: '#93c5fd' },
      font: dimOthers
        ? { color: '#9ca3af' }
        : { color: '#111827' }
    }
  ]

  // v3: 如果 future 扩展为多节点，这里保留 hook
  // （目前 demo 只有 subject/object，object 作为非 dominant）

  const edges = [
    {
      from: r.subjectId,
      to: r.objectId,
      label: r.predicateType,
      width: flashColor.value ? trailWidth.value : glowWidth.value,
      color: flashColor.value
        ? {
            color: flashColor.value,
            opacity: trailOpacity.value
          }
        : (r.polarity ? 'green' : 'red'),
      // v3: 第二层光轨（shadow），让边是“边沿发光”而不是纯颜色跳变
      shadow: flashColor.value
        ? {
            enabled: true,
            color: flashColor.value,
            size: Math.max(8, trailWidth.value * 2),
            x: 0,
            y: 0
          }
        : { enabled: false }
    }
  ]

  if (!network) {
    network = new Network(container.value, { nodes, edges }, {
      edges: {
        smooth: {
          type: 'dynamic'
        }
      },
      interaction: {
        hover: true
      }
    })
  } else {
    network.setData({ nodes, edges })
  }
}


let unsubscribe: (() => void) | null = null

onMounted(() => {
  // 先渲染一帧（如果已有 relation）
  renderGraph()

  // 初始 decision（如果后端有返回 type）
  applyTransition(decision.value as any)

  // 不用 watch：Pinia 原生订阅即可
  unsubscribe = store.$subscribe(() => {
    // relation / decision 任意变化都重绘（low cost：setData）
    renderGraph()
    applyTransition(decision.value as any)
  })
})

onBeforeUnmount(() => {
  try {
    unsubscribe?.()
  } catch (e) {
    // ignore
  }
  unsubscribe = null

  stopRaf()

  if (network) {
    network.destroy()
    network = null
  }
})


</script>

<template>
  <div ref="container" style="height:400px;border:1px solid #ccc;"></div>
</template>