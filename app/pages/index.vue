<script setup lang="ts">
/* eslint-disable @typescript-eslint/no-explicit-any, better-tailwindcss/no-unknown-classes, vue/max-attributes-per-line, vue/singleline-html-element-content-newline */
import type { Feature, FeatureCollection, Geometry } from 'geojson'

const { t, locale, setLocale } = useI18n()
const colorMode = useColorMode()

const sample = {
  type: 'FeatureCollection',
  features: [{
    type: 'Feature',
    properties: { name: 'Demo area' },
    geometry: {
      type: 'Polygon',
      coordinates: [[[4.85, 52.35], [5.05, 52.35], [5.05, 52.25], [4.85, 52.25], [4.85, 52.35]]]
    }
  }]
}

const inputText = ref(JSON.stringify(sample, null, 2))
const geojson = ref<FeatureCollection>(sample as FeatureCollection)
const targetCrs = ref('EPSG:3857')
const sourceCrs = ref('EPSG:4326')
const errorMessage = ref('')
const copied = ref(false)
const converted = ref<FeatureCollection | null>(null)
const isConverting = ref(false)

const crsOptions = [
  { label: 'WGS 84', value: 'EPSG:4326', description: 'Geografische lengte-/breedtegraad' },
  { label: 'Web Mercator', value: 'EPSG:3857', description: 'Webkaarten en tiles' },
  { label: 'RD New (Amersfoort)', value: 'EPSG:28992', description: 'Nederland' },
  { label: 'ETRS89 / UTM zone 32N', value: 'EPSG:25832', description: 'Europa' },
  { label: 'WGS 84 / UTM zone 32N', value: 'EPSG:32632', description: 'Europa' },
  { label: 'WGS 84 / UTM zone 31N', value: 'EPSG:32631', description: 'Europa' },
  { label: 'WGS 84 / UTM zone 33N', value: 'EPSG:32633', description: 'Europa' },
  { label: 'NAD83 / UTM zone 10N', value: 'EPSG:26910', description: 'Noord-Amerika' }
]

function findCrs(value: any): string {
  const raw = value?.crs?.properties?.name || value?.crs?.properties?.href || ''
  const match = String(raw).match(/EPSG(?::|::|\/)?(\d+)/i)
  return match ? `EPSG:${match[1]}` : 'EPSG:4326'
}

function parseInput() {
  try {
    const parsed = JSON.parse(inputText.value)
    if (!parsed || !['FeatureCollection', 'Feature', 'Point', 'LineString', 'Polygon', 'MultiPoint', 'MultiLineString', 'MultiPolygon'].includes(parsed.type)) throw new Error('invalid')
    geojson.value = parsed.type === 'FeatureCollection'
      ? parsed
      : { type: 'FeatureCollection', features: [parsed.type === 'Feature' ? parsed : { type: 'Feature', properties: {}, geometry: parsed }] }
    sourceCrs.value = findCrs(parsed)
    errorMessage.value = ''
  } catch {
    errorMessage.value = t('invalidJson')
  }
}

function geometryToFeatureCollection(value: FeatureCollection | null) {
  if (!value) return
  geojson.value = value
  inputText.value = JSON.stringify(value, null, 2)
  sourceCrs.value = 'EPSG:4326'
  converted.value = null
  errorMessage.value = ''
}

function mapCoordinates(geometry: Geometry, transform: (coordinate: number[]) => number[]): Geometry {
  const map = (value: any): any => Array.isArray(value[0]) ? value.map(map) : transform(value)
  return { ...geometry, coordinates: map((geometry as any).coordinates) } as Geometry
}

async function convert() {
  parseInput()
  if (errorMessage.value) return
  isConverting.value = true
  try {
    if (sourceCrs.value === targetCrs.value) {
      converted.value = geojson.value
      return
    }
    const { load } = await import('terrio-js/load')
    const terrio = await load({ threads: false })
    const convertCoordinates = (coordinate: number[]) => {
      const [x, y, ...rest] = coordinate
      if (x === undefined || y === undefined) throw new Error('Invalid coordinate')
      const [nextX, nextY] = terrio.transformPoints(new Float64Array([x]), new Float64Array([y]), sourceCrs.value, targetCrs.value, null, {})
      if (nextX === undefined || nextY === undefined) throw new Error('Terrio returned no coordinate')
      return [nextX, nextY, ...rest]
    }
    converted.value = {
      ...geojson.value,
      features: geojson.value.features.map((feature: Feature) => ({
        ...feature,
        geometry: feature.geometry ? mapCoordinates(feature.geometry, convertCoordinates) : null
      }))
    } as FeatureCollection
  } catch (error) {
    console.error('GeoJSON conversion failed', error)
    errorMessage.value = t('conversionError')
  } finally {
    isConverting.value = false
  }
}

function getBbox(value: FeatureCollection | null) {
  if (!value) return null
  const points: number[][] = []
  const visit = (coordinates: any) => Array.isArray(coordinates[0]) ? coordinates.forEach(visit) : points.push(coordinates)
  value.features.forEach((feature: Feature) => feature.geometry && visit((feature.geometry as any).coordinates))
  if (!points.length) return null
  const xValues = points.map(p => p[0]).filter((value): value is number => value !== undefined)
  const yValues = points.map(p => p[1]).filter((value): value is number => value !== undefined)
  if (!xValues.length || !yValues.length) return null
  return [Math.min(...xValues), Math.min(...yValues), Math.max(...xValues), Math.max(...yValues)]
}

const output = computed(() => converted.value || geojson.value)
const bbox = computed(() => getBbox(output.value))
const featureCount = computed(() => output.value?.features.length || 0)

async function copyText(value: unknown) {
  await navigator.clipboard.writeText(JSON.stringify(value, null, 2))
  copied.value = true
  setTimeout(() => copied.value = false, 1800)
}

function download(value: unknown, filename: string) {
  const blob = new Blob([JSON.stringify(value, null, 2)], { type: 'application/geo+json' })
  const url = URL.createObjectURL(blob)
  const anchor = document.createElement('a')
  anchor.href = url
  anchor.download = filename
  anchor.click()
  URL.revokeObjectURL(url)
}

function toggleLocale() {
  setLocale(locale.value === 'en' ? 'nl' : 'en')
}

function toggleColorMode() {
  colorMode.preference = colorMode.value === 'dark' ? 'light' : 'dark'
}

useSeoMeta({
  title: 'GeoConvert — GeoJSON coordinate converter',
  description: 'Convert GeoJSON between coordinate reference systems directly in your browser.'
})
</script>

<template>
  <div class="app-page">
    <header class="topbar">
      <NuxtLink to="/" class="brand">
        <span class="brand-mark"><UIcon name="i-lucide-globe-2" /></span>
        <span>{{ t('appName') }}</span>
      </NuxtLink>
      <div class="topbar-actions">
        <span class="privacy-note"><UIcon name="i-lucide-shield-check" /> {{ t('privacy') }}</span>
        <UButton icon="i-lucide-sun-moon" color="neutral" variant="ghost" aria-label="Toggle color mode" @click="toggleColorMode()" />
        <UButton color="neutral" variant="soft" size="sm" @click="toggleLocale">{{ locale.toUpperCase() }}</UButton>
      </div>
    </header>

    <main class="workspace">
      <section class="hero">
        <div>
          <p class="eyebrow">GEOJSON TOOLKIT</p>
          <h1>{{ t('tagline') }}</h1>
          <p class="hero-copy">Transform, draw and export spatial data with confidence.</p>
        </div>
        <div class="hero-badge"><UIcon name="i-lucide-zap" /> Client-side only</div>
      </section>

      <div class="tool-grid">
        <section class="panel input-panel">
          <div class="panel-heading">
            <div><span class="step">01</span><h2>{{ t('input') }}</h2></div>
            <UBadge color="neutral" variant="subtle">{{ t('features', { count: featureCount }) }}</UBadge>
          </div>
          <p class="panel-help">{{ t('pasteHint') }}</p>
          <UTextarea v-model="inputText" :rows="14" autoresize class=" w-full" spellcheck="false" @blur="parseInput" />
          <p v-if="errorMessage" class="error-text"><UIcon name="i-lucide-circle-alert" /> {{ errorMessage }}</p>
          <div class="crs-row">
            <div class="field">
              <label>{{ t('sourceCrs') }} <span>{{ t('sourceDetected') }}</span></label>
              <UInput v-model="sourceCrs" icon="i-lucide-crosshair" />
            </div>
            <UIcon name="i-lucide-arrow-right" class="arrow-icon" />
            <div class="field">
              <label>{{ t('targetCrs') }}</label>
              <USelectMenu v-model="targetCrs" :items="crsOptions" value-key="value" label-key="label" :search-input="{ placeholder: t('searchCrs') }" />
            </div>
          </div>
          <UButton block size="lg" icon="i-lucide-repeat-2" :loading="isConverting" @click="convert">{{ t('convert') }}</UButton>
        </section>

        <section class="panel map-panel">
          <div class="panel-heading">
            <div><span class="step">02</span><h2>{{ t('preview') }}</h2></div>
            <span class="map-source"><span class="live-dot" /> OpenFreeMap · Liberty</span>
          </div>
          <GeoMap :model-value="geojson" @update:model-value="geometryToFeatureCollection" />
          <p class="map-note"><UIcon name="i-lucide-info" /> {{ t('draw') }} a shape, then edit its vertices directly on the map.</p>
        </section>
      </div>

      <section class="panel output-panel">
        <div class="panel-heading">
          <div><span class="step">03</span><h2>{{ t('output') }}</h2><UBadge v-if="converted" color="success" variant="subtle">{{ t('converted') }}</UBadge></div>
          <div class="output-actions">
            <UButton icon="i-lucide-copy" color="neutral" variant="soft" size="sm" @click="copyText(output)">{{ copied ? t('copied') : t('copy') }}</UButton>
            <UButton icon="i-lucide-download" color="neutral" variant="soft" size="sm" @click="download(output, 'converted.geojson')">{{ t('download') }}</UButton>
          </div>
        </div>
        <div class="output-grid">
          <pre class="output-code">{{ JSON.stringify(output, null, 2) }}</pre>
          <div class="bbox-card">
            <div class="bbox-title"><UIcon name="i-lucide-scan" /> {{ t('bbox') }}</div>
            <code v-if="bbox">{{ JSON.stringify(bbox) }}</code>
            <p v-else>{{ t('noGeometry') }}</p>
            <div class="bbox-actions">
              <UButton size="xs" color="neutral" variant="ghost" icon="i-lucide-copy" :disabled="!bbox" @click="copyText(bbox)">{{ t('copyBbox') }}</UButton>
              <UButton size="xs" color="neutral" variant="ghost" icon="i-lucide-download" :disabled="!bbox" @click="download(bbox, 'bbox.json')">{{ t('downloadBbox') }}</UButton>
            </div>
          </div>
        </div>
      </section>
    </main>
  </div>
</template>
