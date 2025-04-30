---
title: Example Map with Leaflet
toc: false
---
# Example Map with Leaflet

```js
const div = display(document.createElement("div"));
div.style = "height: 400px;";

const map = L.map(div)
  .setView([46.519653, 6.632273], 13);

L.tileLayer("https://tile.openstreetmap.org/{z}/{x}/{y}.png", {
  attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>'
})
  .addTo(map);

L.marker([46.519653, 6.632273])
  .addTo(map)
  .bindPopup("Lausanne")
  .openPopup();
```

## Documentation
- Leaflet documentation:<br />https://leafletjs.com/reference.html
- Observable Framework documentation:<br />https://observablehq.com/framework/lib/leaflet