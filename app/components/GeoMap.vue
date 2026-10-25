<script setup lang="ts">
/* eslint-disable @typescript-eslint/no-explicit-any, better-tailwindcss/no-unknown-classes */
import type { FeatureCollection } from 'geojson'

import 'maplibre-gl/dist/maplibre-gl.css'
import '@mapbox/mapbox-gl-draw/dist/mapbox-gl-draw.css'

const props = defineProps<{
  modelValue: FeatureCollection | null
}>()

const emit = defineEmits<{
  'update:modelValue': [value: FeatureCollection | null]
}>()

const mapElement = ref<HTMLElement>()
const map = shallowRef<any>()
const draw = shallowRef<any>()
const activeTool = ref<'polygon' | 'circle' | 'rectangle' | 'square' | null>(null)
let shapeStart: [number, number] | null = null

const tools = [
  { id: 'polygon', icon: 'i-lucide-pentagon' },
  { id: 'circle', icon: 'i-lucide-circle' },
  { id: 'rectangle', icon: 'i-lucide-square-dashed' },
  { id: 'square', icon: 'i-lucide-square' }
] as const

function renderGeoJson(data: FeatureCollection | null) {
  if (!map.value || !map.value.isStyleLoaded()) return
  const source = map.value.getSource('user-data')
  if (source) {
    source.setData(data || { type: 'FeatureCollection', features: [] })
  } else {
    map.value.addSource('user-data', {
      type: 'geojson',
      data: data || { type: 'FeatureCollection', features: [] }
    })
    map.value.addLayer({
      id: 'user-fill',
      type: 'fill',
      source: 'user-data',
      paint: { 'fill-color': '#8b5cf6', 'fill-opacity': 0.22 }
    })
    map.value.addLayer({
      id: 'user-line',
      type: 'line',
      source: 'user-data',
      paint: { 'line-color': '#a78bfa', 'line-width': 2.5 }
    })
    map.value.addLayer({
      id: 'user-point',
      type: 'circle',
      source: 'user-data',
      paint: { 'circle-color': '#c4b5fd', 'circle-radius': 5, 'circle-stroke-color': '#7c3aed', 'circle-stroke-width': 2 }
    })
  }
}

function emitDrawn() {
  const collection = draw.value?.getAll() as FeatureCollection
  emit('update:modelValue', collection.features.length ? collection : null)
  renderGeoJson(collection.features.length ? collection : null)
}

function createBox(start: [number, number], end: [number, number], square: boolean) {
  let east = end[0]
  let north = end[1]
  if (square) {
    const dx = Math.abs(end[0] - start[0])
    const dy = Math.abs(end[1] - start[1])
    const size = Math.max(dx, dy)
    east = start[0] + (end[0] >= start[0] ? size : -size)
    north = start[1] + (end[1] >= start[1] ? size : -size)
  }
  return {
    type: 'Feature',
    properties: {},
    geometry: {
      type: 'Polygon',
      coordinates: [[
        [start[0], start[1]], [east, start[1]], [east, north],
        [start[0], north], [start[0], start[1]]
      ]]
    }
  }
}

function createCircle(center: [number, number], edge: [number, number]) {
  const radius = Math.hypot(edge[0] - center[0], edge[1] - center[1])
  const coordinates = Array.from({ length: 65 }, (_, index) => {
    const angle = (index / 64) * Math.PI * 2
    return [center[0] + Math.cos(angle) * radius, center[1] + Math.sin(angle) * radius]
  })
  return { type: 'Feature', properties: {}, geometry: { type: 'Polygon', coordinates: [coordinates] } }
}

function onMapClick(event: any) {
  if (!activeTool.value || activeTool.value === 'polygon') return
  const point: [number, number] = [event.lngLat.lng, event.lngLat.lat]
  if (!shapeStart) {
    shapeStart = point
    return
  }
  const feature = activeTool.value === 'circle'
    ? createCircle(shapeStart, point)
    : createBox(shapeStart, point, activeTool.value === 'square')
  draw.value.deleteAll()
  draw.value.add(feature)
  activeTool.value = null
  shapeStart = null
  draw.value.changeMode('simple_select')
  emitDrawn()
}

async function selectTool(tool: typeof tools[number]['id']) {
  if (!map.value || !draw.value) return
  shapeStart = null
  activeTool.value = tool
  if (tool === 'polygon') draw.value.changeMode('draw_polygon')
  else draw.value.changeMode('simple_select')
}

function clearDrawing() {
  draw.value?.deleteAll()
  activeTool.value = null
  shapeStart = null
  emit('update:modelValue', null)
  renderGeoJson(null)
}

function editDrawing() {
  if (draw.value?.getAll().features.length) draw.value.changeMode('direct_select', {
    featureId: draw.value.getAll().features[0].id
  })
}

onMounted(async () => {
  const [{ Map, NavigationControl }, { default: MapboxDraw }] = await Promise.all([
    import('maplibre-gl'),
    import('@mapbox/mapbox-gl-draw')
  ])
  map.value = new Map({
    container: mapElement.value!,
    style: 'https://tiles.openfreemap.org/styles/liberty',
    center: [5.3, 52.2],
    zoom: 7
  })
  map.value.addControl(new NavigationControl(), 'top-right')
  draw.value = new MapboxDraw({ displayControlsDefault: false })
  map.value.addControl(draw.value)
  map.value.on('load', () => renderGeoJson(props.modelValue))
  map.value.on('click', onMapClick)
  map.value.on('draw.create', emitDrawn)
  map.value.on('draw.update', emitDrawn)
})

watch(() => props.modelValue, (value) => {
  renderGeoJson(value)
}, { deep: true })

onBeforeUnmount(() => map.value?.remove())

defineExpose({ selectTool, clearDrawing, editDrawing })
</script>

<template>
  <div class="map-shell">
    <div ref="mapElement" class="map-canvas" />
    <div class="map-toolbar">
      <UTooltip v-for="tool in tools" :key="tool.id" :text="$t(tool.id)">
        <UButton
          :icon="tool.icon"
          :color="activeTool === tool.id ? 'primary' : 'neutral'"
          :variant="activeTool === tool.id ? 'solid' : 'soft'"
          size="sm"
          :aria-label="$t(tool.id)"
          @click="selectTool(tool.id)"
        />
      </UTooltip>
      <USeparator orientation="vertical" class="h-6 mx-1" />
      <UTooltip :text="$t('edit')">
        <UButton icon="i-lucide-pencil" color="neutral" variant="soft" size="sm" :aria-label="$t('edit')" @click="editDrawing" />
      </UTooltip>
      <UTooltip :text="$t('clear')">
        <UButton icon="i-lucide-trash-2" color="neutral" variant="soft" size="sm" :aria-label="$t('clear')" @click="clearDrawing" />
      </UTooltip>
    </div>
    <div v-if="activeTool && activeTool !== 'polygon'" class="map-hint">
      {{ $t('draw') }}: {{ $t(activeTool) }} — click twice on the map
    </div>
  </div>
</template>
