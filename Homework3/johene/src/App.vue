<script setup lang="ts">
import { ref, onMounted } from 'vue'
import * as d3 from 'd3'

import AthletesMap from './components/AthletesMap.vue'
import MedalBarChart from './components/MedalBarChart.vue'
import SankeyDiagram from './components/SankeyDiagram.vue'

// selected country from map
const selectedCountry = ref<string | null>(null)

// shared Top Countries list (single source of truth)
const topCountries = ref<string[]>([])

/* ---------------------------------------
   Compute Top 10 Countries (once)
   Using medals_total.csv as ground truth
--------------------------------------- */
async function computeTopCountries() {
  const raw = await d3.csv(
    '../../data/Paris_2024_Olympic_Games/medals_total.csv'
  )

  const rows = raw.map(d => ({
    country: d.country_long || d.country || 'Unknown',
    total: +d['Total']!
  }))

  topCountries.value = rows
    .sort((a, b) => b.total - a.total)
    .slice(0, 10)
    .map(d => d.country)
}

onMounted(async () => {
  await computeTopCountries()
})

/* map event handler*/
function handleCountrySelected(country: string | null) {
  selectedCountry.value = country
}
</script>

<template>
  <VContainer
    id="main-container"
    fluid
    class="pa-0 d-flex flex-column"
  >
    <!-- Title -->
    <div class="title-bar">
      <h1>Paris 2024 Olympic Dashboard</h1>
      <p>
        Athletes and medals overview. Click on a country in the map to
        explore its medal distribution in the Sankey diagram.
      </p>
    </div>

    <!-- Map -->
    <div class="map-panel">
      <AthletesMap
        :selectedCountry="selectedCountry"
        @countrySelected="handleCountrySelected"
      />
    </div>

    <!-- Bottom Row -->
    <div class="bottom-panel">
      <div class="panel half">
        <MedalBarChart
          :topCountries="topCountries"
        />
      </div>

      <div class="panel half">
        <SankeyDiagram
          :selectedCountry="selectedCountry"
          :topCountries="topCountries"
        />
      </div>
    </div>
  </VContainer>
</template>
