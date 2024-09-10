<template>
  <div id="map" style="height: 800px;"></div>
  <div>
    <h1>Tri par la Colonie de Fourmis</h1>
    <div>Distance Totale : {{ totalDistance }} km</div>
  </div>
</template>

<script>
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';

class AntColonyOptimization {
  constructor(graph, numAnts, maxIterations, alpha, beta, evaporation) {
    this.graph = graph;
    this.numAnts = numAnts;
    this.maxIterations = maxIterations;
    this.alpha = alpha;
    this.beta = beta;
    this.evaporation = evaporation;
    this.pheromone = this.initializePheromone(graph.length);
  }

  initializePheromone(size) {
    const pheromone = [];
    for (let i = 0; i < size; i++) {
      pheromone[i] = new Array(size).fill(1.0);
    }
    return pheromone;
  }

  optimize() {
    let bestTour = null;
    let bestLength = Infinity;

    for (let iteration = 0; iteration < this.maxIterations; iteration++) {
      const ants = this.createAnts();
      for (const ant of ants) {
        this.walk(ant);
        const tourLength = this.length(ant);
        if (tourLength < bestLength) {
          bestLength = tourLength;
          bestTour = ant.slice();
        }
      }
      this.updatePheromone(ants);
    }

    return { bestTour, bestLength };
  }

  createAnts() {
    const ants = [];
    for (let i = 0; i < this.numAnts; i++) {
      ants.push(this.createAnt());
    }
    return ants;
  }

  createAnt() {
    const tour = new Array(this.graph.length).fill(null);
    tour[0] = Math.floor(Math.random() * this.graph.length);
    for (let i = 1; i < this.graph.length; i++) {
      tour[i] = this.selectNext(tour[i - 1], tour);
    }
    return tour;
  }

  selectNext(current, tour) {
    const probabilities = [];
    for (let i = 0; i < this.graph.length; i++) {
      if (tour.includes(i)) {
        probabilities.push(0);
      } else {
        const pheromone = this.pheromone[current][i] ** this.alpha;
        const heuristic = (1.0 / this.graph[current][i]) ** this.beta;
        probabilities.push(pheromone * heuristic);
      }
    }
    const sum = probabilities.reduce((a, b) => a + b, 0);
    probabilities.forEach((v, i) => probabilities[i] = v / sum);

    let pSum = 0;
    const rand = Math.random();
    for (let i = 0; i < probabilities.length; i++) {
      pSum += probabilities[i];
      if (rand < pSum) {
        return i;
      }
    }
    return probabilities.length - 1;
  }

  walk(tour) {
    for (let i = 1; i < tour.length; i++) {
      tour[i] = this.selectNext(tour[i - 1], tour);
    }
  }

  length(tour) {
    let length = 0;
    for (let i = 1; i < tour.length; i++) {
      length += this.graph[tour[i - 1]][tour[i]];
    }
    length += this.graph[tour[tour.length - 1]][tour[0]];
    return length;
  }

  updatePheromone(ants) {
    this.pheromone.forEach((row, i) => row.forEach((value, j) => { this.pheromone[i][j] *= (1.0 - this.evaporation); }));

    for (const ant of ants) {
      const deltaPheromone = 1.0 / this.length(ant);
      for (let i = 1; i < ant.length; i++) {
        this.pheromone[ant[i - 1]][ant[i]] += deltaPheromone;
        this.pheromone[ant[i]][ant[i - 1]] += deltaPheromone;
      }
      this.pheromone[ant[ant.length - 1]][ant[0]] += deltaPheromone;
      this.pheromone[ant[0]][ant[ant.length - 1]] += deltaPheromone;
    }
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
    antColonyOptimization(path) {
      const graph = [];
      for (let i = 0; i < path.length; i++) {
        graph[i] = [];
        for (let j = 0; j < path.length; j++) {
          graph[i][j] = this.haversineDistance(path[i], path[j]);
        }
      }

      const aco = new AntColonyOptimization(graph, 10, 100, 1.0, 5.0, 0.5);
      const result = aco.optimize();
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

    this.path = this.antColonyOptimization(coordinates);
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