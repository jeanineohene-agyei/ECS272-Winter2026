<script setup lang="ts">
import * as d3 from 'd3'
import { sankey, sankeyLinkHorizontal } from 'd3-sankey'
import { ref, watch, onMounted, onBeforeUnmount } from 'vue'
import { debounce } from 'lodash'

import type { ComponentSize, Margin } from '../types'

interface SankeyNode {
  name: string
}

interface SankeyLink {
  source: string
  target: string
  value: number
}

const size = ref<ComponentSize>({ width: 0, height: 0 })
const container = ref<HTMLElement | null>(null)

const nodes = ref<SankeyNode[]>([])
const links = ref<SankeyLink[]>([])

const margin: Margin = {
  top: 20,
  right: 20,
  bottom: 20,
  left: 20
}

/* resize handling */
function onResize() {
  if (!container.value) return
  size.value = {
    width: container.value.clientWidth,
    height: container.value.clientHeight
  }
}

const debouncedOnResize = debounce(onResize, 100)

/* name normalization helper */

function normalizeName(name: string, reversed: boolean): string {
  const parts = name.trim().split(/\s+/)
  if (parts.length < 2) return name.toLowerCase()

  return reversed
    ? `${parts[1]} ${parts[0]}`.toLowerCase()
    : `${parts[0]} ${parts[1]}`.toLowerCase()
}

/* load + build Sankey data */
async function loadData() {
  const medallists = await d3.csv(
    '../../data/Paris_2024_Olympic_Games/medallists.csv'
  )

  const medals = await d3.csv(
    '../../data/Paris_2024_Olympic_Games/medals.csv'
  )

  /* name map to country lookup */
  const nameToCountry = new Map<string, string>()
  medallists.forEach(d => {
    if (!d.name || !d.country) return
    const key = normalizeName(d.name, true) // last first change to first last
    nameToCountry.set(key, d.country)
  })

  /* join medals with country */
  const joined = medals
    .map(d => {
      if (!d.name || !d.discipline || !d.medal_type) return null
      const key = normalizeName(d.name, false)
      const country = nameToCountry.get(key)
      if (!country) return null

      return {
        country,
        discipline: d.discipline,
        medal: d.medal_type
      }
    })
    .filter(d => d !== null) as {
      country: string
      discipline: string
      medal: string
    }[]

  /* reduce size for dashboard */
  const topCountries = Array.from(
    d3.rollups(joined, v => v.length, d => d.country)
      .sort((a, b) => b[1] - a[1])
      .slice(0, 8)
      .map(d => d[0])
  )

  const topDisciplines = Array.from(
    d3.rollups(joined, v => v.length, d => d.discipline)
      .sort((a, b) => b[1] - a[1])
      .slice(0, 8)
      .map(d => d[0])
  )

  const filtered = joined.filter(
    d =>
      topCountries.includes(d.country) &&
      topDisciplines.includes(d.discipline)
  )

  /* build flows */
  const countryToDiscipline = d3.rollups(
    filtered,
    v => v.length,
    d => d.country,
    d => d.discipline
  )

  const disciplineToMedal = d3.rollups(
    filtered,
    v => v.length,
    d => d.discipline,
    d => d.medal
  )

  const linkMap: SankeyLink[] = []

  countryToDiscipline.forEach(([country, disciplines]) => {
    disciplines.forEach(([discipline, count]) => {
      linkMap.push({
        source: country,
        target: discipline,
        value: count
      })
    })
  })

  disciplineToMedal.forEach(([discipline, medals]) => {
    medals.forEach(([medal, count]) => {
      linkMap.push({
        source: discipline,
        target: medal,
        value: count
      })
    })
  })

  /* build node list */
  const nodeNames = new Set<string>()
  linkMap.forEach(l => {
    nodeNames.add(l.source)
    nodeNames.add(l.target)
  })

  nodes.value = Array.from(nodeNames).map(name => ({ name }))
  links.value = linkMap
}

/* render sankey */
function initSankey() {
  const svg = d3.select('#sankey-svg')
  svg.selectAll('*').remove()

  const colorScale = d3.scaleOrdinal<string>()
    .domain(nodes.value.map(d => d.name))
    .range(d3.schemeTableau10)


  const width = size.value.width
  const height = size.value.height

  const sankeyGenerator = sankey<SankeyNode, any>()
    .nodeId(d => d.name)
    .nodeWidth(14)
    .nodePadding(10)
    .extent([
      [margin.left, margin.top],
      [width - margin.right, height - margin.bottom]
    ])

  const graph = sankeyGenerator({
    nodes: nodes.value.map(d => ({ ...d })),
    links: links.value.map(d => ({ ...d }))
  })

  /* links */
  svg.append('g')
    .selectAll('path')
    .data(graph.links)
    .join('path')
    .attr('d', sankeyLinkHorizontal())
    .attr('fill', 'none')
    .attr('stroke', d => colorScale(d.source.name))
    .attr('stroke-opacity', 0.45)
    .attr('stroke-width', d => Math.max(1, d.width))

  /* nodes */
  const node = svg.append('g')
    .selectAll('g')
    .data(graph.nodes)
    .join('g')

  node.append('rect')
    .attr('x', d => d.x0!)
    .attr('y', d => d.y0!)
    .attr('height', d => d.y1! - d.y0!)
    .attr('width', d => d.x1! - d.x0!)
    .attr('fill', d => colorScale(d.name))
    .attr('stroke', '#333')

  node.append('text')
    .attr('x', d => d.x0! < width / 2 ? d.x1! + 4 : d.x0! - 4)
    .attr('y', d => (d.y0! + d.y1!) / 2)
    .attr('dy', '0.35em')
    .attr('text-anchor', d => d.x0! < width / 2 ? 'start' : 'end')
    .style('font-size', '10px')
    .text(d => d.name)

  svg.append('text')
    .attr('x', width / 2)
    .attr('y', 16)
    .attr('text-anchor', 'middle')
    .style('font-size', '13px')
    .style('font-weight', 'bold')
    .text('Distribution of Olympic Medals by Country, Discipline, and Medal Type')
}

watch([size, nodes, links], ([s, n, l]) => {
  if (s.width > 0 && s.height > 0 && n.length > 0 && l.length > 0) {
    initSankey()
  }
})

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
    <svg id="sankey-svg" width="100%" height="100%"></svg>
  </div>
</template>

<style scoped>
.chart-container {
  width: 100%;
  height: 100%;
}
</style>
