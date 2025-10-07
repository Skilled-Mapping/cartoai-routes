# 🗺️ CartoAI Routes — Squarespace Map Integration

This repository hosts the **GeoJSON route data** displayed on the [CartoAI / Skilled Mapping](https://www.skilledmapping.com) website.  
The data is served via **jsDelivr CDN** and loaded dynamically in a Squarespace page using **Leaflet.js**.

---

## 📦 Project structure

cartoai-routes/
│
├── routes_batch_01.geojson
├── routes_batch_02.geojson
├── routes_batch_03.geojson
├── routes_batch_04.geojson
└── README.md


Each `routes_batch_X.geojson` file contains a **FeatureCollection of LineStrings** (or MultiLineStrings) representing roads already surveyed by CartoAI’s thermal capture system.

---

## 🌍 How it’s used on Squarespace

1. Squarespace does **not** allow custom file hosting for `.geojson` with CORS headers.  
   → To bypass this, the files are hosted here on GitHub and served via **jsDelivr**, which provides public, CORS-friendly URLs.

2. The Squarespace page includes a **Leaflet map** that loads all the routes from jsDelivr at runtime using JavaScript.

3. The final site code snippet embedded in Squarespace is shown below.

---

## 🧩 Code snippet used on Squarespace

Paste this entire block into a **Code Block** on your Squarespace page (not Markdown):

```html
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/leaflet.css" />
<script src="https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/leaflet.js" defer></script>

<style>
  #map { height: 70vh; width: 100%; }
  .sqs-block-code, .sqs-block-code pre { background: transparent !important; padding:0 !important; margin:0 !important; border:0 !important; }
</style>

<div id="map"></div>

<script>
(function(){
  const FILES = [
    'https://cdn.jsdelivr.net/gh/Skilled-Mapping/cartoai-routes/routes_batch_01.geojson?v=1',
    'https://cdn.jsdelivr.net/gh/Skilled-Mapping/cartoai-routes/routes_batch_02.geojson?v=1',
    'https://cdn.jsdelivr.net/gh/Skilled-Mapping/cartoai-routes/routes_batch_03.geojson?v=1',
    'https://cdn.jsdelivr.net/gh/Skilled-Mapping/cartoai-routes/routes_batch_04.geojson?v=1'
  ];

  // Detect [lat,lon] vs [lon,lat] and fix if needed
  function looksLikeLatLonUK(p){ const a=p?.[0], b=p?.[1]; return typeof a==='number'&&typeof b==='number'&&a>48&&a<60&&b>-11&&b<4; }
  const swapLine = cs => cs.map(c => Array.isArray(c[0]) ? swapLine(c) : [c[1], c[0]]);
  function fixGeo(geo){
    function maybeSwap(g){
      if (!g) return g;
      if (g.type==='LineString' && g.coordinates?.length && looksLikeLatLonUK(g.coordinates[0])) g.coordinates = swapLine(g.coordinates);
      if (g.type==='MultiLineString' && g.coordinates?.length && looksLikeLatLonUK(g.coordinates[0]?.[0])) g.coordinates = swapLine(g.coordinates);
      if (g.type==='GeometryCollection' && g.geometries) g.geometries.forEach(maybeSwap);
      return g;
    }
    if (geo.type==='FeatureCollection') geo.features.forEach(f => f.geometry = maybeSwap(f.geometry));
    else if (geo.type==='Feature') geo.geometry = maybeSwap(geo.geometry);
    else maybeSwap(geo);
    return geo;
  }

  async function loadAll(map){
    const layers=[];
    for (const url of FILES){
      try{
        const res = await fetch(url, { cache: 'no-store' });
        if(!res.ok) throw new Error('HTTP '+res.status);
        const geo = await res.json();
        const layer = L.geoJSON(fixGeo(geo), { style:{ color:'red', weight:3, opacity:0.9 } }).addTo(map);
        layers.push(layer);
      }catch(e){ console.error('Failed to load', url, e); }
    }
    if(layers.length){
      const b = L.featureGroup(layers).getBounds();
      if (b.isValid()) map.fitBounds(b.pad(0.1));
    }
  }

  function init(){
    const el = document.getElementById('map');
    if(!el || window.__leafletInit || !window.L) return;
    window.__leafletInit = true;
    const map = L.map('map', { preferCanvas:true }).setView([54,-2],6);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',{maxZoom:19,attribution:'© OpenStreetMap'}).addTo(map);
    loadAll(map);
  }

  function tryInit(){ if(window.L) init(); else setTimeout(tryInit,50); }
  document.addEventListener('DOMContentLoaded', tryInit);
  window.addEventListener('load', tryInit);
  document.addEventListener('ss:load', tryInit); // for Squarespace AJAX navigation
})();
</script>
```

# 🔁 Updating routes

1. Edit or add new .geojson files here in the repo.
2. Commit to main — jsDelivr automatically updates.
3. In the Squarespace code, bump the cache version — for example,
change ?v=1 → ?v=2 to ensure visitors get the latest data.

Optionally, pin to a specific commit for stability:
```
https://cdn.jsdelivr.net/gh/Skilled-Mapping/cartoai-routes@<commit-sha>/routes_batch_01.geojson
```

# 🧠 Notes

* This setup avoids Squarespace’s file restrictions and CORS limitations.
* jsDelivr is globally cached, free, and doesn’t require API keys.
* For very large datasets (>10–20 MB per file), consider:
  * Simplifying geometries with mapshaper.org
  * Splitting files by region or using vector tiles later.
