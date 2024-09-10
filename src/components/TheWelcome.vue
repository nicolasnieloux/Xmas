<template>
  <div id="map" style="height: 800px;"></div>
  <div>
    <div>Total Distance: {{ totalDistance }} km</div>
  </div>
</template>

<script>
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';

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
      const R = 6371; // Radius of the Earth in kilometers

      const dLat = toRad(coord2[0] - coord1[0]);
      const dLon = toRad(coord2[1] - coord1[1]);

      const lat1 = toRad(coord1[0]);
      const lat2 = toRad(coord2[0]);

      const a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
          Math.sin(dLon / 2) * Math.sin(dLon / 2) * Math.cos(lat1) * Math.cos(lat2);
      const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

      return R * c;
    },
    calculateTotalDistance(path) {
      return path.reduce((total, point, index) => {
        if (index === path.length - 1) return total;
        return total + this.haversineDistance(point, path[index + 1]);
      }, 0);
    },
    greedyAlgorithm(coordinates) {
      // Start with specific indices for triangle, for example 2 and 21 but include the departure point
      const initialIndices = [0, 2, 21]; // including indices 2 and 21
      let path = initialIndices.map(i => coordinates[i]);

      // Remove chosen initial coordinates (excluding the departure point) from the list
      const remainingCoords = coordinates.filter((_, index) => !initialIndices.includes(index));

      // Apply the greedy algorithm to insert remaining cities
      remainingCoords.forEach(coord => {
        let minIncrease = Infinity;
        let bestPosition = 1; // Start after the departure point

        for (let j = 0; j < path.length; j++) {
          const nextIndex = (j + 1) % path.length;
          const increase = this.haversineDistance(path[j], coord) +
              this.haversineDistance(path[nextIndex], coord) -
              this.haversineDistance(path[j], path[nextIndex]);

          if (increase < minIncrease) {
            minIncrease = increase;
            bestPosition = j + 1;
          }
        }

        path.splice(bestPosition, 0, coord);
      });

      return path;
    }
  },
  mounted() {
    const map = L.map('map');

    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
    }).addTo(map);

    const coordinates = [
      [45.171112, 5.695952], // Point 2
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
      [45.180943, 5.698965], // Point 21
      [45.176205, 5.692165],
      [45.171244, 5.689872]
    ];

    // const departure = [45.533329, 5.9];
    // coordinates.unshift(departure);

    L.marker(coordinates[0], {
      icon: L.divIcon({
        className: 'custom-div-icon',
        html: `<img class="leaflet-marker-icon" src="https://leafletjs.com/examples/custom-icons/leaf-red.png" /><div class="number-marker" style="color: white;">0</div>`,
        iconSize: [30, 42],
        iconAnchor: [15, 42]
      })
    }).addTo(map).bindPopup("<b>Point de départ</b><br />Voici votre point de départ.").openPopup();

    const bounds = L.latLngBounds(coordinates);
    map.fitBounds(bounds);

    const path = this.greedyAlgorithm(coordinates);

    this.totalDistance = this.calculateTotalDistance(path);

    path.forEach((point, i) => {
      L.marker(point, {
        icon: L.divIcon({
          className: 'custom-div-icon',
          html: `<img class="leaflet-marker-icon" src="${i === 0 ? 'https://leafletjs.com/examples/custom-icons/leaf-red.png' : 'https://leafletjs.com/examples/custom-icons/leaf-green.png'}" /><div class="number-marker" style="color: white;">${i}</div>`,
          iconSize: [30, 42],
          iconAnchor: [15, 42]
        })
      }).addTo(map);

      if (i < path.length - 1) {
        L.polyline([point, path[i + 1]], { color: 'blue' }).addTo(map);
      }
    });

    this.path = path;
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