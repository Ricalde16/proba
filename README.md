<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Cálculo Solar</title>
  <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
  <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    body {
      font-family: Arial, sans-serif;
      background: linear-gradient(to bottom, rgba(18,18,18,1) 0%, rgba(24,46,43,0.9) 40%, rgba(8,135,120,0.8) 100%);
      color: #e0e0e0;
      margin: 0;
      padding: 0;
      text-align: center;
      min-height: 100vh;
    }
    .container {
      max-width: 500px;
      margin: 20px auto;
      padding: 20px;
      background: linear-gradient(to bottom, rgba(8,135,120,0.7) 0%, rgba(24,46,43,0.8) 40%, rgba(18,18,18,0.9) 100%);
      border-radius: 10px;
      box-shadow: 0px 4px 10px rgba(255,255,255,0.1);
      backdrop-filter: blur(10px);
      position: relative;
    }
    input, button {
      width: 90%;
      padding: 8px;
      margin: 5px 0;
      border: none;
      border-radius: 5px;
      font-size: 14px;
    }
    input {
      background: #2c2c2c;
      color: #e0e0e0;
    }
    button {
      background: #11ad93;
      color: #fff;
      cursor: pointer;
    }
    button:hover {
      background: #0d2420;
    }
    #map {
      height: 300px;
      border-radius: 10px;
      margin-top: 10px;
    }
    canvas {
      width: 100% !important;
      height: 300px;
      max-height: 300px;
      margin-top: 10px;
    }
    /* Indicador de carga */
    #loading {
      display: none;
      position: absolute;
      top: 10px;
      right: 10px;
      background: rgba(0,0,0,0.7);
      padding: 5px 10px;
      border-radius: 5px;
      font-size: 14px;
      color: #fff;
    }
    /* Estilos para estadísticas */
    #estadisticas {
      margin-top: 10px;
      font-size: 14px;
      background: rgba(44,44,44,0.7);
      padding: 10px;
      border-radius: 5px;
    }
    footer {
      background: rgba(18,18,18,1);
      color: #e0e0e0;
      text-align: center;
      padding: 1px;
      font-size: 10px;
      margin-top: 10px;
      position: relative;
      bottom: 0;
      width: 100%;
    }
  </style>
</head>
<body>

  <div class="container">
    <div id="loading">Cargando...</div>
    <h2>CÁLCULO DE ENERGÍA SOLAR</h2>

    <h2>UBICACIÓN</h2>
    <label for="latitude">Latitud:</label>
    <input type="text" id="latitude" placeholder="Ingresa la latitud">

    <label for="longitude">Longitud:</label>
    <input type="text" id="longitude" placeholder="Ingresa la longitud">

    <button id="calcular-manual">Ingresar ubicación manualmente</button>
    <button id="auto-location">Usar mi ubicación</button>

    <h2>TIPO DE PANEL</h2>
    <label for="area">Área del Panel (m²):</label>
    <input type="number" id="area" placeholder="Ejemplo: 1" value="1">

    <label for="eficiencia">Eficiencia del Panel (%):</label>
    <input type="number" id="eficiencia" placeholder="Ejemplo: 15" value="15">

    <button onclick="calcularEnergia()">Calcular Energía</button>

    <h3>Resultado:</h3>
    <p id="resultado">...</p>

    <!-- Botones para funciones adicionales -->
    <button id="mostrar-estadisticas">Mostrar Estadísticas</button>
    <button id="descargar-datos">Descargar Datos</button>
    <div id="estadisticas"></div>

    <div id="map"></div>
    <canvas id="grafica"></canvas>
  </div>

  <footer>
    <p>&copy; 2025 Calculadora Solar | Desarrollado por Sistemas operativos trifásicos.</p>
  </footer>

  <script>
    let map = L.map('map').setView([19.4326, -99.1332], 13);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '© OpenStreetMap'
    }).addTo(map);
    let marker;
    // Variable global para almacenar la radiación diaria (solo datos válidos y/o convertidos)
    let radiacionDiariaGlobal = [];
    // Variable global para almacenar la URL de consulta a la API POWER
    let apiQueryURL = "";

    // Función para mostrar el indicador de carga
    function mostrarCarga() {
      document.getElementById("loading").style.display = "block";
    }
    // Función para ocultar el indicador de carga
    function ocultarCarga() {
      document.getElementById("loading").style.display = "none";
    }

    document.getElementById("auto-location").addEventListener("click", function() {
      if (navigator.geolocation) {
        mostrarCarga();
        navigator.geolocation.getCurrentPosition(position => {
          let lat = position.coords.latitude;
          let lon = position.coords.longitude;
          document.getElementById("latitude").value = lat;
          document.getElementById("longitude").value = lon;

          map.setView([lat, lon], 13);
          if (marker) map.removeLayer(marker);
          marker = L.marker([lat, lon]).addTo(map).bindPopup("Ubicación actual").openPopup();

          obtenerRadiacion(lat, lon);
        }, () => {
          alert("No se pudo obtener la ubicación.");
          ocultarCarga();
        });
      } else {
        alert("Geolocalización no soportada en este navegador.");
      }
    });

    document.getElementById("calcular-manual").addEventListener("click", function() {
      let lat = parseFloat(document.getElementById("latitude").value);
      let lon = parseFloat(document.getElementById("longitude").value);
      if (isNaN(lat) || isNaN(lon)) {
        alert("Ingresa coordenadas válidas.");
        return;
      }
      mostrarCarga();
      map.setView([lat, lon], 13);
      if (marker) map.removeLayer(marker);
      marker = L.marker([lat, lon]).addTo(map).bindPopup("Ubicación ingresada").openPopup();

      obtenerRadiacion(lat, lon);
    });

    // ============================
    //  OBTENER RADIACIÓN (API)
    // ============================
    function obtenerRadiacion(lat, lon) {
      // Calcular fechas: últimos 30 días (excluyendo el día actual)
      const today = new Date();
      const endDate = new Date(today);
      endDate.setDate(today.getDate() - 1);
      const startDate = new Date(today);
      startDate.setDate(today.getDate() - 30);

      const formatDate = date => {
        const y = date.getFullYear();
        const m = ('0' + (date.getMonth() + 1)).slice(-2);
        const d = ('0' + date.getDate()).slice(-2);
        return `${y}${m}${d}`;
      };

      apiQueryURL = `https://power.larc.nasa.gov/api/temporal/daily/point?parameters=ALLSKY_SFC_SW_DWN&community=RE&longitude=${lon}&latitude=${lat}&start=${formatDate(startDate)}&end=${formatDate(endDate)}&format=JSON`;

      fetch(apiQueryURL)
        .then(response => response.json())
        .then(data => {
          if (data && data.properties && data.properties.parameter && data.properties.parameter.ALLSKY_SFC_SW_DWN) {
            const radObj = data.properties.parameter.ALLSKY_SFC_SW_DWN;
            const keys = Object.keys(radObj).sort();
            let radiacionDiaria = keys.map(key => radObj[key]);

            // 1) Filtrar valores inválidos (-999)
            radiacionDiaria = radiacionDiaria.filter(val => val !== -999);
            if (radiacionDiaria.length === 0) {
              throw new Error("No hay datos válidos en la respuesta de la API");
            }

            // 2) Detectar si los datos parecen W/m² (umbral > 25) y convertir a kWh/m²/día
            const maxValue = Math.max(...radiacionDiaria);
            if (maxValue > 25) {
              radiacionDiaria = radiacionDiaria.map(val => (val * 24) / 1000);
            }

            // 3) Limitar (cap) a un máximo razonable, por ejemplo 15 kWh/m²/día
            radiacionDiaria = radiacionDiaria.map(val => (val > 15 ? 15 : val));

            radiacionDiariaGlobal = radiacionDiaria;
            actualizarGrafica(radiacionDiariaGlobal);
            // Se utiliza el último valor válido para el cálculo de energía
            calcularEnergia(radiacionDiariaGlobal[radiacionDiariaGlobal.length - 1]);
          } else {
            throw new Error("Datos no válidos de la API");
          }
          ocultarCarga();
        })
        .catch(error => {
          console.error("Error al obtener datos de la API POWER:", error);
          alert("No se pudo obtener datos reales de radiación. Se usarán datos simulados.");

          // Fallback: datos simulados (entre 200 y 350 W/m², por ejemplo)
          let simulados = [];
          for (let i = 0; i < 30; i++) {
            simulados.push(Math.random() * (350 - 200) + 200);
          }

          // Asumir que son W/m² si el máximo > 25, convertirlos a kWh/m²/día
          if (Math.max(...simulados) > 25) {
            simulados = simulados.map(val => (val * 24) / 1000);
          }

          // Capping a 15 kWh/m²/día
          simulados = simulados.map(val => (val > 15 ? 15 : val));

          radiacionDiariaGlobal = simulados;
          actualizarGrafica(radiacionDiariaGlobal);
          calcularEnergia(radiacionDiariaGlobal[radiacionDiariaGlobal.length - 1]);
          ocultarCarga();
        });
    }

    // ============================
    //   CÁLCULO DE ENERGÍA
    // ============================
    // Fórmula: kWh/día = radiación (kWh/m²/día) × área (m²) × (eficiencia/100)
    function calcularEnergia(radiacion = 300) {
      let area = parseFloat(document.getElementById("area").value);
      let eficiencia = parseFloat(document.getElementById("eficiencia").value) / 10000;

      if (isNaN(area) || isNaN(eficiencia) || area <= 0 || eficiencia <= 0) {
        document.getElementById("resultado").innerHTML = "Ingresa valores válidos.";
        return;
      }

      let energiaGenerada = radiacion * area * eficiencia;
      let mensaje = energiaGenerada > 3 ? "Buena ubicación para un panel solar." : "Ubicación poco eficiente para un panel solar.";
      document.getElementById("resultado").innerHTML = `Energía generada: <b>${energiaGenerada.toFixed(2)} kWh/día</b><br>${mensaje}`;
    }

    // ============================
    //   ACTUALIZAR GRÁFICA
    // ============================
    function actualizarGrafica(datos) {
      let ctx = document.getElementById("grafica").getContext("2d");
      if (window.miGrafica) {
        window.miGrafica.destroy();
      }

      // Cálculo del promedio móvil (últimos 5 valores)
      let promedioMovil = datos.map((_, i, arr) => {
        let inicio = Math.max(0, i - 4);
        let subset = arr.slice(inicio, i + 1);
        return subset.reduce((a, b) => a + b, 0) / subset.length;
      });

      window.miGrafica = new Chart(ctx, {
        type: 'line',
        data: {
          labels: Array.from({ length: datos.length }, (_, i) => `Día ${i + 1}`),
          datasets: [
            {
              label: "Radiación Solar (kWh/m²/día)",
              data: datos,
              borderColor: "#11ad93",
              backgroundColor: "rgba(24,46,43,0.5)",
              borderWidth: 2,
              pointRadius: 4,
              pointBackgroundColor: "#0d2420",
              fill: true
            },
            {
              label: "Promedio de radiación en 30 días",
              data: promedioMovil,
              borderColor: "#0af5cf",
              borderWidth: 2,
              borderDash: [5, 5],
              pointRadius: 2,
              pointBackgroundColor: "#0d2420",
              fill: false
            }
          ]
        },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          scales: {
            y: {
              beginAtZero: false
            }
          }
        }
      });
    }

    // ============================
    //   MOSTRAR ESTADÍSTICAS
    // ============================
    function mostrarEstadisticas() {
      if (radiacionDiariaGlobal.length === 0) {
        alert("No hay datos para mostrar. Genera los datos primero.");
        return;
      }
      const min = Math.min(...radiacionDiariaGlobal).toFixed(2);
      const max = Math.max(...radiacionDiariaGlobal).toFixed(2);
      const sum = radiacionDiariaGlobal.reduce((acc, curr) => acc + curr, 0);
      const avg = (sum / radiacionDiariaGlobal.length).toFixed(2);
      const texto = `<strong>Estadísticas de Radiación Solar (kWh/m²/día):</strong><br>
                     Mínimo: ${min}<br>
                     Máximo: ${max}<br>
                     Promedio: ${avg}`;
      document.getElementById("estadisticas").innerHTML = texto;
    }

    // ============================
    //   DESCARGAR DATOS CSV
    // ============================
    function descargarDatos() {
      if (radiacionDiariaGlobal.length === 0) {
        alert("No hay datos para descargar. Genera los datos primero.");
        return;
      }
      let csvContent = "data:text/csv;charset=utf-8,";
      csvContent += "Consulta API," + apiQueryURL + "\n";
      csvContent += "Día,Radiación (kWh/m²/día)\n";
      radiacionDiariaGlobal.forEach((dato, index) => {
        csvContent += `${index + 1},${dato.toFixed(2)}\n`;
      });
      const encodedUri = encodeURI(csvContent);
      const link = document.createElement("a");
      link.setAttribute("href", encodedUri);
      link.setAttribute("download", "radiacion_solar.csv");
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);
    }

    // Asignación de eventos a los botones adicionales
    document.getElementById("mostrar-estadisticas").addEventListener("click", mostrarEstadisticas);
    document.getElementById("descargar-datos").addEventListener("click", descargarDatos);
  </script>
</body>
</html>
