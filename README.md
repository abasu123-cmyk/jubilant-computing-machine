# directions
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>Navigation Software</title>
<meta name="viewport" content="initial-scale=1,maximum-scale=1,user-scalable=no">
<link href="https://api.mapbox.com/mapbox-gl-js/v3.10.0/mapbox-gl.css" rel="stylesheet">
<script src="https://api.mapbox.com/mapbox-gl-js/v3.10.0/mapbox-gl.js"></script>
<script src="https://api.mapbox.com/mapbox-gl-js/plugins/mapbox-gl-geocoder/v4.7.2/mapbox-gl-geocoder.min.js"></script>
<link rel="stylesheet" href="https://api.mapbox.com/mapbox-gl-js/plugins/mapbox-gl-geocoder/v4.7.2/mapbox-gl-geocoder.css">
<script async defer src="https://maps.googleapis.com/maps/api/js?key=YOUR_GOOGLE_MAPS_API_KEY&libraries=places"></script>
<style>
body { margin: 0; padding: 0; }
#map { position: absolute; top: 0; bottom: 0; width: 100%; }
#sidebar {
    position: absolute;
    top: 10px;
    left: 10px;
    width: 350px;
    background: rgba(255, 255, 255, 0.9);
    padding: 15px;
    border-radius: 10px;
    box-shadow: 0px 4px 10px rgba(0, 0, 0, 0.2);
    font-family: Arial, sans-serif;
    z-index: 1000;
}
#routes-list { max-height: 150px; overflow-y: auto; margin-top: 10px; }
</style>
</head>
<body>
    <div id="sidebar">
        <b>From:</b> <div id="start-geocoder"></div><br>
        <b>Destination:</b> <div id="end-geocoder"></div><br>
        <b>Travel Mode:</b> 
        <select id="mode">
            <option value="driving">Car</option>
            <option value="walking">Walk</option>
            <option value="cycling">Bicycle</option>
        </select><br><br>
        <b>Estimated Time:</b> <span id="time">N/A</span><br><br>
        <b>Next Step:</b> <span id="next-step">N/A</span><br><br>
        <b>Route Options:</b>
        <ul id="routes-list"></ul>
    </div>
    <div id="map"></div> 

<script>
    mapboxgl.accessToken = 'pk.eyJ1IjoiYWJhc3UxMDAiLCJhIjoiY203cGE1aTN5MDF3aTJpc2piZm1ibnEwOCJ9.QAceeshyu617lQ6GmKDGLw';
    const map = new mapboxgl.Map({
        container: 'map',
        style: 'mapbox://styles/mapbox/streets-v11',
        center: [-74.5, 40],
        zoom: 9
    });

    map.addControl(new mapboxgl.GeolocateControl({ positionOptions: { enableHighAccuracy: true }, trackUserLocation: true }));

    const startGeocoder = new MapboxGeocoder({ accessToken: mapboxgl.accessToken, mapboxgl: mapboxgl, placeholder: "Enter starting location" });
    document.getElementById('start-geocoder').appendChild(startGeocoder.onAdd(map));

    const endGeocoder = new MapboxGeocoder({ accessToken: mapboxgl.accessToken, mapboxgl: mapboxgl, placeholder: "Enter destination" });
    document.getElementById('end-geocoder').appendChild(endGeocoder.onAdd(map));

    let startCoords = null;
    let endCoords = null;
    let currentRoute = null;

    startGeocoder.on('result', (event) => { startCoords = event.result.center; updateRoute(); });
    endGeocoder.on('result', (event) => { endCoords = event.result.center; updateRoute(); });

    document.getElementById('mode').addEventListener('change', updateRoute);

    function updateRoute() {
        if (startCoords && endCoords) {
            const mode = document.getElementById('mode').value;
            getMapboxRoutes(startCoords, endCoords, mode);
        }
    }

    function getMapboxRoutes(start, end, mode) {
        const url = `https://api.mapbox.com/directions/v5/mapbox/${mode}/${start[0]},${start[1]};${end[0]},${end[1]}?steps=true&geometries=geojson&access_token=${mapboxgl.accessToken}`;

        fetch(url)
            .then(response => response.json())
            .then(data => {
                if (data.routes.length > 0) {
                    currentRoute = data.routes[0];
                    drawRouteOnMap(currentRoute);
                    startNavigation(currentRoute);
                }
            })
            .catch(error => console.error('Error fetching Mapbox routes:', error));
    }

    function drawRouteOnMap(route) {
        const geojson = { type: 'Feature', properties: {}, geometry: route.geometry };
        if (map.getSource('route')) {
            map.getSource('route').setData(geojson);
        } else {
            map.addSource('route', { type: 'geojson', data: geojson });
            map.addLayer({ id: 'route', type: 'line', source: 'route', layout: { 'line-join': 'round', 'line-cap': 'round' }, paint: { 'line-color': '#ff0000', 'line-width': 5 } });
        }
        document.getElementById('time').innerText = (route.duration / 60).toFixed(2) + ' mins';
    }

    function startNavigation(route) {
        const steps = route.legs[0].steps;
        let stepIndex = 0;
        function updateStep(position) {
            if (stepIndex < steps.length) {
                const step = steps[stepIndex];
                document.getElementById('next-step').innerText = step.maneuver.instruction;
                speak(step.maneuver.instruction);
                
                const stepEnd = step.maneuver.location;
                const distance = getDistance(position, stepEnd);
                if (distance < 30) stepIndex++;
            }
        }
        navigator.geolocation.watchPosition((pos) => {
            const position = [pos.coords.longitude, pos.coords.latitude];
            updateStep(position);
        }, console.error, { enableHighAccuracy: true });
    }

    function getDistance(pos1, pos2) {
        const R = 6371000;
        const dLat = (pos2[1] - pos1[1]) * Math.PI / 180;
        const dLon = (pos2[0] - pos1[0]) * Math.PI / 180;
        const a = Math.sin(dLat / 2) * Math.sin(dLat / 2) + Math.cos(pos1[1] * Math.PI / 180) * Math.cos(pos2[1] * Math.PI / 180) * Math.sin(dLon / 2) * Math.sin(dLon / 2);
        return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    }

    function speak(text) {
        const utterance = new SpeechSynthesisUtterance(text);
        speechSynthesis.speak(utterance);
    }
</script>
</body>
</html>
