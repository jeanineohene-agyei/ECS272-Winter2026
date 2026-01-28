<script setup lang="ts">
import * as d3 from 'd3'
import { ref, watch, onMounted, onBeforeUnmount } from 'vue'
import { debounce } from 'lodash'

import type { ComponentSize, Margin } from '../types'

interface HeatmapCell {
  country: string
  discipline: string
  value: number
}

const size = ref<ComponentSize>({ width: 0, height: 0 })
const container = ref<HTMLElement | null>(null)
const heatmapData = ref<HeatmapCell[]>([])

const legendSpace = 40   // space reserved for color legend

const margin: Margin = {
  top: 36,
  right: 20,
  bottom: 60,
  left: 120
}

/* resize handling*/
function onResize() {
  if (!container.value) return
  size.value = {
    width: container.value.clientWidth,
    height: container.value.clientHeight
  }
}

const debouncedOnResize = debounce(onResize, 100)

/* load + join data
   medallists + medals */
function normalizeName(name: string, reversed: boolean): string {
  const parts = name.trim().split(/\s+/)
  if (parts.length < 2) return name.toLowerCase()

  return reversed
    ? `${parts[1]} ${parts[0]}`.toLowerCase() // "Last First" → "First Last"
    : `${parts[0]} ${parts[1]}`.toLowerCase() // already "First Last"
}

async function loadHeatmapData() {
  const medallists = await d3.csv(
    '../../data/Paris_2024_Olympic_Games/medallists.csv'
  )

  const medals = await d3.csv(
    '../../data/Paris_2024_Olympic_Games/medals.csv'
  )

  /* ---- name → country lookup ---- */
  const nameToCountry = new Map<string, string>()

    medallists.forEach(d => {
    if (!d.name || !d.country) return

    const normalized = normalizeName(d.name, true) // last first change to first last
    nameToCountry.set(normalized, d.country)
    })


  /* join medals with country */
  const joined = medals
  .map(d => {
    if (!d.name || !d.discipline) return null

    const normalized = normalizeName(d.name, false) // already First Last
    const country = nameToCountry.get(normalized)

    if (!country) return null

    return {
      country,
      discipline: d.discipline
    }
  })
  .filter(d => d !== null) as { country: string; discipline: string }[]


  /*  (country, discipline) map to count */
  const rolled = d3.rollups(
    joined,
    v => v.length,
    d => d.country,
    d => d.discipline
  )

  let cells: HeatmapCell[] = []
  rolled.forEach(([country, disciplines]) => {
    disciplines.forEach(([discipline, count]) => {
      cells.push({ country, discipline, value: count })
    })
  })

  /* reduce size for dashboard */
  const topCountries = Array.from(
    d3.rollups(cells, v => d3.sum(v, d => d.value), d => d.country)
      .sort((a, b) => b[1] - a[1])
      .slice(0, 15)
      .map(d => d[0])
  )

  const topDisciplines = Array.from(
    d3.rollups(cells, v => d3.sum(v, d => d.value), d => d.discipline)
      .sort((a, b) => b[1] - a[1])
      .slice(0, 8)
      .map(d => d[0])
  )

  heatmapData.value = cells.filter(
    d =>
      topCountries.includes(d.country) &&
      topDisciplines.includes(d.discipline)
  )
}

/* render heatmap */
function initHeatmap() {
  const svg = d3.select('#heatmap-svg')
  svg.selectAll('*').remove()

  const width = size.value.width
  const height = size.value.height

  const countries = Array.from(
    new Set(heatmapData.value.map(d => d.country))
  )

  const disciplines = Array.from(
    new Set(heatmapData.value.map(d => d.discipline))
  )

  const xScale = d3.scaleBand()
    .domain(disciplines)
    .range([margin.left, width - margin.right - legendSpace])
    .padding(0.05)

  const yScale = d3.scaleBand()
    .domain(countries)
    .range([margin.top, height - margin.bottom])
    .padding(0.05)

  const maxValue = d3.max(heatmapData.value, d => d.value) ?? 0

  const colorScale = d3.scaleSequential()
    .domain([0, maxValue])
    .interpolator(d3.interpolateBlues)

  /* axes */
  svg.append('g')
    .attr('transform', `translate(0, ${height - margin.bottom})`)
    .call(d3.axisBottom(xScale))
    .selectAll('text')
    .attr('transform', 'rotate(-30)')
    .style('text-anchor', 'end')
    .style('font-size', '10px')

  svg.append('g')
    .attr('transform', `translate(${margin.left}, 0)`)
    .call(d3.axisLeft(yScale))
    .style('font-size', '10px')

  /* cells */
  svg.append('g')
    .selectAll('rect')
    .data(heatmapData.value)
    .join('rect')
    .attr('x', d => xScale(d.discipline)!)
    .attr('y', d => yScale(d.country)!)
    .attr('width', xScale.bandwidth())
    .attr('height', yScale.bandwidth())
    .attr('fill', d => colorScale(d.value))
    .attr('stroke', '#eee')

  /* -----------------------------
   Legend
  ------------------------------ */

  const legendHeight = Math.min(120, height * 0.5)
  const legendWidth = 10

  const legendScale = d3.scaleLinear()
    .domain([0, maxValue])
    .range([legendHeight, 0])

  const legendAxis = d3.axisRight(legendScale)
    .ticks(4)
    .tickSize(4)
    .tickFormat(d3.format('d'))

  const defs = svg.append('defs')

  const gradient = defs.append('linearGradient')
    .attr('id', 'heatmap-gradient')
    .attr('x1', '0%')
    .attr('y1', '100%')
    .attr('x2', '0%')
    .attr('y2', '0%')

  d3.range(0, 1.01, 0.1).forEach(t => {
    gradient.append('stop')
      .attr('offset', `${t * 100}%`)
      .attr('stop-color', colorScale(t * maxValue))
  })

  const legend = svg.append('g')
    .attr(
      'transform',
      `translate(${width - margin.right - legendSpace + 10},
                ${(height - legendHeight) / 2})`
    )

  legend.append('rect')
    .attr('width', legendWidth)
    .attr('height', legendHeight)
    .style('fill', 'url(#heatmap-gradient)')
    .style('stroke', '#ccc')

  legend.append('g')
    .attr('transform', `translate(${legendWidth}, 0)`)
    .call(legendAxis)

  legend.append('text')
    .attr('x', -4)
    .attr('y', -8)
    .style('font-size', '10px')
    .style('font-weight', 'bold')
    .text('Medals')


  /* title */
  svg.append('text')
    .attr('x', width / 2)
    .attr('y', 18)
    .attr('text-anchor', 'middle')
    .style('font-size', '13px')
    .style('font-weight', 'bold')
    .text('Medal Count by Country and Discipline')
}


watch([size, heatmapData], ([s, d]) => {
  if (s.width > 0 && s.height > 0 && d.length > 0) {
    initHeatmap()
  }
})

onMounted(async () => {
  window.addEventListener('resize', debouncedOnResize)
  onResize()
  await loadHeatmapData()
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', debouncedOnResize)
})
</script>

<template>
  <div class="chart-container" ref="container">
    <svg id="heatmap-svg" width="100%" height="100%"></svg>
  </div>
</template>

<style scoped>
.chart-container {
  width: 100%;
  height: 100%;
}
</style>
