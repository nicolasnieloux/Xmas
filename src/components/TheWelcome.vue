<template>
  <div>
    <div id="map" style="height: 800px;"></div>
    <div>
      <h1>Tri par Algorithme Génétique</h1>
      <div>Distance Totale : {{ totalDistance }} km</div>
    </div>
  </div>
</template>

<script>
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';
import Papa from 'papaparse';

class GeneticAlgorithm {
  constructor(graph, populationSize, maxGenerations, mutationRate) {
    this.graph = graph;
    this.populationSize = populationSize;
    this.maxGenerations = maxGenerations;
    this.mutationRate = mutationRate;
    this.population = this.initPopulation();
  }

  initPopulation() {
    const population = [];
    for (let i = 0; i < this.populationSize; i++) {
      const tour = Array.from({ length: this.graph.length }, (_, index) => index);
      this.shuffle(tour);
      population.push(tour);
    }
    return population;
  }

  shuffle(arr) {
    for (let i = arr.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [arr[i], arr[j]] = [arr[j], arr[i]];
    }
  }

  optimize() {
    for (let generation = 0; generation < this.maxGenerations; generation++) {
      this.population = this.population
          .map(individual => ({ tour: individual, fitness: this.fitness(individual) }))
          .sort((a, b) => a.fitness - b.fitness)
          .map(individual => individual.tour);

      const matingPool = this.population.slice(0, this.populationSize / 2);

      // Crossover
      for (let i = 0; i < this.populationSize; i++) {
        const parent1 = matingPool[Math.floor(Math.random() * matingPool.length)];
        const parent2 = matingPool[Math.floor(Math.random() * matingPool.length)];
        this.population[i] = this.crossover(parent1, parent2);
      }

      // Mutation
      this.population.forEach(individual => {
        if (Math.random() < this.mutationRate) {
          this.mutate(individual);
        }
      });
    }

    const bestIndividual = this.population.reduce((best, individual) => {
      return this.fitness(individual) < this.fitness(best) ? individual : best;
    });

    return { bestTour: bestIndividual, bestLength: this.fitness(bestIndividual) };
  }

  fitness(tour) {
    let totalDistance = 0;
    for (let i = 0; i < tour.length - 1; i++) {
      totalDistance += this.graph[tour[i]][tour[i + 1]];
    }
    return totalDistance;
  }

  crossover(parent1, parent2) {
    const start = Math.floor(Math.random() * parent1.length);
    const end = start + Math.floor(Math.random() * (parent1.length - start));
    const child = Array(parent1.length).fill(null);

    for (let i = start; i < end; i++) {
      child[i] = parent1[i];
    }

    let current = 0;
    for (let i = 0; i < parent2.length; i++) {
      if (!child.includes(parent2[i])) {
        while (child[current] !== null) {
          current++;
        }
        child[current] = parent2[i];
      }
    }

    return child;
  }

  mutate(tour) {
    const i = Math.floor(Math.random() * tour.length);
    const j = Math.floor(Math.random() * tour.length);
    [tour[i], tour[j]] = [tour[j], tour[i]];
  }
}

export default {
  name: 'MapComponent',
  data() {
    return {
      totalDistance: 0,
      path: []
    };
  },
  methods: {
    haversineDistance(coord1, coord2) {
      const toRad = angle => angle * (Math.PI / 180);
      const R = 6371; // Rayon de la Terre en kilomètres

      const dLat = toRad(coord2[0] - coord1[0]);
      const dLon = toRad(coord2[1] - coord1[1]);

      const lat1 = toRad(coord1[0]);
      const lat2 = toRad(coord2[0]);

      const a = Math.sin(dLat / 2) * Math.sin(dLat / 2) + Math.sin(dLon / 2) * Math.sin(dLon / 2) * Math.cos(lat1) * Math.cos(lat2);
      const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

      return R * c;
    },
    async loadCsvData() {
      try {
        const response = await fetch('/csv/small.csv');
        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }
        const csvText = await response.text();
        const parsedData = Papa.parse(csvText, { header: true, dynamicTyping: true });

        const coordinates = parsedData.data.map(entry => [entry.latitude, entry.longitude]);

        this.drawMap(coordinates);
      } catch (error) {
        console.error(error);
      }
    },
    drawMap(coordinates) {
      const map = L.map('map').setView(coordinates[0], 13);
      L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '&copy; OpenStreetMap contributors'
      }).addTo(map);

      const path = [];
      coordinates.forEach((coord, idx) => {
        const marker = L.marker(coord).addTo(map);
        path.push(coord);

        if (idx > 0) {
          const previousCoord = coordinates[idx - 1];
          const line = L.polyline([previousCoord, coord], { color: 'blue' }).addTo(map);
        }
      });

      this.path = path;
    }
  },
  mounted() {
    this.loadCsvData();
  }
};
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