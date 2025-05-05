---
title: Cadastre Rénové
toc: false
---

```js
// Explicit import of leaflet to avoid issues with the Leaflet.heat plugin
import L from "npm:leaflet";
```

```js
// Wait for L to be defined before importing the Leaflet.heat plugin
// This is necessary because Leaflet.heat depends on the L variable being defined
if (L === undefined) console.error("L is undefined");

// Leaflet.heat: https://github.com/Leaflet/Leaflet.heat/
import "./plugins/leaflet-heat.js";
```

# Cadastre Rénové
Cette page présente le cadastre rénové de Lausanne (1889).

## Carte

```js
const geojson = FileAttachment("./data/lausanne-1888-cadastre-renove-points-20250409.geojson").json()
```
<!-- Create the map container -->
<div id="map-container" style="height: 500px; margin: 1em 0 2em 0;"></div>

```js
// Create Map and Layer - Runs Once
function createMapAndLayer(mapContainer, geojsonData) {
    const map = L.map(mapContainer).setView([46.55, 6.65], 11);

    const osmLayer = L.tileLayer("https://tile.openstreetmap.org/{z}/{x}/{y}.png", {
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>'
    });

    const cartoLayer = L.tileLayer("https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}@2x.png", {
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>'
    }).addTo(map);

    // Crate a control to switch between layers
    const layerControl = L.control.layers().addTo(map);

    // Add the OSM and Carto layers to the control
    layerControl.addBaseLayer(osmLayer, "OSM");
    layerControl.addBaseLayer(cartoLayer, "Carto");

    // Store map from geom_id -> leaflet layer instance
    const featureLayersMap = new Map();

    const geoJsonLayer = L.geoJSON(geojsonData, {
        pointToLayer: function (feature, latlng) {
            return L.circleMarker(latlng, {
                radius: 4,
                fillColor: "#CCC",
                color: "#666",
                weight: 1,
                opacity: 1,
                fillOpacity: 0.5
            });
        },
        onEachFeature: function (feature, layer) {
            // Store a reference to the layer using its merge_id
            // As a single merge_id can be used for multiple features, we use a Set
            // to store all layers associated with that merge_id
            if (feature.properties && feature.properties.merge_id) {
                const merge_id = feature.properties.merge_id;

                // Check if the merge_id already exists in the map
                if (!featureLayersMap.has(merge_id)) {
                    featureLayersMap.set(merge_id, new Set());
                }
                // Add the layer to the Set associated with the merge_id
                featureLayersMap.get(merge_id).add(layer);

                // Bind a popup to the layer with the details of the corresponding entries
                const entries = mergeIDMap.get(merge_id);
                if (entries) {
                    const popupContent = entries.map(entry => {
                        return columnNames.map(column => {
                            return `<strong>${column}:</strong> ${entry[column]}<br>`;
                        }).join("") + "<hr>";
                    }).join("");
                    layer.bindPopup(popupContent);
                }
            }
        }
    }).addTo(map);
    
    layerControl.addOverlay(geoJsonLayer, "Points");

    // Return the the map instance, the layer group, and the mapping
    return { map, layerControl, geoJsonLayer, featureLayersMap };
}

// Call the creation function and store the results
const mapElements = createMapAndLayer("map-container", geojson);
```

```js
// Reactive Update Cell - Runs when filteredMergeIDsSet changes
function updateMapFilter(geoJsonLayer, featureLayersMap, filteredMergeIDsSet) {
    let featuresAdded = 0;
    let featuresRemoved = 0;

    // Iterate through all the layers we stored
    featureLayersMap.forEach((layerSet, merge_id) => {
        const shouldBeVisible = filteredMergeIDsSet.has(merge_id);
        // LayerSet may contain multiple layers for the same merge_id
        layerSet.forEach(layer => {
            const isVisible = geoJsonLayer.hasLayer(layer);
            if (shouldBeVisible && !isVisible) {
                // If the layer is not already added, add it
                geoJsonLayer.addLayer(layer);
                featuresAdded++;
            } else if (!shouldBeVisible && isVisible) {
                // If the layer is currently displayed but should not be, remove it
                geoJsonLayer.removeLayer(layer);
                featuresRemoved++;
            }
        });
    });
}

// Call the update function. This cell depends on filteredMergeIDsSet and mapElements.
const mapUpdateStatus = updateMapFilter(mapElements.geoJsonLayer, mapElements.featureLayersMap, filteredMergeIDsSet)
```

```js
// Map the GeoJSON data to an array of entries matching the required pattern for the heatmap
// e.g. [50.5, 30.5, 0.2] // lat, lng, intensity
const heatmapData = geojson.features.map(feature => {
    const coords = feature.geometry.coordinates;
    const lat = coords[1];
    const lng = coords[0];
    const intensity = 0.5;
    return [lat, lng, intensity];
});
```

```js
// Create a heatmap layer using the heatmapData
const heatmapLayer = L.heatLayer(heatmapData, {
    radius: 10,
    blur: 15,
});

// Add the heatmap layer to the layer control
mapElements.layerControl.addOverlay(heatmapLayer, "Heatmap");
```

## Registre

```js
const registre = FileAttachment("./data/lausanne-1888-cadastre-renove-registre-20250410.csv").csv()
```

```js
// Create a list of column names
const columnNames = Object.keys(registre[0]);
```

```js
// Create a Map of the merge_id to corresponding entries in the registre
// This is used to populate the popups with the relevant data for each point
const mergeIDMap = new Map();
registre.forEach(entry => {
    const merge_id = entry.merge_id;
    if (!mergeIDMap.has(merge_id)) {
        mergeIDMap.set(merge_id, []);
    }
    mergeIDMap.get(merge_id).push(entry);
});
```

_Utilisez le champ de recherche pour filtrer les entrées du registre. Les points affichés sur la carte sont mis à jour en fonction du résultat de la recherche._

```js
const searchResults = view(Inputs.search(registre))
```

```js
Inputs.table(searchResults)
```

```js
const filteredMergeIDs = searchResults.map(r => r.merge_id)
```

```js
const filteredMergeIDsSet = new Set(filteredMergeIDs);
```
