<!-- <script setup lang="ts">
import { range } from '@antfu/utils'
import type { DataSet } from 'vis-data'
import type { DataItemCollectionType, TimelineOptions } from 'vis-timeline'
import { Timeline } from 'vis-timeline'
import type { TimelineOption } from 'echarts/types/dist/shared'
import { inspectSSR, onRefetch, root, waterfallShowResolveId } from '../../logic'
import { getHot } from '../../logic/hot'
import { rpc } from '../../logic/rpc'
import 'vis-timeline/styles/vis-timeline-graph2d.css'

const data = shallowRef(await rpc.getWaterfallInfo(inspectSSR.value))
const startTime = computed(() => Math.min(...Object.values(data.value).map(i => i[0]?.start ?? Infinity)))
const endTime = computed(() => Math.max(...Object.values(data.value).map(i => i[i.length - 1]?.end ?? -Infinity)) + 1000)
const scale = ref(0.3)

const searchText = ref('')
const searchFn = computed(() => {
  const text = searchText.value.trim()
  if (text === '') {
    return () => true
  }
  const regex = new RegExp(text, 'i')
  return (name: string) => regex.test(name)
})

const container = ref<HTMLElement>()
const { x: containerScrollX } = useScroll(container)
const { width: containerWidth } = useElementSize(container)
const visibleMin = computed(() => (containerScrollX.value - 500) / scale.value)
const visibleMax = computed(() => (containerScrollX.value + containerWidth.value + 500) / scale.value)
const tickNum = computed(() => range(
  Math.floor(Math.max(0, visibleMin.value) / 1000),
  Math.ceil(Math.min(endTime.value - startTime.value, visibleMax.value) / 1000),
))

interface WaterfallSpan {
  kind: keyof typeof classNames
  fade: boolean
  start: number
  end: number
  id: string
  name: string
}

function getHashColorFromStringWithPrefix(str: string) {
  const hash = str.split('').reduce((acc, char) => char.charCodeAt(0) + ((acc << 5) - acc), 0)
  const color = `hsl(${hash % 360}, 100%, 50%)`
  return color
}

const colorMap = computed(() => {
  const colorMap = new Set()

  for (const [id] of Object.entries(data.value)) {
    const color = getHashColorFromStringWithPrefix(id)

    colorMap.add(color)
  }

  console.log(Array.from(colorMap))

  return Array.from(colorMap)
})

const waterfallData = computed(() => {
  const result: WaterfallSpan[][] = []
  const rowsEnd: number[] = []

  console.log('origin data', data.value)

  for (let [id, steps] of Object.entries(data.value)) {
    if (!waterfallShowResolveId.value) {
      steps = steps.filter(i => !i.isResolveId)
    }
    if (steps.length === 0) {
      continue
    }
    const start = steps[0].start - startTime.value
    const end = steps[steps.length - 1].end - startTime.value
    const spans: WaterfallSpan[] = steps.map(v => ({
      kind: v.isResolveId ? 'resolve' : 'transform',
      start: v.start - startTime.value,
      end: v.end - startTime.value,
      name: v.name,
      id,
      fade: !searchFn.value(v.name) && !searchFn.value(id),
    }))
    const groupSpan: WaterfallSpan = {
      kind: 'group',
      fade: spans.every(i => i.fade),
      start,
      end,
      id,
      name: 'group',
    }
    spans.push(groupSpan)
    const isEnd = rowsEnd.findIndex((rowEnd, i) => {
      if (rowEnd <= start) {
        rowsEnd[i] = end
        result[i].push(...spans)
        return true
      }
      return false
    })
    if (isEnd === -1) {
      result.push(spans)
      rowsEnd.push(end)
    }
  }

  console.log(result)

  return result
})

const timelineData = computed(() => {
  const result: DataItemCollectionType[] = []

  for (let [id, steps] of Object.entries(data.value)) {
    if (!waterfallShowResolveId.value) {
      steps = steps.filter(i => !i.isResolveId)
    }
    if (steps.length === 0) {
      continue
    }
    const start = steps[0].start - startTime.value
    const end = steps[steps.length - 1].end - startTime.value
    // const spans: WaterfallSpan[] = steps.map(v => ({
    //   kind: v.isResolveId ? 'resolve' : 'transform',
    //   start: v.start - startTime.value,
    //   end: v.end - startTime.value,
    //   name: v.name,
    //   id,
    //   fade: !searchFn.value(v.name) && !searchFn.value(id),
    // }))
    // const groupSpan: WaterfallSpan = {
    //   kind: 'group',
    //   fade: spans.every(i => i.fade),
    //   start,
    //   end,
    //   id,
    //   name: 'group',
    // }
    // spans.push(groupSpan)
    // const isEnd = rowsEnd.findIndex((rowEnd, i) => {
    //   if (rowEnd <= start) {
    //     rowsEnd[i] = end
    //     result[i].push(...spans)
    //     return true
    //   }
    //   return false
    // })
    // if (isEnd === -1) {
    //   result.push(spans)
    //   rowsEnd.push(end)
    // }

    // const generateColor = (name: string) => {
    //   const hash = name.split('').reduce((acc, char) => char.charCodeAt(0) + ((acc << 5) - acc), 0)
    //   const color = `hsl(${hash % 360}, 100%, 50%)`
    //   return color
    // }

    const spans = steps.map(i => ({
      id: id + i.name + Math.random(),
      content: i.name,
      start: i.start,
      end: i.end,
      title: `
        <div>
          <div>${i.name}</div>
          <div>${i.end - i.start}ms</div>
        <div>
      `,
      // className: getPluginColorHash(id),
    } as unknown as DataItemCollectionType))

    result.push(...spans)
  }

  return result
})

onMounted(() => {
  const options: TimelineOptions = {
    height: '80vh',
    tooltip: {
      followMouse: true,
    },
    // max: 1000 * 60 * 60 * 2,
    min: startTime.value,
    max: endTime.value,
    format: {
      minorLabels: {
        millisecond: 'SSS',
        second: 's',
        minute: 'HH:mm',
        hour: 'HH:mm',
        weekday: 'ddd D',
        day: 'D',
        month: 'MMM',
        year: 'YYYY',
      },
    },
  }

  // const timeline = new Timeline(container.value!, timelineData.value, options)
})

async function refetch() {
  data.value = await rpc.getWaterfallInfo(inspectSSR.value)
}

onRefetch.on(refetch)
watch(inspectSSR, refetch)

getHot().then((hot) => {
  if (hot) {
    hot.on('vite-plugin-inspect:update', refetch)
  }
})

function getPositionStyle(start: number, end: number) {
  return {
    left: `${start * scale.value}px`,
    width: `${Math.max((end - start) * scale.value, 1)}px`,
  }
}

const classNames = {
  resolve: 'outline-red-200 outline-offset--1 bg-gray-300 dark:bg-gray-500 bg-op-80 z-1',
  transform: 'outline-blue-200 outline-offset--1 bg-gray-300 dark:bg-gray-500 bg-op-80 z-1',
  group: 'outline-orange-700 dark:outline-orange-200',
}

watch(scale, (newScale, oldScale) => {
  containerScrollX.value = (containerScrollX.value - 40) * newScale / oldScale + 40
})
</script> -->

<!-- <template>
  <NavBar>
    <RouterLink class="my-auto icon-btn !outline-none" to="/">
      <div i-carbon-arrow-left />
    </RouterLink>
    <div my-auto text-sm font-mono>
      Waterfall
    </div>
    <input v-model="searchText" placeholder="Search..." class="w-full px-4 py-2 text-xs">

    <button text-lg icon-btn title="Inspect SSR" @click="inspectSSR = !inspectSSR">
      <div i-carbon-cloud-services :class="inspectSSR ? 'opacity-100' : 'opacity-25'" />
    </button>
    <button class="text-lg icon-btn" title="Show resolveId" @click="waterfallShowResolveId = !waterfallShowResolveId">
      <div i-carbon-connect-source :class="waterfallShowResolveId ? 'opacity-100' : 'opacity-25'" />
    </button>
    <button text-lg icon-btn title="Zoom in" @click="scale += 0.1">
      <div i-carbon-zoom-in />
    </button>
    <button text-lg icon-btn title="Zoom in" :disabled="scale <= 0.11" @click="scale -= 0.1">
      <div i-carbon-zoom-out />
    </button>
    <div flex-auto />
  </NavBar>
  <Container of-auto :class="colorMap">
    <div relative m-4 w-full flex flex-col gap-1 pb-2 pt-8>
      <div ref="container" />

      <pre hidden>
        {{ waterfallData }}
      </pre>
    </div>
  </Container>
</template>

<style scoped>

</style> -->
