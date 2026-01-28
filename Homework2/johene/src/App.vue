<script setup lang="ts">
import AthletesMap from './components/AthletesMap.vue'
import MedalHeatmap from './components/MedalHeatmap.vue'
import SankeyDiagram from './components/SankeyDiagram.vue'
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
      <p>Athletes and medals overview</p>
    </div>

    <!-- Map (TOP PANEL) -->
    <div class="map-panel">
      <AthletesMap />
    </div>

    <!-- Bottom row (TWO VIEWS) -->
    <div class="bottom-panel">
      <div class="panel half">
        <MedalHeatmap />
      </div>
      <div class="panel half">
        <SankeyDiagram />
      </div>
    </div>
  </VContainer>
</template>