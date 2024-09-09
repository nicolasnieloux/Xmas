<template>
  <div>
    <div id="map" style="height: 800px;"></div>
    <div id="distance">
      <h3>Distance totale parcourue : {{ totalDistance.toFixed(2) }} km</h3>
    </div>
  </div>
</template>

<script>
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';

export default {
  name: 'MapComponent',
  data() {
    return {
      totalDistance: 0 // Initialisation de la distance totale
    };
  },
  methods: {
    // Méthode pour calculer la distance haversine entre deux coordonnées GPS
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
    // Méthode pour trouver le plus proche voisin d'un point donné non visité
    findNearestNeighbor(coordinates, point, visited) {
      const distances = coordinates
          .map((coord, index) => ({
            index: index,
            distance: this.haversineDistance(point, coord)
          }))
          .filter(neighbor => !visited.includes(neighbor.index));

      distances.sort((a, b) => a.distance - b.distance);

      return distances.length ? coordinates[distances[0].index] : null;
    },
    // Méthode pour générer le polyligne du chemin
    generatePath(map, path) {
      path.forEach((segment, index) => {
        const [start, end, distance] = segment;

        // Ajouter la polyline pour chaque segment du chemin
        const polyline = L.polyline([start, end], { color: 'blue' }).addTo(map);

        // Afficher la distance sur la polyline
        polyline.bindPopup(`Distance: ${distance.toFixed(2)} km`).openPopup();

        L.marker(end, {
          icon: L.icon({
            iconUrl: 'https://leafletjs.com/examples/custom-icons/leaf-green.png',
            shadowUrl: 'https://leafletjs.com/examples/custom-icons/leaf-shadow.png'
          })
        }).addTo(map).bindPopup(`<b>Point ${index + 1}</b>`);
      });
    },
    // Méthode pour obtenir la distance totale d'un chemin
    getTotalDistance(path) {
      return path.reduce((total, [start, end, distance]) => total + distance, 0);
    },
    // Méthode pour effectuer un swap 2-opt et retourner la nouvelle version du chemin
    twoOptSwap(route, i, k) {
      const newRoute = route.slice(0, i)
          .concat(route.slice(i, k + 1).reverse())
          .concat(route.slice(k + 1));
      return newRoute;
    },
    // Algorithme 2-opt pour optimiser le chemin
    twoOpt(path) {
      let improved = true;
      while (improved) {
        improved = false;
        for (let i = 1; i < path.length - 1; i++) {
          for (let k = i + 1; k < path.length; k++) {
            const newPath = this.twoOptSwap(path, i, k);
            if (this.getTotalDistance(newPath) < this.getTotalDistance(path)) {
              path = newPath;
              improved = true;
            }
          }
        }
      }
      return path;
    }
  },
  mounted() {
    // Initialisation de la carte
    const map = L.map('map');

    // Ajout du layer OpenStreetMap
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributeurs'
    }).addTo(map);

    // Coordonnées GPS des marqueurs
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

    // Point de départ
    const departure = [45.533329, 5.9];

    // Ajout du marqueur pour le point de départ
    L.marker(departure, {
      icon: L.icon({
        iconUrl: 'https://leafletjs.com/examples/custom-icons/leaf-red.png',
        shadowUrl: 'https://leafletjs.com/examples/custom-icons/leaf-shadow.png'
      })
    }).addTo(map).bindPopup("<b>Point de départ</b><br />Voici votre point de départ.").openPopup();

    // Calcul des limites pour centrer la carte
    const allCoordinates = [...coordinates, departure];
    const bounds = L.latLngBounds(allCoordinates);
    map.fitBounds(bounds);

    // Algorithme pour trouver successivement les plus proches voisins
    let currentPoint = departure;
    const path = [];
    const visited = [];
    let totalDistance = 0;

    while (visited.length < coordinates.length) {
      const nearestNeighbor = this.findNearestNeighbor(coordinates, currentPoint, visited);
      if (nearestNeighbor) {
        const distance = this.haversineDistance(currentPoint, nearestNeighbor);
        visited.push(coordinates.indexOf(nearestNeighbor));
        path.push([currentPoint, nearestNeighbor, distance]);
        totalDistance += distance;  // Ajout de la distance à la distance totale
        currentPoint = nearestNeighbor;
      } else {
        break;
      }
    }

    // Optimisation du chemin avec 2-opt
    let optimizedPath = path.slice();
    optimizedPath = this.twoOpt(optimizedPath);

    // Stocker la distance totale optimisée dans le data de Vue pour l'affichage
    this.totalDistance = this.getTotalDistance(optimizedPath);

    // Génération des polylines pour visualiser le chemin optimisé
    this.generatePath(map, optimizedPath);
  }
};
</script>

<style>
#map {
  width: 800px;
  height: 800px; /* Augmentez ici la hauteur de la carte */
}
</style>