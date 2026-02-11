<script setup lang="ts">
import * as d3 from 'd3'
import { feature } from 'topojson-client'
import { ref, watch, onMounted, onBeforeUnmount } from 'vue'
import { debounce } from 'lodash'
import type { ComponentSize } from '../types'

const props = defineProps<{
  selectedCountry: string | null
}>()

const emit = defineEmits<{
  (e: 'countrySelected', country: string | null): void
}>()

const size = ref<ComponentSize>({ width: 0, height: 0 })
const mapContainer = ref<HTMLElement | null>(null)
const athleteCounts = ref<Map<string, number>>(new Map())
let zoomBehavior: d3.ZoomBehavior<SVGSVGElement, unknown> | null = null

// Country normalization
const COUNTRY_ALIASES: Record<string, string> = {
  'United States': 'United States of America',
  'USA': 'United States of America',
  'Great Britain': 'United Kingdom',
  "People's Republic of China": 'China',
  'Korea': 'South Korea',
  'Republic of Korea': 'South Korea',
  'DPR Korea': 'North Korea',
  'Chinese Taipei': 'Taiwan',
  'Hong Kong, China': 'Hong Kong',
  'Russian Olympic Committee': 'Russia',
  'Russian Federation': 'Russia',
  'Islamic Republic of Iran': 'Iran'
}

function normalizeCountry(country?: string, countryLong?: string): string | null {
  const raw = countryLong || country
  if (!raw) return null
  return COUNTRY_ALIASES[raw] ?? raw
}

// resize handling
function onResize() {
  if (!mapContainer.value) return
  size.value = {
    width: mapContainer.value.clientWidth,
    height: mapContainer.value.clientHeight
  }
}
const debouncedOnResize = debounce(onResize, 100)

// load athlete data
async function loadAthletes() {
  const data = await d3.csv('../../data/Paris_2024_Olympic_Games/athletes.csv')
  const counts = new Map<string, number>()

  data.forEach(d => {
    const name = normalizeCountry(d.country as string, d.country_long as string)
    if (!name) return
    counts.set(name, (counts.get(name) ?? 0) + 1)
  })

  athleteCounts.value = counts
}

// zoom controls
function zoomIn() {
  const svg = d3.select<SVGSVGElement, unknown>('#map-svg')
  if (!zoomBehavior) return
  svg.transition().duration(250).call(zoomBehavior.scaleBy as any, 1.25)
}

function zoomOut() {
  const svg = d3.select<SVGSVGElement, unknown>('#map-svg')
  if (!zoomBehavior) return
  svg.transition().duration(250).call(zoomBehavior.scaleBy as any, 0.8)
}

function resetZoom() {
  const svg = d3.select<SVGSVGElement, unknown>('#map-svg')
  if (!zoomBehavior) return
  svg.transition().duration(350).call(zoomBehavior.transform as any, d3.zoomIdentity)
}

// render Map
async function initMap() {
  const svg = d3.select<SVGSVGElement, unknown>('#map-svg')
  const tooltip = d3.select('#map-tooltip')

  svg.selectAll('*').remove()
  svg.style('cursor', 'grab')

  svg.append('text')
    .attr('x', size.value.width / 2)
    .attr('y', 18)
    .attr('text-anchor', 'middle')
    .style('font-size', '14px')
    .style('font-weight', 'bold')
    .text('Number of Athletes by Country (Paris 2024)')

  const world = await d3.json(
    'https://cdn.jsdelivr.net/npm/world-atlas@2/countries-110m.json'
  )

  const countries = feature(world as any, (world as any).objects.countries)

  const innerMargin = { top: 20, right: 100, bottom: 20, left: 20 }

  const projection = d3.geoNaturalEarth1()
    .fitExtent(
      [
        [innerMargin.left, innerMargin.top],
        [size.value.width - innerMargin.right, size.value.height - innerMargin.bottom]
      ],
      countries as any
    )

  const path = d3.geoPath().projection(projection)

  const maxAthletes = d3.max(Array.from(athleteCounts.value.values())) ?? 0
  const colorScale = d3.scaleSequential()
    .domain([0, maxAthletes])
    .interpolator(d3.interpolateBlues)

  const mapLayer = svg.append('g').attr('class', 'map-layer')

  mapLayer
    .selectAll('path')
    .data((countries as any).features)
    .join('path')
    .attr('d', (d: any) => path(d)!)
    .attr('fill', (d: any) => {
      const name = d.properties.name
      const count = athleteCounts.value.get(name)
      return count ? colorScale(count) : '#e0e0e0'
    })
    .attr('stroke', (d: any) =>
      props.selectedCountry === d.properties.name ? '#000' : '#999'
    )
    .attr('stroke-width', (d: any) =>
      props.selectedCountry === d.properties.name ? 2 : 0.5
    )

    /* hover */
    .on('mouseenter', function (event, d: any) {
      const name = d.properties.name
      const count = athleteCounts.value.get(name) ?? 0

      tooltip
        .style('opacity', 1)
        .html(`<strong>${name}</strong><br/>Athletes: ${count}`)
    })
    .on('mousemove', function (event) {
      tooltip
        .style('left', `${event.pageX + 12}px`)
        .style('top', `${event.pageY + 12}px`)
    })
    .on('mouseleave', function () {
      tooltip.style('opacity', 0)
    })

    /* click interaction */
    .on('click', function (event, d: any) {
      const name = d.properties.name

      if (props.selectedCountry === name) {
        emit('countrySelected', null) // toggle off
      } else {
        emit('countrySelected', name)
      }
    })

  /* zoom */
  zoomBehavior = d3.zoom<SVGSVGElement, unknown>()
    .scaleExtent([1, 8])
    .translateExtent([
      [0, 0],
      [size.value.width, size.value.height]
    ])
    .on('zoom', (event) => {
      mapLayer.attr('transform', event.transform)
    })

  svg.call(zoomBehavior as any)

// Legend
const legendHeight = Math.min(240, size.value.height * 0.5)
const legendWidth = 14

const legendScale = d3.scaleLinear()
  .domain([0, maxAthletes])
  .range([legendHeight, 0])

const legendAxis = d3.axisRight(legendScale)
  .ticks(5)
  .tickSize(6)

const defs = svg.append('defs')

const gradient = defs.append('linearGradient')
  .attr('id', 'legend-gradient')
  .attr('x1', '0%')
  .attr('y1', '100%')
  .attr('x2', '0%')
  .attr('y2', '0%')

d3.range(0, 1.01, 0.1).forEach(t => {
  gradient.append('stop')
    .attr('offset', `${t * 100}%`)
    .attr('stop-color', colorScale(t * maxAthletes))
})

const legend = svg.append('g')
  .attr(
    'transform',
    `translate(${size.value.width - 60}, ${(size.value.height - legendHeight) / 2})`
  )

legend.append('rect')
  .attr('width', legendWidth)
  .attr('height', legendHeight)
  .style('fill', 'url(#legend-gradient)')
  .style('stroke', '#ccc')

legend.append('g')
  .attr('transform', `translate(${legendWidth}, 0)`)
  .call(legendAxis)

legend.append('text')
  .attr('x', -10)
  .attr('y', -12)
  .style('font-size', '12px')
  .style('font-weight', 'bold')
  .text('Athletes')

}

// Watchers
watch([size, athleteCounts, () => props.selectedCountry], ([s, counts]) => {
  if (s.width > 0 && s.height > 0 && counts.size > 0) {
    initMap()
  }
})

onMounted(async () => {
  window.addEventListener('resize', debouncedOnResize)
  onResize()
  await loadAthletes()
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', debouncedOnResize)
})
</script>

<template>
  <div class="chart-container" ref="mapContainer">
    <svg id="map-svg" width="100%" height="100%"></svg>

    <div class="zoom-controls">
      <button class="zoom-btn" @click="zoomIn">+</button>
      <button class="zoom-btn" @click="zoomOut">−</button>
      <button class="zoom-btn zoom-reset" @click="resetZoom">⟳</button>
    </div>

    <div id="map-tooltip" class="tooltip"></div>
  </div>
</template>

<style scoped>
.chart-container {
  width: 100%;
  height: 100%;
  position: relative;
}

.tooltip {
  position: absolute;
  pointer-events: none;
  background: rgba(255,255,255,0.95);
  border: 1px solid #ccc;
  border-radius: 4px;
  padding: 6px 8px;
  font-size: 12px;
  box-shadow: 0 2px 6px rgba(0,0,0,0.15);
  opacity: 0;
}

.zoom-controls {
  position: absolute;
  left: 10px;
  top: 40px;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.zoom-btn {
  width: 34px;
  height: 34px;
  border: 1px solid #ccc;
  border-radius: 6px;
  background: white;
  cursor: pointer;
  font-size: 18px;
}
</style>
