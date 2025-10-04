# moje-trasy-gps
<!doctype html>
<html lang="pl">
<head>
  <meta charset="utf-8" />
  <title>Mapa GPX — wyświetl trasę</title>
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"/>
  <style>
    body { margin:0; font-family:system-ui,Segoe UI,Roboto,Arial; }
    #map { height: 80vh; width: 100%; }
    .controls { padding:10px; display:flex; gap:8px; align-items:center; flex-wrap:wrap; }
    input[type="text"] { width: 360px; padding:6px; }
    .small { font-size: 0.9rem; color:#555; }
    button { padding:6px 10px; cursor:pointer; }
    #shareLink { width: 100%; }
  </style>
</head>
<body>
  <div class="controls">
    <label class="small">GPX URL: <input id="gpxUrl" type="text" placeholder="https://.../track.gpx"></label>
    <button id="loadUrl">Wczytaj z URL</button>
    <label class="small">lub wgraj plik GPX: <input id="fileInput" type="file" accept=".gpx"></label>
    <button id="fitBounds">Dopasuj widok</button>
    <button id="generateLink">Generuj udostępnialny link (jeśli użyto URL)</button>
  </div>
  <div id="map"></div>
  <div class="controls">
    <input id="shareLink" type="text" readonly placeholder="Tu pojawi się link do udostępniania"/>
  </div>

  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
  <!-- leaflet-gpx: skrypt parsujący GPX i rysujący na Leaflet -->
  <script src="https://unpkg.com/leaflet-gpx@1.7.0/gpx.js"></script>

  <script>
    // Inicjalizacja mapy
    const map = L.map('map').setView([51.0, 19.0], 6);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      maxZoom: 19,
      attribution: '&copy; OpenStreetMap'
    }).addTo(map);

    let currentGpxLayer = null;

    function clearGpx() {
      if (currentGpxLayer) {
        map.removeLayer(currentGpxLayer);
        currentGpxLayer = null;
      }
    }

    function loadGpxFromUrl(url, options = {}) {
      clearGpx();
      // leaflet-gpx obsługuje crossOrigin, podpina progress, itp.
      const gpx = new L.GPX(url, {
        async: true,
        marker_options: {
          startIconUrl: null,
          endIconUrl: null,
          shadowUrl: null
        },
        polyline_options: {
          color: options.color || '#FF0000',
          weight: 4,
          opacity: 0.8
        }
      });

      gpx.on('loaded', function(e) {
        currentGpxLayer = gpx;
        if (!options.noFit) {
          map.fitBounds(gpx.getBounds(), { padding: [20,20] });
        }
      });

      gpx.on('error', function(err) {
        alert('Błąd wczytywania GPX: ' + (err && err.message ? err.message : 'nieznany'));
      });

      gpx.addTo(map);
    }

    // Obsługa parametru ?gpx=URL
    function getQueryParam(name) {
      const params = new URLSearchParams(location.search);
      return params.get(name);
    }

    // Przyciski
    document.getElementById('loadUrl').addEventListener('click', () => {
      const url = document.getElementById('gpxUrl').value.trim();
      if (!url) { alert('Wprowadź URL pliku GPX.'); return; }
      loadGpxFromUrl(url);
    });

    // Wczytywanie lokalnego pliku GPX (FileReader -> blob URL)
    document.getElementById('fileInput').addEventListener('change', (e) => {
      const f = e.target.files[0];
      if (!f) return;
      const reader = new FileReader();
      reader.onload = function(evt) {
        // Tworzymy blob URL, aby leaflet-gpx mógł go wczytać
        const blob = new Blob([evt.target.result], { type: 'application/gpx+xml' });
        const blobUrl = URL.createObjectURL(blob);
        loadGpxFromUrl(blobUrl);
        // Uwaga: blob URL działa tylko lokalnie w tej sesji przeglądarki.
      };
      reader.readAsText(f);
    });

    document.getElementById('fitBounds').addEventListener('click', () => {
      if (currentGpxLayer) map.fitBounds(currentGpxLayer.getBounds(), { padding: [20,20] });
      else alert('Brak załadowanej trasy.');
    });

    document.getElementById('generateLink').addEventListener('click', () => {
      const url = document.getElementById('gpxUrl').value.trim();
      if (!url) {
        alert('Aby wygenerować link musisz wprowadzić publiczny URL do pliku GPX (pole "GPX URL").');
        return;
      }
      // Bezpiecznie zakoduj w parametrze query
      const pageUrl = new URL(location.href.split('?')[0]);
      pageUrl.searchParams.set('gpx', url);
      const link = pageUrl.toString();
      const out = document.getElementById('shareLink');
      out.value = link;
      out.select();
      document.execCommand('copy');
      alert('Link skopiowany do schowka (jeśli przeglądarka na to pozwala).');
    });

    // Jeśli strona otwarta z parametrem ?gpx=...
    const gpxParam = getQueryParam('gpx');
    if (gpxParam) {
      // Wstaw wartość do pola i wczytaj
      document.getElementById('gpxUrl').value = gpxParam;
      // Pamiętaj: server z GPX musi zezwalać na CORS jeśli cross-origin
      loadGpxFromUrl(gpxParam);
    }

  </script>
</body>
</html>
