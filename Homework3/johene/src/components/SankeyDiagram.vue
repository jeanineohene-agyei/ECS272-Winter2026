<script setup lang="ts">
import * as d3 from 'd3'
import { sankey, sankeyLinkHorizontal } from 'd3-sankey'
import { ref, watch, onMounted, onBeforeUnmount } from 'vue'
import { debounce } from 'lodash'
import type { ComponentSize, Margin } from '../types'

// country name normalization
const COUNTRY_ALIASES: Record<string, string> = {
  'United States': 'United States of America',
  'USA': 'United States of America',
  'Great Britain': 'United Kingdom',
  "People's Republic of China": 'China',
  'Republic of Korea': 'South Korea',
  'DPR Korea': 'North Korea',
  'Ivory Coast': "Côte d'Ivoire",
  'DR Congo': 'Democratic Republic of the Congo',
  'Congo': 'Republic of the Congo',
  'Swaziland': 'Eswatini',
  'Cape Verde': 'Cabo Verde',
  'Tanzania': 'United Republic of Tanzania',
  'The Gambia': 'Gambia',
  'Guinea Bissau': 'Guinea-Bissau',
  'Russian Federation': 'Russia'
}

function normalizeCountryName(name: string): string {
  return COUNTRY_ALIASES[name] ?? name
}

function cleanCountry(name: string): string {
  return name
    .toLowerCase()
    .replace(/[^a-z\s]/g, '')     // remove punctuation
    .replace(/\b(republic|democratic|united|federal|state|kingdom|of|the)\b/g, '')
    .replace(/\s+/g, ' ')
    .trim()
}

const props = defineProps<{
  selectedCountry: string | null
}>()

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

/* store full dataset for dynamic filtering */
let fullData: {
  country: string
  discipline: string
  medal: string
}[] = []

const margin: Margin = {
  top: 20,
  right: 20,
  bottom: 20,
  left: 20
}

// Resize
function onResize() {
  if (!container.value) return
  size.value = {
    width: container.value.clientWidth,
    height: container.value.clientHeight
  }
}
const debouncedOnResize = debounce(onResize, 100)

// name normalization
function normalizeName(name: string): string {
  const parts = name.trim().split(/\s+/)

  // Separate ALL CAPS tokens (likely last name)
  const last = parts.filter(p => p === p.toUpperCase())
  const first = parts.filter(p => p !== p.toUpperCase())

  return [...last, ...first]
    .join(' ')
    .toLowerCase()
}

// Load Data
async function loadData() {
  const medallists = await d3.csv(
    '../../data/Paris_2024_Olympic_Games/medallists.csv'
  )

  const medals = await d3.csv(
    '../../data/Paris_2024_Olympic_Games/medals.csv'
  )

  const nameToCountry = new Map<string, string>()

  medallists.forEach(d => {
    if (!d.name || !d.country) return
    const key = normalizeName(d.name)
    nameToCountry.set(key, d.country)
  })

  fullData = medals
    .map(d => {
      if (!d.name || !d.discipline || !d.medal_type) return null
      const key = normalizeName(d.name)
      const country = nameToCountry.get(key)
      if (!country) return null

      return {
        country: normalizeCountryName(country),
        discipline: d.discipline,
        medal: d.medal_type
      }
    })
    .filter(d => d !== null) as any[]

  buildSankeyData()
}

// build sankey bbased on selection
function buildSankeyData() {
  let filtered = fullData

  if (props.selectedCountry) {
  const selectedClean = cleanCountry(props.selectedCountry)

  filtered = fullData.filter(
    d => cleanCountry(d.country) === selectedClean
  )
  } else {
  // --- Step 1: Compute total medals per country ---
  const countryTotals = d3.rollups(
    fullData,
    v => v.length,
    d => d.country
  )

  // Sort descending by total medal count
  countryTotals.sort((a, b) => b[1] - a[1])

  // Take top 10 countries
  const topCountries = countryTotals
    .slice(0, 10)
    .map(d => d[0])

  // --- Step 2: Keep your existing discipline logic ---
  const topDisciplines = d3.rollups(
    fullData,
    v => v.length,
    d => d.discipline
  )
    .sort((a, b) => b[1] - a[1])
    .slice(0, 8)
    .map(d => d[0])

  // --- Step 3: Filter ---
  filtered = fullData.filter(
    d =>
      topCountries.includes(d.country) &&
      topDisciplines.includes(d.discipline)
  )
}


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

  const nodeNames = new Set<string>()
  linkMap.forEach(l => {
    nodeNames.add(l.source)
    nodeNames.add(l.target)
  })

  nodes.value = Array.from(nodeNames).map(name => ({ name }))
  links.value = linkMap
}

// render sankey
function initSankey() {
  const svg = d3.select('#sankey-svg')
  svg.selectAll('*').remove()

  const width = size.value.width
  const height = size.value.height

  // Identify node types
  const medalColors: Record<string, string> = {
    'Gold Medal': '#D4AF37',
    'Silver Medal': '#C0C0C0',
    'Bronze Medal': '#CD7F32'
  }

  // Countries = nodes that appear as sources but not medal types
  const countrySet = new Set(
    links.value.map(l => l.source)
  )

  const disciplineSet = new Set(
    links.value
      .filter(l => !medalColors[l.target])
      .map(l => l.target)
  )

  // Country color scale
  const countryColor = d3.scaleOrdinal<string>()
    .domain(Array.from(countrySet))
    .range(d3.schemeTableau10)

  // const colorScale = d3.scaleOrdinal<string>()
  //   .domain(nodes.value.map(d => d.name))
  //   .range(d3.schemeTableau10)

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
    .attr('stroke', d => {
      if (medalColors[d.target.name]) {
        return medalColors[d.target.name]
      }

      if (disciplineSet.has(d.source.name)) {
        return '#bbbbbb'
      }

      return countryColor(d.source.name)
    })
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
    .attr('fill', d => {
      if (medalColors[d.name]) {
        return medalColors[d.name]
      }

      if (disciplineSet.has(d.name)) {
        return '#cccccc'   // sports in neutral gray
      }

      return countryColor(d.name)  // countries get color
    })
    .attr('stroke', '#333')

  node.append('text')
    .attr('x', d => d.x0! < width / 2 ? d.x1! + 4 : d.x0! - 4)
    .attr('y', d => (d.y0! + d.y1!) / 2)
    .attr('dy', '0.35em')
    .attr('text-anchor', d => d.x0! < width / 2 ? 'start' : 'end')
    .style('font-size', '10px')
    .text(d => d.name)

  /* dynamic title */
  svg.append('text')
    .attr('x', width / 2)
    .attr('y', 16)
    .attr('text-anchor', 'middle')
    .style('font-size', '13px')
    .style('font-weight', 'bold')
    .text(
    props.selectedCountry
      ? `Flow of Medals from ${props.selectedCountry} to Sports and Medal Types`
      : 'Flow of Medals from Top Countries to Sports and Medal Types'
  )
}

// Watchers
watch([size, nodes, links], ([s, n, l]) => {
  if (s.width > 0 && s.height > 0 && n.length > 0 && l.length > 0) {
    initSankey()
  }
})

watch(
  () => props.selectedCountry,
  () => {
    if (fullData.length > 0) {
      buildSankeyData()
    }
  }
)

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
