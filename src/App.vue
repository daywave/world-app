<template>
  <div class="globe-container">
    <div ref="mapContainer" class="map-container" />
    
    <!-- Overlay de carga -->
    <div v-if="loading" class="loading-overlay">
      <div class="loading-spinner"></div>
      <p>Cargando datos del mundo...</p>
    </div>
    
    <!-- Panel de información de capital -->
    <div v-if="selectedCapital" class="info-panel">
      <h2>{{ selectedCapital.name }}, {{ selectedCapital.country }}</h2>
      <p><strong>Continente:</strong> {{ selectedCapital.continent }}</p>
      <p><strong>Población:</strong> {{ selectedCapital.population?.toLocaleString() }} habitantes</p>
      <p><strong>Idiomas:</strong> {{ selectedCapital.languages }}</p>
      <p><strong>Moneda:</strong> {{ selectedCapital.currency }}</p>
      <p>{{ selectedCapital.additionalInfo }}</p>
      <div class="capital-image" v-if="selectedCapital.flag">
        <img :src="selectedCapital.flag" :alt="`Bandera de ${selectedCapital.country}`">
      </div>
      <button @click="selectedCapital = null">Cerrar</button>
    </div>
    
    <!-- Controles principales -->
    <div class="controls">
      <button @click="rotateGlobe(true)">Rotar ↻</button>
      <button @click="rotateGlobe(false)">Rotar ↺</button>
      <button @click="resetView">Resetear Vista</button>
      <button @click="refreshData">Actualizar Datos</button>
    </div>
    
    <!-- Panel de filtros y búsqueda -->
    <div class="filter-panel">
      <div class="search-box">
        <input 
          v-model="searchQuery" 
          @input="handleSearch" 
          placeholder="Buscar capital o país..."
          type="search"
        />
      </div>
      
      <div class="continent-filters">
        <button 
          v-for="continent in continents" 
          :key="continent.value"
          @click="filterByContinent(continent.value)"
          :class="{ active: activeFilter === continent.value }"
        >
          {{ continent.label }}
        </button>
        <button @click="filterByContinent(null)" :class="{ active: activeFilter === null }">
          Todos
        </button>
      </div>
      
      <div class="theme-toggle">
        <button @click="toggleDarkMode">
          {{ darkMode ? '☀️ Modo Claro' : '🌙 Modo Oscuro' }}
        </button>
      </div>
    </div>
<div class="developer-panel">
  <p><strong>By:</strong> ddaywave</p>
  <p>Para cuando te aburras en el trabajo :)</p>
</div>

  </div>
</template>

<script>
import mapboxgl from 'mapbox-gl';
import 'mapbox-gl/dist/mapbox-gl.css';
import { ref, onMounted, onBeforeUnmount, computed } from 'vue';
import axios from 'axios';

export default {
  setup() {
    // Configuración básica
    const mapContainer = ref(null);
    const map = ref(null);
    const selectedCapital = ref(null);
    const rotationInterval = ref(null);
    const capitals = ref([]);
    const loading = ref(false);
    const error = ref(null);
    const searchQuery = ref('');
    const activeFilter = ref(null);
    const darkMode = ref(false);
    const mapMarkers = ref([]);

    mapboxgl.accessToken = 'pk.eyJ1IjoiZGRheXdhdmUiLCJhIjoiY21iaGxneHA2MDBqODJqcTl4dml2MjM2NiJ9.ld7tzkkKwz3mqC3Hc3qQBw';

    // Continentes disponibles
    const continents = ref([
      { value: 'Africa', label: 'África' },
      { value: 'Americas', label: 'Américas' },
      { value: 'Asia', label: 'Asia' },
      { value: 'Europe', label: 'Europa' },
      { value: 'Oceania', label: 'Oceanía' }
    ]);

    // Capitales filtradas (para búsqueda)
    const filteredCapitals = computed(() => {
      let result = capitals.value;
      
      // Aplicar filtro de continente
      if (activeFilter.value) {
        result = result.filter(c => c.continent === activeFilter.value);
      }
      
      // Aplicar búsqueda
      if (searchQuery.value) {
        const query = searchQuery.value.toLowerCase();
        result = result.filter(c => 
          c.name.toLowerCase().includes(query) || 
          c.country.toLowerCase().includes(query)
        );
      }
      
      return result;
    });

    onMounted(async () => {
      await loadCapitals();
      initMap();
    });

    onBeforeUnmount(() => {
      if (map.value) map.value.remove();
      stopRotation();
    });

    // Cargar datos de países
    const loadCapitals = async () => {
      loading.value = true;
      error.value = null;
      
      try {
        const countriesResponse = await axios.get('https://restcountries.com/v3.1/all');
        
        capitals.value = countriesResponse.data
          .filter(country => country.capital && country.capital[0] && country.latlng)
          .map(country => {
            const languages = country.languages ? 
              Object.values(country.languages).join(', ') : 'No disponible';
            
            const currency = country.currencies ?
              Object.values(country.currencies)[0].name : 'No disponible';
            
            return {
              id: country.cca3,
              name: country.capital[0],
              country: country.name.common,
              coordinates: [country.latlng[1], country.latlng[0]],
              continent: country.region,
              population: country.population,
              languages,
              currency,
              flag: country.flags?.png,
              additionalInfo: `La capital de ${country.name.common} se encuentra en ${country.subregion || country.region}.`,
              timezones: country.timezones?.join(', ')
            };
          });
      } catch (err) {
        console.error("Error al cargar datos de países:", err);
        error.value = "Error al cargar datos. Intenta nuevamente más tarde.";
      } finally {
        loading.value = false;
      }
    };

    // Inicializar mapa
    const initMap = () => {
      const style = darkMode.value ? 
        'mapbox://styles/mapbox/dark-v11' : 
        'mapbox://styles/mapbox/satellite-streets-v12';
      
      map.value = new mapboxgl.Map({
        container: mapContainer.value,
        style,
        projection: 'globe',
        zoom: 1.5,
        center: [0, 20]
      });

      map.value.on('style.load', () => {
        map.value.setFog({
          color: darkMode.value ? 'rgb(8, 18, 42)' : 'rgb(186, 210, 235)',
          'high-color': darkMode.value ? 'rgb(36, 92, 223)' : 'rgb(36, 92, 223)',
          'horizon-blend': 0.02,
          'space-color': darkMode.value ? 'rgb(0, 0, 0)' : 'rgb(11, 11, 25)',
          'star-intensity': darkMode.value ? 1.0 : 0.6
        });

        addCapitalMarkers();
      });

      map.value.on('click', 'capital-markers', (e) => {
        const feature = e.features[0];
        selectCapital(feature.properties.capitalId);
      });
    };

    // Añadir marcadores al mapa
    const addCapitalMarkers = () => {
  if (!map.value) return;

  // Limpiar capas vectoriales existentes
  if (map.value.getLayer('capital-markers')) map.value.removeLayer('capital-markers');
  if (map.value.getLayer('capital-labels')) map.value.removeLayer('capital-labels');
  if (map.value.getSource('capitals')) map.value.removeSource('capitals');

  // Eliminar marcadores HTML anteriores
  mapMarkers.value.forEach(marker => marker.remove());
  mapMarkers.value = [];

  // Agregar marcadores personalizados por cada capital
  filteredCapitals.value.forEach(capital => {
    // Crear el elemento HTML del marcador
    const el = document.createElement('div');
    el.className = 'capital-marker';
    el.style.backgroundColor = getContinentColor(capital.continent);
    el.style.width = '12px';
    el.style.height = '12px';
    el.style.borderRadius = '50%';
    el.style.border = `2px solid ${darkMode.value ? '#000' : '#fff'}`;
    el.style.cursor = 'pointer';

    // Manejar clic en el marcador
    el.addEventListener('click', () => {
      selectCapital(capital.id);
    });

    // Crear el marcador con Mapbox
    const marker = new mapboxgl.Marker(el)
      .setLngLat(capital.coordinates)
      .addTo(map.value);

    // Guardar el marcador para poder removerlo luego
    mapMarkers.value.push(marker);
  });
};

// Función auxiliar para obtener color por continente
const getContinentColor = (continent) => {
  switch (continent) {
    case 'Europe': return '#4e79a7';
    case 'Asia': return '#f28e2b';
    case 'Africa': return '#e15759';
    case 'Americas': return '#76b7b2';
    case 'Oceania': return '#59a14f';
    default: return '#9c755f';
  }
};

    // Seleccionar capital
    const selectCapital = (capitalId) => {
      const capital = capitals.value.find(c => c.id === capitalId);
      if (capital) {
        selectedCapital.value = capital;
        map.value.flyTo({
          center: capital.coordinates,
          zoom: 6,
          essential: true,
          speed: 1.5,
          curve: 1.2
        });
      }
    };

    // Filtrar por continente
    const filterByContinent = (continent) => {
      activeFilter.value = continent;
      updateMapFilters();
    };

    // Actualizar filtros del mapa
    const updateMapFilters = () => {
      if (!map.value || !map.value.getSource('capitals')) return;
      
      // Actualizar datos mostrados
      map.value.getSource('capitals').setData({
        type: 'FeatureCollection',
        features: filteredCapitals.value.map(capital => ({
          type: 'Feature',
          geometry: {
            type: 'Point',
            coordinates: capital.coordinates
          },
          properties: {
            capitalId: capital.id,
            name: capital.name,
            country: capital.country,
            continent: capital.continent
          }
        }))
      });
      
      // Volar a la primera capital si hay filtros aplicados
      if (filteredCapitals.value.length > 0 && (activeFilter.value || searchQuery.value)) {
        map.value.flyTo({
          center: filteredCapitals.value[0].coordinates,
          zoom: 2,
          speed: 1.2
        });
      }
    };

    // Buscar capitales
    const handleSearch = () => {
      updateMapFilters();
      
      // Si hay un solo resultado, seleccionarlo automáticamente
      if (filteredCapitals.value.length === 1) {
        selectCapital(filteredCapitals.value[0].id);
      }
    };

    // Alternar modo oscuro
    const toggleDarkMode = () => {
      darkMode.value = !darkMode.value;
      
      if (map.value) {
        map.value.setStyle(darkMode.value ? 
          'mapbox://styles/mapbox/dark-v11' : 
          'mapbox://styles/mapbox/satellite-streets-v12'
        );
        
        // Volver a cargar los marcadores después de cambiar el estilo
        map.value.once('style.load', () => {
          addCapitalMarkers();
        });
      }
    };

    // Rotación del globo
    const rotateGlobe = (clockwise = true) => {
      stopRotation();
      const rotationSpeed = clockwise ? 0.5 : -0.5;
      rotationInterval.value = setInterval(() => {
        const currentRotation = map.value.getBearing();
        map.value.rotateTo(currentRotation + rotationSpeed, { duration: 0 });
      }, 50);
    };

    const stopRotation = () => {
      if (rotationInterval.value) {
        clearInterval(rotationInterval.value);
        rotationInterval.value = null;
      }
    };

    const resetView = () => {
      stopRotation();
      searchQuery.value = '';
      activeFilter.value = null;
      updateMapFilters();
      map.value.flyTo({
        center: [0, 20],
        zoom: 1.5,
        bearing: 0,
        pitch: 0,
        duration: 2000
      });
    };

    const refreshData = async () => {
      await loadCapitals();
      updateMapFilters();
    };

    return {
      mapContainer,
      selectedCapital,
      loading,
      error,
      searchQuery,
      activeFilter,
      darkMode,
      continents,
      rotateGlobe,
      resetView,
      refreshData,
      filterByContinent,
      handleSearch,
      toggleDarkMode
    };
  }
};
</script>

<style>

body {
  margin: 0;
  font-family: 'Courier New', monospace;
  background-color: #1e1e1e;
}

.globe-container {
  position: relative;
  width: 100%;
  height: 100vh;
}

.map-container {
  width: 100%;
  height: 100%;
}

.loading-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  color: white;
}

.loading-spinner {
  border: 5px solid #f3f3f3;
  border-top: 5px solid #3498db;
  border-radius: 50%;
  width: 50px;
  height: 50px;
  animation: spin 1s linear infinite;
  margin-bottom: 15px;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.info-panel {
  position: absolute;
  top: 20px;
  right: 20px;
  width: 320px;
  max-height: 80vh;
  overflow-y: auto;
  background: rgba(255, 255, 255, 0.95);
  padding: 20px;
  border-radius: 10px;
  box-shadow: 0 0 20px rgba(0, 0, 0, 0.3);
  z-index: 10;
}

.info-panel h2 {
  margin-top: 0;
  color: #2c3e50;
  border-bottom: 1px solid #eee;
  padding-bottom: 10px;
}

.info-panel p {
  margin: 8px 0;
  color: #34495e;
  line-height: 1.5;
}

.info-panel strong {
  color: #2c3e50;
}

.capital-image img {
  width: 100%;
  border-radius: 5px;
  margin: 15px 0;
  box-shadow: 0 0 5px rgba(0, 0, 0, 0.2);
}

/* Controles principales */
.controls {
  position: absolute;
  bottom: 30px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 10;
  display: flex;
  gap: 10px;
  background: rgba(255, 255, 255, 0.8);
  padding: 10px;
  border-radius: 20px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
}

.controls button {
  padding: 8px 15px;
  background: #3498db;
  color: white;
  border: none;
  border-radius: 15px;
  cursor: pointer;
  font-weight: bold;
  transition: all 0.3s;
}

.controls button:hover {
  background: #2980b9;
  transform: translateY(-2px);
}

/* Panel de filtros */
.filter-panel {
  position: absolute;
  top: 20px;
  left: 20px;
  z-index: 10;
  background: rgba(68, 65, 65, 0.9);
  padding: 15px;
  border-radius: 10px;
  box-shadow: 0 0 10px rgba(216, 11, 11, 0.1);
  max-width: 300px;
}

.search-box {
  margin-bottom: 15px;
}

.search-box input {
  width: 100%;
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 20px;
  font-size: 14px;
}

.continent-filters {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-bottom: 15px;
}

.continent-filters button {
  padding: 6px 12px;
  background: blue;
  border: none;
  border-radius: 15px;
  cursor: pointer;
  font-size: 12px;
  transition: all 0.2s;
}

.continent-filters button:hover {
  background: #e0e0e0;
}

.continent-filters button.active {
  background: #3498db;
  color: white;
}

.theme-toggle button {
  width: 100%;
  padding: 8px 12px;
  background: #333;
  color: white;
  border: none;
  border-radius: 20px;
  cursor: pointer;
  transition: all 0.3s;
}

.theme-toggle button:hover {
  background: #555;
}

button {
  padding: 8px 15px;
  background: #3498db;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  margin-top: 10px;
  transition: background 0.3s;
}

button:hover {
  background: #2980b9;
}

.capital-marker {
  border-radius: 50%;
  box-shadow: 0 0 5px rgba(0,0,0,0.3);
}

/* Estilos para modo oscuro */
.dark-mode .filter-panel,
.dark-mode .info-panel {
  background: rgba(40, 40, 40, 0.9);
  color: #fff;
}

.dark-mode .info-panel h2,
.dark-mode .info-panel p,
.dark-mode .info-panel strong {
  color: #fff;
}

.dark-mode .search-box input {
  background: #333;
  color: #fff;
  border-color: #555;
}

.developer-panel {
  position: absolute;
  bottom: 10px;
  right: 10px;
  background-color: rgba(255, 255, 255, 0.85);
  color: #333;
  padding: 15px 15px;
  border-radius: 8px;
  font-size: 13px;
  box-shadow: 0 2px 6px rgba(0,0,0,0.2);
  max-width: 220px;
  z-index: 10;
  transition: background-color 0.3s ease;
}

.dark-mode .developer-panel {
  background-color: rgba(20, 20, 20, 0.85);
  color: #eee;
}

</style>