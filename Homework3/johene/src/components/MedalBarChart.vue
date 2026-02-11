<script setup lang="ts">
import * as d3 from 'd3'
import { ref, watch, onMounted, onBeforeUnmount } from 'vue'
import { debounce } from 'lodash'

import type { ComponentSize, Margin } from '../types'

interface MedalRow {
  country: string
  gold: number
  silver: number
  bronze: number
  total: number
}

const size = ref<ComponentSize>({ width: 0, height: 0 })
const container = ref<HTMLElement | null>(null)
const headerRef = ref<HTMLElement | null>(null)
const data = ref<MedalRow[]>([])

const medalType = ref<'total' | 'gold' | 'silver' | 'bronze'>('total')

const margin: Margin = {
  top: 10,
  right: 20,
  bottom: 45,
  left: 150
}

// resize handling
function onResize() {
  if (!container.value) return
  size.value = {
    width: container.value.clientWidth,
    height: container.value.clientHeight
  }
}

const debouncedOnResize = debounce(onResize, 100)

// load data
async function loadData() {
  const raw = await d3.csv(
    '../../data/Paris_2024_Olympic_Games/medals_total.csv'
  )

  data.value = raw.map(d => ({
    country: d.country_long || d.country || 'Unknown',
    gold: +d['Gold Medal']!,
    silver: +d['Silver Medal']!,
    bronze: +d['Bronze Medal']!,
    total: +d['Total']!
  }))
}

// render / update chart
function initChart() {
  const svg = d3.select('#bar-svg')

  const headerHeight = headerRef.value?.clientHeight ?? 0
  const width = size.value.width
  const height = size.value.height - headerHeight

  svg
    .attr('width', width)
    .attr('height', height)

  /* persistent layers */
  const barsLayer = svg.select<SVGGElement>('g.bars').empty()
    ? svg.append('g').attr('class', 'bars')
    : svg.select<SVGGElement>('g.bars')

  const labelsLayer = svg.select<SVGGElement>('g.labels').empty()
    ? svg.append('g').attr('class', 'labels')
    : svg.select<SVGGElement>('g.labels')

  const yAxisLayer = svg.select<SVGGElement>('g.y-axis').empty()
    ? svg.append('g').attr('class', 'y-axis')
    : svg.select<SVGGElement>('g.y-axis')

  const xAxisLayer = svg.select<SVGGElement>('g.x-axis').empty()
    ? svg.append('g').attr('class', 'x-axis')
    : svg.select<SVGGElement>('g.x-axis')

  /* sort */
  const sorted = [...data.value]
    .sort((a, b) => b[medalType.value] - a[medalType.value])
    .slice(0, 15)

  const maxValue = d3.max(sorted, d => d[medalType.value]) ?? 0

  /* scales */
  const xScale = d3.scaleLinear()
    .domain([0, maxValue])
    .nice()
    .range([margin.left, width - margin.right])

  const yScale = d3.scaleBand()
    .domain(sorted.map(d => d.country).reverse())
    .range([margin.top, height - margin.bottom])
    .padding(0.12)

  /* x-axis */
  const FONT_FAMILY = 'Inter, Avenir, Helvetica, Arial, sans-serif'
  const TICK_SIZE = 11

  // x-axis
  xAxisLayer
    .attr('transform', `translate(0, ${height - margin.bottom})`)
    .transition()
    .duration(600)
    .ease(d3.easeCubicInOut)
    .call(d3.axisBottom(xScale).ticks(5))
    .selection()
    .call(g =>
      g.selectAll('text')
        .style('font-family', FONT_FAMILY)
        .style('font-size', `${TICK_SIZE}px`)
    )

  // y-axis
  yAxisLayer
    .attr('transform', `translate(${margin.left}, 0)`)
    .transition()
    .duration(700)
    .ease(d3.easeCubicInOut)
    .call(d3.axisLeft(yScale))
    .selection()
    .call(g =>
      g.selectAll('text')
        .style('font-family', FONT_FAMILY)
        .style('font-size', `${TICK_SIZE}px`)
    )

  // bars (ordering transition)
  const bars = barsLayer
    .selectAll<SVGRectElement, MedalRow>('rect')
    .data(sorted, d => d.country)

  bars.join(
    enter => enter
      .append('rect')
      .attr('x', margin.left)
      .attr('y', d => yScale(d.country)!)
      .attr('height', yScale.bandwidth())
      .attr('width', 0)
      .attr('fill', '#4a90e2')
      .call(enter => enter.transition()
        .duration(700)
        .ease(d3.easeCubicInOut)
        .attr('width', d => xScale(d[medalType.value]) - margin.left)
      ),

    update => update
      .call(update => update.transition()
        .duration(700)
        .ease(d3.easeCubicInOut)
        .attr('y', d => yScale(d.country)!)
        .attr('height', yScale.bandwidth())
        .attr('width', d => xScale(d[medalType.value]) - margin.left)
      ),

    exit => exit
      .transition()
      .duration(400)
      .attr('width', 0)
      .remove()
  )

 // value labels (follow bars)
  const labels = labelsLayer
    .selectAll<SVGTextElement, MedalRow>('text')
    .data(sorted, d => d.country)

  labels.join(
    enter => enter
      .append('text')
      .attr('x', margin.left)
      .attr('y', d => yScale(d.country)! + yScale.bandwidth() / 2 + 4)
      .style('font-size', '11px')
      .style('fill', '#222')
      .text(d => d[medalType.value].toString())
      .call(enter => enter.transition()
        .duration(700)
        .ease(d3.easeCubicInOut)
        .attr('x', d => xScale(d[medalType.value]) + 6)
      ),

    update => update
      .text(d => d[medalType.value].toString())
      .call(update => update.transition()
        .duration(700)
        .ease(d3.easeCubicInOut)
        .attr('x', d => xScale(d[medalType.value]) + 6)
        .attr('y', d => yScale(d.country)! + yScale.bandwidth() / 2 + 4)
      ),

    exit => exit.remove()
  )
}

// watchers
watch([size, data, medalType], ([s, d]) => {
  if (s.width > 0 && s.height > 0 && d.length > 0) {
    initChart()
  }
})

// lifecycle
onMounted(async () => {
  window.addEventListener('resize', debouncedOnResize)
  onResize()
  await loadData()
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', debouncedOnResize)
})
</script>

<template>
  <div class="chart-container" ref="container">
    <!-- Header -->
    <div class="header" ref="headerRef">
      <div class="title">
        Medals by Country — {{ medalType.toUpperCase() }}
      </div>

      <div class="filter">
        <label>Medal type:</label>

        <div class="select-wrapper">
          <select v-model="medalType">
            <option value="total">Total</option>
            <option value="gold">Gold</option>
            <option value="silver">Silver</option>
            <option value="bronze">Bronze</option>
          </select>
          <span class="arrow">▾</span>
        </div>
      </div>
    </div>

    <svg id="bar-svg"></svg>
  </div>
</template>

<style scoped>
.chart-container {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 6px 12px;
  font-size: 13px;
  flex-shrink: 0;
}

.title {
  font-weight: bold;
}

.filter {
  display: flex;
  align-items: center;
  gap: 6px;
}

.select-wrapper {
  position: relative;
  display: inline-block;
}

.select-wrapper select {
  padding: 4px 28px 4px 8px;
  border: 1px solid #ccc;
  border-radius: 6px;
  background: white;
  font-size: 13px;
  cursor: pointer;

  appearance: none;
  -webkit-appearance: none;
  -moz-appearance: none;
}

.select-wrapper .arrow {
  position: absolute;
  right: 8px;
  top: 50%;
  transform: translateY(-50%);
  pointer-events: none;
  font-size: 11px;
  color: #555;
}

svg {
  width: 100%;
  flex-grow: 1;
}
</style>
