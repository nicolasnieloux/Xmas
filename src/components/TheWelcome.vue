<template>
  <div id="map" style="height: 800px;"></div>
  <div>
    <h1>Tri par Algorithme Génétique</h1>
    <div>Distance Totale : {{ totalDistance }} km</div>
  </div>
</template>

<script>
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';

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

      const a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
          Math.sin(dLon / 2) * Math.sin(dLon / 2) * Math.cos(lat1) * Math.cos(lat2);
      const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

      return R * c;
    },
    generatePath(map, path) {
      const latlngs = path.map(coord => L.latLng(coord[0], coord[1]));
      L.polyline(latlngs, { color: 'blue' }).addTo(map);
    },
    calculateTotalDistance(path) {
      return path.reduce((total, point, index) => {
        if (index === path.length - 1) return total;
        return total + this.haversineDistance(point, path[index + 1]);
      }, 0);
    },
    geneticAlgorithmOptimization(path) {
      const graph = [];
      for (let i = 0; i < path.length; i++) {
        graph[i] = [];
        for (let j = 0; j < path.length; j++) {
          graph[i][j] = this.haversineDistance(path[i], path[j]);
        }
      }

      const ga = new GeneticAlgorithm(graph, 20, 100, 0.1);
      const result = ga.optimize();
      const optimizedPath = result.bestTour.map(i => path[i]);
      this.totalDistance = result.bestLength;
      return optimizedPath;
    }
  },
  mounted() {
    const map = L.map('map');

    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributeurs'
    }).addTo(map);

    const coordinates = [
      [45.171112, 5.695952],
      [45.183152, 5.699386],
      [45.174115, 5.711106],
      [45.176123, 5.722083],
      [45.184301, 5.719791],
      [45.184252, 5.730698],
      [45.170588, 5.716664],
      [45.193702, 5.691028],
      [45.165641, 5.739938],
      [45.178718, 5.744940],
      [45.176857, 5.762518],
      [45.188512, 5.767172],
      [45.174017, 5.706729],
      [45.174458, 5.687902],
      [45.185110, 5.733667],
      [45.185702, 5.734507],
      [45.184726, 5.734666],
      [45.184438, 5.733735],
      [45.184902, 5.735256],
      [45.174812, 5.698095],
      [45.169851, 5.695723],
      [45.180943, 5.698965],
      [45.176205, 5.692165],
      [45.171244, 5.689872]
    ];

    this.path = this.geneticAlgorithmOptimization(coordinates);
    this.totalDistance = this.calculateTotalDistance(this.path);

    this.path.forEach((coord, index) => {
      L.marker(coord, {
        icon: L.divIcon({
          className: 'custom-icon',
          html: `<div class="number-marker" style="background-color: blue; color: white;">${index}</div>`,
          iconSize: [25, 25]
        })
      }).addTo(map)
          .bindPopup(`<b>Point ${index}</b><br>Lat: ${coord[0]}, Lng: ${coord[1]}`);
    });

    this.generatePath(map, this.path);
    map.fitBounds(this.path);
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