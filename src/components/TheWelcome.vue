<template>
  <div>
    <div id="map" style="height: 800px;"></div>
    <div>
      <h1>Tri par Algorithme Génétique</h1>
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
    const response = await fetch('/csv/small.csv');
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

    runGeneticAlgorithm(coordinates);
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

function runGeneticAlgorithm(coordinates) {
  const populationSize = 100;
  const numGenerations = 200;
  const mutationRate = 0.01;

  const numCities = coordinates.length;

  function createIndividual() {
    const individual = Array.from({ length: numCities }, (_, i) => i);
    for (let i = numCities - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [individual[i], individual[j]] = [individual[j], individual[i]];
    }
    return individual;
  }

  function createPopulation() {
    return Array.from({ length: populationSize }, createIndividual);
  }

  function calculateFitness(individual) {
    let totalDistance = 0;
    for (let i = 0; i < numCities; i++) {
      const start = coordinates[individual[i]];
      const end = coordinates[individual[(i + 1) % numCities]];
      totalDistance += haversineDistance(start, end);
    }
    return 1 / totalDistance;
  }

  function selectParent(population) {
    const fitnesses = population.map(calculateFitness);
    const sumFitness = fitnesses.reduce((sum, fitness) => sum + fitness, 0);
    const probabilities = fitnesses.map(fitness => fitness / sumFitness);

    const rand = Math.random();
    let sumProbability = 0;

    for (let i = 0; i < population.length; i++) {
      sumProbability += probabilities[i];
      if (sumProbability >= rand) {
        return population[i];
      }
    }

    return population[population.length - 1];
  }

  function crossover(parent1, parent2) {
    const start = Math.floor(Math.random() * numCities);
    const end = start + Math.floor(Math.random() * (numCities - start));

    const child = Array(numCities).fill(-1);
    for (let i = start; i < end; i++) {
      child[i] = parent1[i];
    }

    let current = 0;
    for (const gene of parent2) {
      if (!child.includes(gene)) {
        while (child[current] !== -1) {
          current++;
        }
        child[current] = gene;
      }
    }

    return child;
  }

  function mutate(individual) {
    for (let i = 0; i < numCities; i++) {
      if (Math.random() < mutationRate) {
        const j = Math.floor(Math.random() * numCities);
        [individual[i], individual[j]] = [individual[j], individual[i]];
      }
    }
  }

  function evolve(population) {
    const newPopulation = [];
    for (let i = 0; i < populationSize; i++) {
      const parent1 = selectParent(population);
      const parent2 = selectParent(population);
      const child = crossover(parent1, parent2);
      mutate(child);
      newPopulation.push(child);
    }
    return newPopulation;
  }

  let population = createPopulation();
  let bestIndividual = population[0];
  let bestFitness = calculateFitness(bestIndividual);

  for (let generation = 0; generation < numGenerations; generation++) {
    population = evolve(population);
    for (const individual of population) {
      const fitness = calculateFitness(individual);

      if (fitness > bestFitness) {
        bestFitness = fitness;
        bestIndividual = individual;
      }
    }
  }

  const bestPath = bestIndividual.map(index => coordinates[index]);
  path.value = bestPath;
  const bestPathDistance = 1 / bestFitness;
  totalDistance.value = parseFloat(bestPathDistance.toFixed(2)); // Round to 2 decimal places for better readability

  drawMap(bestPath);
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
</style>