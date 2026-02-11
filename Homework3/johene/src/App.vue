<script setup lang="ts">
import { ref } from 'vue'

import AthletesMap from './components/AthletesMap.vue'
import MedalBarChart from './components/MedalBarChart.vue'
import SankeyDiagram from './components/SankeyDiagram.vue'

/* shared state for coordinated views */ 

// null = default (top 8 countries in Sankey)
// string = clicked country from map
const selectedCountry = ref<string | null>(null)

// event handler from map
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
      <p>Athletes and medals overview. Click on a country in the map to render the Sankey diagram (double-click will reset the Sankey diagram).</p>
    </div>

    <!-- Map (TOP PANEL) -->
    <div class="map-panel">
      <AthletesMap
        :selectedCountry="selectedCountry"
        @countrySelected="handleCountrySelected"
      />
    </div>

    <!-- Bottom row (TWO VIEWS) -->
    <div class="bottom-panel">
      <div class="panel half">
        <MedalBarChart />
      </div>

      <div class="panel half">
        <SankeyDiagram
          :selectedCountry="selectedCountry"
        />
      </div>
    </div>
  </VContainer>
</template>
