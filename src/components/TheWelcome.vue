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
  console.log('Calculating distance between', coord1, 'and', coord2); // Debug log
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

  const distance = R * c;
  console.log('Distance:', distance); // Debug log
  return distance;
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

    console.log('Coordinates:', coordinates); // Debug log

    drawMap(coordinates);

    console.log('Coordinates:', coordinates); // Debug log

    drawMap(coordinates);
  } catch (error) {
    console.error(error);
  }
}

function drawMap(coordinates) {
  const map = L.map('map').setView(coordinates[0], 13);
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '&copy; OpenStreetMap contributors'
  }).addTo(map);

  const localPath = [];
  let localTotalDistance = 0;

  coordinates.forEach((coord, idx) => {
    const marker = L.marker(coord).addTo(map);
    localPath.push(coord);

    if (idx > 0) {
      const previousCoord = coordinates[idx - 1];
      L.polyline([previousCoord, coord], { color: 'blue' }).addTo(map);

      // Calculate distance from previous point and add to total distance
      console.log('Previous Coord:', previousCoord, 'Current Coord:', coord); // Debug log
      const distance = haversineDistance(previousCoord, coord);
      console.log('Calculated Distance:', distance); // Debug log
     console.log(localTotalDistance += distance);
    }
  });

  path.value = localPath;
  totalDistance.value = parseFloat(localTotalDistance.toFixed(2)); // Round to 2 decimal places for better readability
  console.log('Total Distance:', totalDistance.value); // Debug log
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