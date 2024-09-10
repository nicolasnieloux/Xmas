<template>
  <div id="map" style="height: 800px;"></div>
  <div>
    <h1>Tri par voisin 2opt</h1>

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
    // Générer le polyligne du chemin
    generatePath(map, path) {
      path.forEach((segment) => {
        const [start, end] = segment;
        L.polyline([start, end], { color: 'blue' }).addTo(map);
      });
    },
    // Calculer la distance totale
    calculateTotalDistance(path) {
      return path.reduce((total, point, index) => {
        if (index === path.length - 1) return total; // Pas de retour au point de départ
        return total + this.haversineDistance(point, path[index + 1]);
      }, 0);
    },
    // Algorithme 2-opt pour optimiser le trajet
    twoOpt(path) {
      const swap = (path, i, k) => {
        return path.slice(0, i)
            .concat(path.slice(i, k + 1).reverse())
            .concat(path.slice(k + 1));
      };

      const totalDistance = (path) => {
        let dist = 0;
        for (let i = 0; i < path.length - 1; i++) {
          dist += this.haversineDistance(path[i], path[i + 1]);
        }
        return dist; // Pas de retour au point de départ
      };

      let improvement = true;
      while (improvement) {
        improvement = false;
        for (let i = 1; i < path.length - 1; i++) {
          for (let k = i + 1; k < path.length; k++) {
            const newPath = swap(path, i, k);
            if (totalDistance(newPath) < totalDistance(path)) {
              path = newPath;
              improvement = true;
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

    // Ajout du marqueur pour le point de départ
    L.marker(coordinates[0], {
      icon: L.divIcon({
        className: 'custom-div-icon',
        html: `<img class="leaflet-marker-icon" src="https://leafletjs.com/examples/custom-icons/leaf-red.png" /><div class="number-marker" style="color: white;">0</div>`,
        iconSize: [30, 42],
        iconAnchor: [15, 42]
      })
    }).addTo(map).bindPopup("<b>Point de départ</b><br />Voici votre point de départ.").openPopup();

    // Calcul des limites pour centrer la carte
    const bounds = L.latLngBounds(coordinates);
    map.fitBounds(bounds);

    // Générer un chemin initial
    let path = coordinates.slice(1); // Exclure le point de départ du chemin initial

    // Appliquer l'algorithme 2-opt pour optimiser le chemin
    path = this.twoOpt(path);

    // Ajouter le point de départ au début du chemin
    path.unshift(coordinates[0]);

    // Mettre à jour la distance totale
    this.totalDistance = this.calculateTotalDistance(path);

    // Générer le polyligne sur la carte
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

    this.path = path; // Enregistrer le chemin pour une utilisation future si nécessaire
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