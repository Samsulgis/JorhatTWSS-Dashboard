<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8"/>
<title>Jorhat TWSS Dashboard</title>
<meta name="viewport" content="width=device-width,initial-scale=1">

<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css">

<style>
html,body,#map { height:100%; margin:0; padding:0; font-family:system-ui;}
.app { display:grid; grid-template-columns:300px 1fr; height:100vh;}
header { grid-column:1/-1; padding:10px 16px; background:#0f172a; color:#fff; display:flex; align-items:center; justify-content:space-between;}
#sidebar { background:#fff; padding:12px; overflow:auto; border-right:1px solid #ddd;}
.layer-row { display:flex; align-items:center; gap:8px; margin:6px 0;}
small { font-size:12px; color:#666;}
#map { position:relative; }
.feature-label { background:rgba(255,255,255,.9); padding:2px 6px; border-radius:4px; font-size:12px; font-weight:600;}
.legend { background:#fff; padding:8px; border-radius:6px; box-shadow:0 6px 18px rgba(8,16,40,.06); font-size:13px;}
/* Mobile layout behaviour */
@media (max-width: 768px) {

  .app {
      grid-template-columns: 1fr !important; /* map full width */
  }

  #sidebar {
      position: absolute;
      width: 260px;
      height: 100%;
      top: 50px;
      left: -260px;          /* hidden position */
      z-index: 999;
      transition: left 0.3s ease;
      box-shadow: 0 0 10px rgba(0,0,0,0.3);
  }

  #sidebar.open {
      left: 0;               /* slide in */
  }

  /* Hamburger button appears only on mobile */
  #toggleSidebar {
      display: block !important;
  }
}
</style>
</head>

<body>
<header>
    <button id="toggleSidebar" 
    style="padding:6px 10px;border:none;border-radius:4px;background:#334155;color:white;display:none;">
    ☰
</button>
  <h2 style="margin:0;">Jorhat TWSS Dashboard</h2>
  <button id="zoomAll" style="padding:6px 10px;border:none;border-radius:4px;background:#4aa3ff;color:white;">Zoom to Data</button>
</header>

<div class="app">

<aside id="sidebar">
  <h3>Layers</h3>

  <div class="layer-row"><input type="checkbox" id="chkESR" checked><label for="chkESR">ESR</label></div>
  <div class="layer-row"><input type="checkbox" id="chkCompleted" checked><label for="chkCompleted">Completed Pipeline</label></div>
  <div class="layer-row"><input type="checkbox" id="chkProposed" checked><label for="chkProposed">Proposed Pipeline</label></div>
  <div class="layer-row"><input type="checkbox" id="chkRoads" checked><label for="chkRoads">Roads / Rail / Waterway</label></div>
  <div class="layer-row"><input type="checkbox" id="chkZoning" checked><label for="chkZoning">Zoning</label></div>
  <div class="layer-row"><input type="checkbox" id="chkRWM" checked><label for="chkRWM">RWM / CWM</label></div>
  <div class="layer-row"><input type="checkbox" id="chkObstacles" checked><label for="chkObstacles">Obstacles</label></div>

  <hr>

  <h3>Basemap</h3>
  <div class="layer-row"><input type="radio" name="bm" id="bmOSM" checked><label for="bmOSM">OSM</label></div>
  <div class="layer-row"><input type="radio" name="bm" id="bmSAT"><label for="bmSAT">Satellite</label></div>

  <hr>
  <div class="legend">
    <strong>Obstacle symbology</strong><br>
    Colors generated from the obstacle `Layer` field. Click obstacle points to see all attributes.
  </div>
</aside>

<div id="map"></div>

</div>

<!-- Leaflet -->
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

<!-- proj4 for UTM → WGS84 reprojection -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/proj4js/2.8.0/proj4.js"></script>

<script>
// DEFINE UTM 46N (EPSG:32646)
proj4.defs("EPSG:32646","+proj=utm +zone=46 +datum=WGS84 +units=m +no_defs");

// Helper: is this coordinate pair likely already lon/lat (EPSG:4326)?
function isLikely4326xy(coord){
  // coord = [x,y] where x = lon, y = lat
  if(!Array.isArray(coord) || coord.length < 2) return false;
  const x = Number(coord[0]), y = Number(coord[1]);
  if(Number.isFinite(x) && Number.isFinite(y)){
    if(Math.abs(x) <= 180 && Math.abs(y) <= 90) return true;
  }
  return false;
}

// REPROJECT GeoJSON coordinates (UTM -> WGS84)
// Important: GeoJSON coordinates arrays must be [lon, lat, (z?)]
// proj4 returns [lon, lat] so we keep that order. We also preserve Z if present.
function reprojectGeoJSON(geojson){
    function fixCoords(c){
        if(!Array.isArray(c)) return c;
        // if coordinate is a number array (point) - detect base case: first element is number
        if (typeof c[0] === 'number'){
            // handle [x,y] or [x,y,z]
            // If it already looks like lon/lat, return as-is.
            if (isLikely4326xy(c)) {
                // ensure format [lon, lat, z?] (proj4 would have returned lon,lat)
                return c.slice(0, c.length); // return copy
            }
            const x = c[0], y = c[1];
            const ll = proj4("EPSG:32646","EPSG:4326",[x,y]); // returns [lon, lat]
            if (c.length > 2){
                return [ll[0], ll[1], c[2]]; // lon, lat, z
            }
            return [ll[0], ll[1]];
        }
        // nested arrays (LineString coords array or Polygon ring)
        return c.map(fixCoords);
    }

    function traverse(obj){
        if (!obj) return obj;
        if (obj.type === "FeatureCollection"){
            obj.features = obj.features.map(traverse);
            return obj;
        } else if (obj.type === "Feature"){
            if (obj.geometry && obj.geometry.coordinates){
                obj = JSON.parse(JSON.stringify(obj)); // copy
                obj.geometry.coordinates = fixCoords(obj.geometry.coordinates);
            }
            return obj;
        } else {
            return obj;
        }
    }
    return traverse(JSON.parse(JSON.stringify(geojson)));
}

// MAP
const map = L.map("map",{zoomControl:true}).setView([26.75,94.2],13);

// BASEMAPS
const osm = L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png").addTo(map);
const sat = L.tileLayer("https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}");

// LAYER GROUPS
const lgESR       = L.layerGroup().addTo(map);
const lgCompleted = L.layerGroup().addTo(map);
const lgProposed  = L.layerGroup().addTo(map);
const lgRoads     = L.layerGroup().addTo(map);
const lgZoning    = L.layerGroup().addTo(map);
const lgRWM       = L.layerGroup().addTo(map);
const lgObstacles = L.layerGroup().addTo(map);

const bounds = L.latLngBounds();

// Simple ESR icon (inline SVG)
const tankIcon = L.icon({
    iconUrl: "data:image/svg+xml;charset=utf-8," +
    encodeURIComponent(`<svg xmlns='http://www.w3.org/2000/svg' width='30' height='30'><rect x='6' y='9' width='18' height='12' rx='3' fill='#2b6cff'/><rect x='9' y='4' width='12' height='3' fill='#2b6cff'/></svg>`),
    iconSize:[24,24], iconAnchor:[12,24]
});

// Convert all properties to a popup HTML table
function buildPopup(props) {
    let html = "<div style='max-height:320px;overflow:auto;'><table style='border-collapse:collapse;width:100%;font-size:13px;'>";
    for (const key in props) {
        let val = props[key];
        if (val === null || val === undefined || val === "") val = "-";
        let display = String(val);
        if (display.length > 250) display = display.slice(0,240) + "…";
        html += `
        <tr>
            <th style="text-align:left;border:1px solid #eee;padding:6px;background:#f7f7f7;vertical-align:top;width:40%">${escapeHtml(key)}</th>
            <td style="border:1px solid #eee;padding:6px;vertical-align:top;">${escapeHtml(display)}</td>
        </tr>`;
    }
    html += "</table></div>";
    return html;
}

// Small helper to escape HTML
function escapeHtml(str){
  return String(str).replace(/[&<>"'`=\/]/g, function(s){ return ({
    '&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;','/':'&#x2F;','`':'&#x60;','=':'&#x3D;'
  })[s]; });
}

// Color map for common obstacle categories (lowercase keys)
const obstacleColorMap = {
  "commercial establishment": "#e6550d",
  "embankment": "#7b3f00",
  "river": "#1e90ff",
  "petrol pump": "#d62828",
  "school": "#7f3fbf",
  "temple": "#ffb703",
  "shop": "#ff8a00",
  "default": "#6b7280"
};

// fallback color generator (stable hash -> color)
function hashColor(str){
  const palette = ["#6b7280","#e6550d","#1e90ff","#7b3f00","#d62828","#7f3fbf","#ff8a00","#2b6cff","#16a34a","#b45309"];
  let h = 0;
  for(let i=0;i<str.length;i++){ h = (h<<5) - h + str.charCodeAt(i); h |= 0; }
  return palette[Math.abs(h) % palette.length];
}

// Determine obstacle style using 'layer' field (or other fallbacks)
function getObstacleStyle(props){
  const candidates = ['layer','Layer','Type_of_Co','Type_of__1','Select_s_1','Site_Obstr','Site_Obs_1','Pipe_Type','Pipe Type','Select_Sid','Site Obstr'];
  let v = null;
  for(const c of candidates){
    if(props && Object.prototype.hasOwnProperty.call(props,c)){
      v = props[c];
      break;
    }
    const found = Object.keys(props||{}).find(k => k.toLowerCase()===c.toLowerCase());
    if(found){ v = props[found]; break; }
  }
  if(!v) v = 'unknown';
  const key = (''+v).toLowerCase().trim();
  const color = obstacleColorMap[key] || hashColor(key);
  return { radius: 6, weight:1, color: "#222", fillColor: color, fillOpacity: 0.9 };
}

// ---------- Generic loader for GeoJSON with reprojection and debug ----------
async function loadLayer(path, targetLayer, options){
    try {
        const res = await fetch(path);
        if(!res.ok){ console.error("Missing:", path, res.status); return; }
        let gj = await res.json();

        // Debug: show first coordinate if available
        try{
          const firstCoord = findFirstCoordinate(gj);
          console.log("First coordinate (raw) for", path, firstCoord);
        }catch(e){}

        gj = reprojectGeoJSON(gj);   // <<< REPROJECT HERE

        try{
          const firstCoord2 = findFirstCoordinate(gj);
          console.log("First coordinate (after reproj) for", path, firstCoord2);
        }catch(e){}

        const layer = L.geoJSON(gj, options);
        layer.addTo(targetLayer);

        try { bounds.extend(layer.getBounds()); } catch(e){ /* ignore */ }
        return layer;
    } catch(err){
        console.error("Error loading", path, err);
    }
}

// helper to find a coordinate inside GeoJSON for debugging
function findFirstCoordinate(gj){
  if(!gj) return null;
  if(gj.type === "FeatureCollection"){
    for(const f of gj.features || []){
      const c = getCoordFromGeometry(f.geometry);
      if(c) return c;
    }
  } else if (gj.type === "Feature"){
    return getCoordFromGeometry(gj.geometry);
  } else if (gj.type && gj.coordinates){
    return getCoordFromGeometry(gj);
  }
  return null;
}
function getCoordFromGeometry(geom){
  if(!geom) return null;
  const coords = geom.coordinates;
  // drill down to first number pair
  function drill(c){
    if(!Array.isArray(c)) return null;
    if(typeof c[0] === 'number') return c;
    for(const i of c){
      const r = drill(i);
      if(r) return r;
    }
    return null;
  }
  return drill(coords);
}

const treeIcon = L.icon({
    iconUrl: "data:image/svg+xml;charset=utf-8," + encodeURIComponent(`
        <svg xmlns='http://www.w3.org/2000/svg' width='32' height='32'>
          <circle cx='16' cy='12' r='10' fill='#2e7d32'/>
          <rect x='14' y='16' width='4' height='12' fill='#5d4037'/>
        </svg>`),
    iconSize: [28, 28],
    iconAnchor: [14, 28]
});

const poleIcon = L.icon({
    iconUrl: "data:image/svg+xml;charset=utf-8," + encodeURIComponent(`
        <svg xmlns='http://www.w3.org/2000/svg' width='32' height='32'>
          <rect x='14' y='4' width='4' height='24' fill='#616161'/>
          <rect x='8' y='8' width='16' height='3' fill='#424242'/>
        </svg>`),
    iconSize: [28, 28],
    iconAnchor: [14, 28]
});

const houseIcon = L.icon({
    iconUrl: "data:image/svg+xml;charset=utf-8," + encodeURIComponent(`
        <svg xmlns='http://www.w3.org/2000/svg' width='32' height='32'>
          <polygon points='16,4 28,14 4,14' fill='#c62828'/>
          <rect x='8' y='14' width='16' height='12' fill='#ef5350'/>
        </svg>`),
    iconSize: [28, 28],
    iconAnchor: [14, 28]
});

const otherIcon = L.icon({
    iconUrl: "data:image/svg+xml;charset=utf-8," + encodeURIComponent(`
        <svg xmlns='http://www.w3.org/2000/svg' width='32' height='32'>
          <circle cx='16' cy='16' r='10' fill='#6b7280'/>
        </svg>`),
    iconSize: [26, 26],
    iconAnchor: [13, 26]
});

// ---------- Load base layers ----------
loadLayer("ESR.geojson", lgESR, {
    pointToLayer:(feature, latlng) => {
        return L.marker(latlng, {icon: tankIcon});
    },
    onEachFeature:(feature, layer) => {
        const name = feature.properties && (feature.properties.Entity || feature.properties.entity || feature.properties.Name || feature.properties.RefName) || "ESR";
        layer.bindTooltip(name,{permanent:true,className:"feature-label",direction:"right",offset:[10,0]});
        layer.on("click", () => layer.bindPopup(buildPopup(feature.properties)).openPopup());
    }
});

loadLayer("Completed Pipeline.geojson", lgCompleted, {
    style: function(f){ return {color:"#fb154b",weight:5,opacity:0.9}; },
    onEachFeature: function(feature, layer){
        const d = feature.properties && (feature.properties.description || feature.properties.Description || feature.properties.Name) || '';
        if(d) layer.bindTooltip(d,{className:"feature-label"});
        layer.on("click", ()=> layer.bindPopup(buildPopup(feature.properties)).openPopup());
    }
});

loadLayer("Proposed Pipeline.geojson", lgProposed, {
    style: function(f){ return {color:"#6ce8fb",weight:4,dashArray:"8,6",opacity:0.95}; },
    onEachFeature: function(feature, layer){
        layer.on("click", ()=> layer.bindPopup(buildPopup(feature.properties)).openPopup());
    }
});

loadLayer("Road.geojson", lgRoads, {
    style: function(feature){
        const props = feature.properties || {};
        const layerField = (props.Layer || props.layer || props.layername || props.TYPE || props.Type || "").toString().toLowerCase();
        if(layerField.includes('rail')) return {color:'#8b5a2b',weight:3,dashArray:'2,6',opacity:0.95};
        if(layerField.includes('water')) return {color:'#2b8cff',weight:3,dashArray:'1,6',opacity:0.95};
        return {color:'#65707a',weight:3,opacity:0.95};
    },
    onEachFeature: function(feature, layer){
        layer.on("click", ()=> layer.bindPopup(buildPopup(feature.properties)).openPopup());
    }
});

loadLayer("Zoning.geojson", lgZoning, {
    style: function(){ return { color:'#0b1220', weight:1, fillColor:'#0b1220', fillOpacity:0.08 }; },
    onEachFeature: function(feature, layer){
        const nm = feature.properties && (feature.properties.Entity || feature.properties.Name || feature.properties.name) || "";
        if(nm) layer.bindTooltip(nm,{permanent:true,className:"feature-label",direction:"center"});
        layer.on("click", ()=> layer.bindPopup(buildPopup(feature.properties)).openPopup());
    }
});

loadLayer("RWM_CWM.geojson", lgRWM, {
    style:function(){ return { color:'#6b7280', weight:2, fillOpacity:0.05 }; },
    onEachFeature:function(feature, layer){ layer.on("click",()=> layer.bindPopup(buildPopup(feature.properties)).openPopup()); }
});

// ---------- Obstacles: symbolise by layer field (case-insensitive) ----------
loadLayer("Obstacles.geojson", lgObstacles, {
    pointToLayer: function(feature, latlng){
        const props = feature.properties || {};
        const raw = (props.layer || props.Layer || "").toString().toLowerCase();

        let icon;
        if (raw.includes("tree"))         icon = treeIcon;
        else if (raw.includes("pole"))    icon = poleIcon;
        else if (raw.includes("house") || raw.includes("building")) icon = houseIcon;
        else                              icon = otherIcon;

        return L.marker(latlng, { icon: icon });
    },

    onEachFeature: function(feature, layer){
        // Tooltip
        const props = feature.properties || {};
        const label = props.Name_for_C || props.Site_Obstr || props.Details_of || "";
        if(label) layer.bindTooltip(label, { className:"feature-label", direction:"right" });

        // Full popup
        layer.on("click", ()=> layer.bindPopup(buildPopup(props)).openPopup());
    }
});

// LAYER TOGGLES
document.getElementById("chkESR").onchange = e => e.target.checked ? map.addLayer(lgESR) : map.removeLayer(lgESR);
document.getElementById("chkCompleted").onchange = e => e.target.checked ? map.addLayer(lgCompleted) : map.removeLayer(lgCompleted);
document.getElementById("chkProposed").onchange = e => e.target.checked ? map.addLayer(lgProposed) : map.removeLayer(lgProposed);
document.getElementById("chkRoads").onchange = e => e.target.checked ? map.addLayer(lgRoads) : map.removeLayer(lgRoads);
document.getElementById("chkZoning").onchange = e => e.target.checked ? map.addLayer(lgZoning) : map.removeLayer(lgZoning);
document.getElementById("chkRWM").onchange = e => e.target.checked ? map.addLayer(lgRWM) : map.removeLayer(lgRWM);
document.getElementById("chkObstacles").onchange = e => e.target.checked ? map.addLayer(lgObstacles) : map.removeLayer(lgObstacles);

// BASEMAP TOGGLES
document.getElementById("bmOSM").onchange = ()=>{ map.addLayer(osm); map.removeLayer(sat);};
document.getElementById("bmSAT").onchange = ()=>{ map.addLayer(sat); map.removeLayer(osm);};

// FIT ALL
document.getElementById("zoomAll").onclick = ()=>{ try{ if(bounds.isValid()) map.fitBounds(bounds.pad(0.15)); } catch(e){ console.warn(e); } };
// MOBILE SIDEBAR TOGGLE
document.getElementById("toggleSidebar").onclick = () => {
    const sb = document.getElementById("sidebar");
    sb.classList.toggle("open");
};

</script>

</body>
</html>
