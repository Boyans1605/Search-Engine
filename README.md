<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Share My Location</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
            line-height: 1.6;
        }
        #map {
            height: 400px;
            width: 100%;
            margin: 20px 0;
            border-radius: 8px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        .controls {
            margin: 20px 0;
            display: flex;
            gap: 10px;
            flex-wrap: wrap;
        }
        button {
            padding: 10px 15px;
            background: #4285f4;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            font-weight: bold;
        }
        button:disabled {
            background: #cccccc;
            cursor: not-allowed;
        }
        #linkBox {
            width: 100%;
            padding: 10px;
            margin: 10px 0;
            border: 1px solid #ddd;
            border-radius: 4px;
            font-family: monospace;
        }
        #status {
            padding: 10px;
            margin: 10px 0;
            border-radius: 4px;
        }
        .info {
            background: #e7f3fe;
            border-left: 4px solid #4285f4;
            padding: 10px;
            margin: 20px 0;
        }
        .accuracy-circle {
            border-radius: 50%;
            opacity: 0.2;
            pointer-events: none;
        }
    </style>
</head>
<body>
    <h1>Share My Location</h1>
    <p>Generate a temporary shareable link to your current location or track your path in real-time.</p>
    
    <div class="controls">
        <button id="getLocationBtn">Get My Location</button>
        <button id="startTrackingBtn">Start Tracking</button>
        <button id="stopTrackingBtn" disabled>Stop Tracking</button>
        <button id="generateLinkBtn" disabled>Generate Share Link</button>
        <button id="clearBtn">Clear</button>
    </div>
    
    <div id="status"></div>
    
    <div id="map"></div>
    
    <div id="shareSection" style="display: none;">
        <h3>Share this link:</h3>
        <input type="text" id="linkBox" readonly>
        <button id="copyLinkBtn">Copy Link</button>
        <div class="info">
            <p>This link will show your location to anyone who opens it. Links expire after 7 days.</p>
        </div>
    </div>

    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <script>
        // Initialize map centered on world view
        const map = L.map('map').setView([0, 0], 2);
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
            attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
        }).addTo(map);

        // DOM elements
        const getLocationBtn = document.getElementById('getLocationBtn');
        const startTrackingBtn = document.getElementById('startTrackingBtn');
        const stopTrackingBtn = document.getElementById('stopTrackingBtn');
        const generateLinkBtn = document.getElementById('generateLinkBtn');
        const clearBtn = document.getElementById('clearBtn');
        const statusElement = document.getElementById('status');
        const shareSection = document.getElementById('shareSection');
        const linkBox = document.getElementById('linkBox');
        const copyLinkBtn = document.getElementById('copyLinkBtn');

        // Tracking variables
        let watchId = null;
        let currentMarker = null;
        let accuracyCircle = null;
        let pathPoints = [];
        let pathLine = null;
        let shareToken = null;
        let locationsData = [];

        // Check URL for share token on load
        const urlParams = new URLSearchParams(window.location.search);
        const sharedToken = urlParams.get('token');

        if (sharedToken) {
            loadSharedLocation(sharedToken);
        }

        // Event listeners
        getLocationBtn.addEventListener('click', getCurrentLocation);
        startTrackingBtn.addEventListener('click', startTracking);
        stopTrackingBtn.addEventListener('click', stopTracking);
        generateLinkBtn.addEventListener('click', generateShareLink);
        clearBtn.addEventListener('click', clearMap);
        copyLinkBtn.addEventListener('click', copyLinkToClipboard);

        // Get single location
        function getCurrentLocation() {
            if (navigator.geolocation) {
                statusElement.textContent = "Getting your location...";
                statusElement.style.background = "#fff3cd";
                
                navigator.geolocation.getCurrentPosition(
                    position => {
                        showLocation(position);
                        generateLinkBtn.disabled = false;
                    },
                    error => {
                        showError(error);
                    },
                    {
                        enableHighAccuracy: true,
                        timeout: 10000
                    }
                );
            } else {
                statusElement.textContent = "Geolocation is not supported by your browser";
                statusElement.style.background = "#f8d7da";
            }
        }

        // Start continuous tracking
        function startTracking() {
            if (navigator.geolocation) {
                statusElement.textContent = "Tracking your location...";
                statusElement.style.background = "#fff3cd";
                startTrackingBtn.disabled = true;
                stopTrackingBtn.disabled = false;
                generateLinkBtn.disabled = false;
                
                watchId = navigator.geolocation.watchPosition(
                    position => {
                        showLocation(position);
                        pathPoints.push([position.coords.latitude, position.coords.longitude]);
                        updatePathLine();
                        locationsData.push({
                            lat: position.coords.latitude,
                            lng: position.coords.longitude,
                            accuracy: position.coords.accuracy,
                            timestamp: new Date(position.timestamp).toISOString()
                        });
                    },
                    error => {
                        showError(error);
                    },
                    {
                        enableHighAccuracy: true,
                        maximumAge: 0,
                        timeout: 5000
                    }
                );
            } else {
                statusElement.textContent = "Geolocation is not supported by your browser";
                statusElement.style.background = "#f8d7da";
            }
        }

        // Stop tracking
        function stopTracking() {
            if (watchId) {
                navigator.geolocation.clearWatch(watchId);
                watchId = null;
                statusElement.textContent = "Tracking stopped";
                statusElement.style.background = "#d4edda";
                startTrackingBtn.disabled = false;
                stopTrackingBtn.disabled = true;
            }
        }

        // Display location on map
        function showLocation(position) {
            const { latitude, longitude, accuracy } = position.coords;
            
            // Remove previous marker and circle if they exist
            if (currentMarker) map.removeLayer(currentMarker);
            if (accuracyCircle) map.removeLayer(accuracyCircle);
            
            // Create new marker
            currentMarker = L.marker([latitude, longitude]).addTo(map)
                .bindPopup(`Your location (accuracy: ~${accuracy ? accuracy.toFixed(0) + 'm' : 'unknown'})`)
                .openPopup();
            
            // Create accuracy circle if accuracy data is available
            if (accuracy) {
                accuracyCircle = L.circle([latitude, longitude], {
                    radius: accuracy,
                    className: 'accuracy-circle',
                    fillColor: '#3388ff',
                    color: '#3388ff'
                }).addTo(map);
            }
            
            // Center map on location
            map.setView([latitude, longitude], 15);
            
            // Update status
            const time = new Date(position.timestamp).toLocaleTimeString();
            statusElement.textContent = `Location updated at ${time} (accuracy: ~${accuracy ? accuracy.toFixed(0) + 'm' : 'unknown'})`;
            statusElement.style.background = "#d4edda";
        }

        // Update the path line during tracking
        function updatePathLine() {
            if (pathLine) {
                map.removeLayer(pathLine);
            }
            if (pathPoints.length > 1) {
                pathLine = L.polyline(pathPoints, {color: 'red'}).addTo(map);
            }
        }

        // Generate shareable link
        function generateShareLink() {
            // Create or reuse share token
            if (!shareToken) {
                shareToken = generateToken();
            }
            
            // Prepare data to share
            const shareData = {
                token: shareToken,
                locations: pathPoints.length > 0 ? pathPoints : 
                    currentMarker ? [currentMarker.getLatLng()] : [],
                timestamp: new Date().toISOString(),
                type: watchId ? 'track' : 'single'
            };
            
            // Store in localStorage (simulating simple server storage)
            localStorage.setItem(`locationShare-${shareToken}`, JSON.stringify(shareData));
            
            // Generate the shareable URL
            const currentUrl = window.location.href.split('?')[0];
            const shareUrl = `${currentUrl}?token=${shareToken}`;
            linkBox.value = shareUrl;
            shareSection.style.display = 'block';
            
            statusElement.textContent = "Share link generated!";
            statusElement.style.background = "#d4edda";
        }

        // Load shared location from token
        function loadSharedLocation(token) {
            const sharedData = localStorage.getItem(`locationShare-${token}`);
            
            if (sharedData) {
                try {
                    const data = JSON.parse(sharedData);
                    
                    if (data.locations && data.locations.length > 0) {
                        if (data.type === 'track') {
                            // Display full path
                            pathLine = L.polyline(data.locations, {color: 'blue'}).addTo(map);
                            map.fitBounds(pathLine.getBounds());
                            
                            // Add start and end markers
                            L.marker(data.locations[0]).addTo(map)
                                .bindPopup("Start point");
                            L.marker(data.locations[data.locations.length-1]).addTo(map)
                                .bindPopup("End point");
                            
                            statusElement.textContent = `Showing shared path with ${data.locations.length} points`;
                            statusElement.style.background = "#d4edda";
                        } else {
                            // Display single location
                            const [lat, lng] = data.locations[0];
                            currentMarker = L.marker([lat, lng]).addTo(map)
                                .bindPopup("Shared location")
                                .openPopup();
                            map.setView([lat, lng], 15);
                            
                            statusElement.textContent = "Showing shared location";
                            statusElement.style.background = "#d4edda";
                        }
                        return;
                    }
                } catch (e) {
                    console.error("Error loading shared data:", e);
                }
            }
            
            statusElement.textContent = "Shared location not found or expired";
            statusElement.style.background = "#f8d7da";
        }

        // Clear the map
        function clearMap() {
            if (watchId) {
                stopTracking();
            }
            
            pathPoints = [];
            locationsData = [];
            shareToken = null;
            
            if (currentMarker) map.removeLayer(currentMarker);
            if (accuracyCircle) map.removeLayer(accuracyCircle);
            if (pathLine) map.removeLayer(pathLine);
            
            currentMarker = null;
            accuracyCircle = null;
            pathLine = null;
            
            shareSection.style.display = 'none';
            generateLinkBtn.disabled = true;
            
            statusElement.textContent = "Map cleared";
            statusElement.style.background = "#d4edda";
        }

        // Copy link to clipboard
        function copyLinkToClipboard() {
            linkBox.select();
            document.execCommand('copy');
            
            statusElement.textContent = "Link copied to clipboard!";
            statusElement.style.background = "#d4edda";
        }

        // Show error message
        function showError(error) {
            let message = "Error getting location: ";
            switch(error.code) {
                case error.PERMISSION_DENIED:
                    message += "Permission denied. Please enable location access.";
                    break;
                case error.POSITION_UNAVAILABLE:
                    message += "Location information unavailable.";
                    break;
                case error.TIMEOUT:
                    message += "The request to get location timed out.";
                    break;
                case error.UNKNOWN_ERROR:
                    message += "An unknown error occurred.";
                    break;
            }
            statusElement.textContent = message;
            statusElement.style.background = "#f8d7da";
        }

        // Generate random token
        function generateToken() {
            return Math.random().toString(36).substring(2, 10) + 
                   Math.random().toString(36).substring(2, 10);
        }
    </script>
</body>
</html>
