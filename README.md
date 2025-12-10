# Jorhat TWSS WebGIS Dashboard

A lightweight, mobile-friendly WebGIS dashboard built for the Jorhat Town Water Supply Scheme.  
The application visualizes completed pipelines, proposed pipelines, ESR tanks, zoning boundaries, roads, and obstacle points using Leaflet.js.

## Features
- Interactive map with toggleable layers
- ESR tank symbols and dynamic labeling
- Completed pipeline and proposed pipeline styling
- Transparent zoning polygons with labels
- Obstacle points with custom symbology
- Mobile-responsive sidebar toggle
- Automatically reprojects UTM (EPSG:32646) GeoJSON to WGS84

## Tech Stack
- **Leaflet.js**
- **JavaScript**
- **HTML/CSS**
- **GeoJSON**
- **Proj4.js**
- **QGIS (for data preparation)**

## How to Use
1. Download or clone the repository.
2. Place all GeoJSON files in the same folder as `index.html`.
3. Open `index.html` in a browser.
4. Toggle layers, switch basemaps, and zoom to data.

## Folder Structure
project/
│── index.html
│── ESR.geojson
│── Completed Pipeline.geojson
│── Proposed Pipeline.geojson
│── Zoning.geojson
│── Roads.geojson
│── Obstacles.geojson

## Author
**Samsul (GIS Analyst)**  
PHED Headquarters, Assam  
Specializing in WebGIS, automation, and GIS-based infrastructure mapping.
