<template>
  <div>
    <div id="map" style="height: 800px;"></div>
    <div>
      <h1>Tri par Algorithme de Colonie de Fourmi</h1>
      <div>Distance Totale : {{ totalDistance }} km</div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';
import Papa from 'papaparse';

const totalDistance = ref(0);
const path = ref([]);

function haversineDistance(coord1, coord2) {
  const toRad = angle => angle * (Math.PI / 180);
  const R = 6371; // Rayon de la Terre en kilomètres

  const dLat = toRad(coord2[0] - coord1[0]);
  const dLon = toRad(coord2[1] - coord1[1]);

  const lat1 = toRad(coord1[0]);
  const lat2 = toRad(coord2[0]);

  const a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
      Math.sin(dLon / 2) * Math.sin(dLon / 2) *
      Math.cos(lat1) * Math.cos(lat2);
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

  return R * c;
}

async function loadCsvData() {
  try {
    const response = await fetch('/csv/isere.csv');
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    const csvText = await response.text();
    const parsedData = Papa.parse(csvText, { header: true, dynamicTyping: true });

    const coordinates = parsedData.data.map(entry => {
      const lat = parseFloat(entry.latitude);
      const lon = parseFloat(entry.longitude);
      if (!isNaN(lat) && !isNaN(lon)) {
        return [lat, lon];
      } else {
        console.warn('Invalid coordinates:', entry); // Logs invalid entries for further inspection
        return null;
      }
    }).filter(coord => coord !== null); // Filter out any null entries

    runAntColonyOptimization(coordinates);
  } catch (error) {
    console.error(error);
  }
}

function drawMap(coordinates) {
  const map = L.map('map').setView(coordinates[0], 10);
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '&copy; OpenStreetMap contributors'
  }).addTo(map);

  coordinates.forEach((coord, idx) => {
    L.marker(coord).addTo(map);
  });

  for (let i = 0; i < coordinates.length - 1; i++) {
    L.polyline([coordinates[i], coordinates[i + 1]], { color: 'blue' }).addTo(map);
  }

  if (coordinates.length > 1) {
    L.polyline([coordinates[coordinates.length - 1], coordinates[0]], { color: 'blue' }).addTo(map);
  }
}

function runAntColonyOptimization(coordinates) {
  const numAnts = 10;
  const numIterations = 100;
  const evaporationRate = 0.5;
  const alpha = 1;
  const beta = 5;
  const pheromoneInfluence = 1;
  const initialPheromone = 1;

  const numCities = coordinates.length;
  const pheromones = Array.from({ length: numCities }, () => Array(numCities).fill(initialPheromone));
  const distances = Array.from({ length: numCities }, (_, i) =>
      Array.from({ length: numCities }, (_, j) => i === j ? 0 : haversineDistance(coordinates[i], coordinates[j]))
  );

  let bestTour = null;
  let bestTourLength = Infinity;

  for (let iter = 0; iter < numIterations; iter++) {
    const allTours = [];
    for (let ant = 0; ant < numAnts; ant++) {
      const tour = [];
      const visited = new Set();
      let currentCity = Math.floor(Math.random() * numCities);
      tour.push(currentCity);
      visited.add(currentCity);

      while (tour.length < numCities) {
        const probabilities = [];
        let totalProbability = 0;

        for (let nextCity = 0; nextCity < numCities; nextCity++) {
          if (!visited.has(nextCity)) {
            const pheromone = Math.pow(pheromones[currentCity][nextCity], alpha);
            const distanceInverse = Math.pow(1 / distances[currentCity][nextCity], beta);
            const probability = pheromone * distanceInverse;
            probabilities.push({ city: nextCity, probability });
            totalProbability += probability;
          }
        }

        const rand = Math.random() * totalProbability;
        let sumProbability = 0;
        let selectedCity = null;

        for (const { city, probability } of probabilities) {
          sumProbability += probability;
          if (sumProbability >= rand) {
            selectedCity = city;
            break;
          }
        }

        tour.push(selectedCity);
        visited.add(selectedCity);
        currentCity = selectedCity;
      }

      allTours.push(tour);
    }

    pheromones.forEach(row => row.fill(row[0] * (1 - evaporationRate)));

    allTours.forEach(tour => {
      const tourLength = tour.reduce((length, _, idx) =>
          length + distances[tour[idx]][tour[(idx + 1) % numCities]], 0);

      if (tourLength < bestTourLength) {
        bestTour = tour;
        bestTourLength = tourLength;
      }

      for (let i = 0; i < numCities; i++) {
        const start = tour[i];
        const end = tour[(i + 1) % numCities];
        pheromones[start][end] += pheromoneInfluence / tourLength;
        pheromones[end][start] += pheromoneInfluence / tourLength;
      }
    });
  }

  totalDistance.value = parseFloat(bestTourLength.toFixed(2));
  path.value = bestTour.map(cityIndex => coordinates[cityIndex]);
  drawMap(path.value);
}

onMounted(() => {
  loadCsvData();
});
</script>

<style>
#map {
  height: 800px;
  width: 800px;
}
.number-marker {
  position: absolute;
  top: 2px;
  left: 2px;
  font-size: 14px;
  font-weight: bold;
  background: black;
  border-radius: 50%;
  width: 20px;
  height: 20px;
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>