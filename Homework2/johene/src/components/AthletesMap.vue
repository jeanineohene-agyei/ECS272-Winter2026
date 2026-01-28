<script setup lang="ts">
import * as d3 from 'd3'
import { feature } from 'topojson-client'
import { ref, watch, onMounted, onBeforeUnmount } from 'vue'
import { debounce } from 'lodash'

import type { ComponentSize } from '../types'


const size = ref<ComponentSize>({ width: 0, height: 0 })
const mapContainer = ref<HTMLElement | null>(null)

// country map to number of athletes
const athleteCounts = ref<Map<string, number>>(new Map())

const COUNTRY_ALIASES: Record<string, string> = {
  'United States': 'United States of America',
  'USA': 'United States of America',

  'Great Britain': 'United Kingdom',

  'China': 'China',
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

function normalizeCountry(
  country?: string,
  countryLong?: string
): string | null {
  const raw = countryLong || country
  if (!raw) return null

  return COUNTRY_ALIASES[raw] ?? raw
}


/* resize handling */
function onResize() {
  if (!mapContainer.value) return
  size.value = {
    width: mapContainer.value.clientWidth,
    height: mapContainer.value.clientHeight
  }
}

const debouncedOnResize = debounce(onResize, 100)

/* load + aggregate athletes.csv */
async function loadAthletes() {
  const data = await d3.csv(
    '../../data/Paris_2024_Olympic_Games/athletes.csv'
  )

  const counts = new Map<string, number>()

  data.forEach(d => {
    const name = normalizeCountry(
      d.country as string,
      d.country_long as string
    )
    if (!name) return

    counts.set(name, (counts.get(name) ?? 0) + 1)
  })

  athleteCounts.value = counts
}

/* render map + legend */
async function initMap() {
  const svg = d3.select('#map-svg')
  svg.selectAll('*').remove()

  svg.append('text')
  .attr('x', size.value.width / 2)
  .attr('y', 18)
  .attr('text-anchor', 'middle')
  .style('font-size', '14px')
  .style('font-weight', 'bold')
  .text('Number of Athletes by Country (Paris 2024)')


  // load world map
  const world = await d3.json(
    'https://cdn.jsdelivr.net/npm/world-atlas@2/countries-110m.json'
  )

  const countries = feature(
    world as any,
    (world as any).objects.countries
  )

  /* layout margins */
  const innerMargin = {
    top: 20,
    right: 100,   // space for legend
    bottom: 20,
    left: 20
  }

  /* projection */
  const projection = d3.geoNaturalEarth1()
    .fitExtent(
      [
        [innerMargin.left, innerMargin.top],
        [
          size.value.width - innerMargin.right,
          size.value.height - innerMargin.bottom
        ]
      ],
      countries as any
    )

  const path = d3.geoPath().projection(projection)

  /* color scale */
  const maxAthletes =
    d3.max(Array.from(athleteCounts.value.values())) ?? 0

  const colorScale = d3.scaleSequential()
    .domain([0, maxAthletes])
    .interpolator(d3.interpolateBlues)

  /* draw map */
  svg.append('g')
    .selectAll('path')
    .data((countries as any).features)
    .join('path')
    .attr('d', (d: any) => path(d)!)
    .attr('fill', (d: any) => {
      const name = d.properties.name
      const count = athleteCounts.value.get(name)
      return count ? colorScale(count) : '#e0e0e0'
    })
    .attr('stroke', '#999')
    .attr('stroke-width', 0.5)

  /* legend */
  const legendHeight = Math.min(240, size.value.height * 0.5)
  const legendWidth = 14

  const legendScale = d3.scaleLinear()
    .domain([0, maxAthletes])
    .range([legendHeight, 0])

  const legendAxis = d3.axisRight(legendScale)
    .ticks(5)
    .tickSize(6)

  // gradient definition
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

  // legend group
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

watch([size, athleteCounts], ([s, counts]) => {
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
  </div>
</template>

<style scoped>
.chart-container {
  width: 100%;
  height: 100%;
}
</style>
