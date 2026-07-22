(() => {
    "use strict";

    const DEFAULT_LOCATION = {
        name: "Chennai",
        city: "Chennai",
        district: "Chennai District",
        admin1: "Tamil Nadu",
        country: "India",
        latitude: 13.0827,
        longitude: 80.2707
    };

    const state = {
        map: null,
        marker: null,
        surfaceLayer: null,
        boundaryLayer: null,
        labelsOverlay: null,
        baseLayers: null,
        selectedLayer: "heat",
        selectedForecastHour: 0,
        globalForecast: [],
        globalPoints: [],
        selectedLocation: DEFAULT_LOCATION,
        selectedForecast: null,
        lastGlobalRefresh: 0,
        searchController: null,
        reverseController: null,
        forecastController: null,
        lastNominatimRequest: 0,
        toastTimer: null,
        activeBaseLayer: null,
        fallbackActivated: false,
        surfaceVisible: true,
        globalAbortController: null
    };

    const elements = {
        stars: document.getElementById("stars"),
        utcClock: document.getElementById("utcClock"),
        utcDate: document.getElementById("utcDate"),
        refreshButton: document.getElementById("refreshButton"),
        searchForm: document.getElementById("searchForm"),
        citySearch: document.getElementById("citySearch"),
        searchResults: document.getElementById("searchResults"),
        locationButton: document.getElementById("locationButton"),
        locationName: document.getElementById("locationName"),
        locationMeta: document.getElementById("locationMeta"),
        locationHierarchy: document.getElementById("locationHierarchy"),
        currentIcon: document.getElementById("currentIcon"),
        currentTemperature: document.getElementById("currentTemperature"),
        currentCondition: document.getElementById("currentCondition"),
        currentUpdated: document.getElementById("currentUpdated"),
        feelsLike: document.getElementById("feelsLike"),
        humidity: document.getElementById("humidity"),
        windSpeed: document.getElementById("windSpeed"),
        pressure: document.getElementById("pressure"),
        cloudCover: document.getElementById("cloudCover"),
        precipitation: document.getElementById("precipitation"),
        layerButtons: [...document.querySelectorAll(".layer-button")],
        forecastHour: document.getElementById("forecastHour"),
        forecastHourLabel: document.getElementById("forecastHourLabel"),
        mapLegend: document.getElementById("mapLegend"),
        mapModeTitle: document.getElementById("mapModeTitle"),
        mapStatus: document.getElementById("mapStatus"),
        coordinateReadout: document.getElementById("coordinateReadout"),
        placeReadout: document.getElementById("placeReadout"),
        mapLoading: document.getElementById("mapLoading"),
        hourlyChart: document.getElementById("hourlyChart"),
        hourlyForecast: document.getElementById("hourlyForecast"),
        dailyForecast: document.getElementById("dailyForecast"),
        forecastSummary: document.getElementById("forecastSummary"),
        autoRefreshText: document.getElementById("autoRefreshText"),
        toast: document.getElementById("toast"),
        worldViewButton: document.getElementById("worldViewButton"),
        weatherToggleButton: document.getElementById("weatherToggleButton"),
        mapServiceNotice: document.getElementById("mapServiceNotice"),
        earthGateway: document.getElementById("earthGateway"),
        enterMapButton: document.getElementById("enterMapButton"),
        enterMapTextButton: document.getElementById("enterMapTextButton"),
        weatherApplication: document.getElementById("weatherApplication")
    };

    const WMO = {
        0: { label: "Clear sky", day: "☀️", night: "🌙" },
        1: { label: "Mainly clear", day: "🌤️", night: "🌙" },
        2: { label: "Partly cloudy", day: "⛅", night: "☁️" },
        3: { label: "Overcast", day: "☁️", night: "☁️" },
        45: { label: "Fog", day: "🌫️", night: "🌫️" },
        48: { label: "Rime fog", day: "🌫️", night: "🌫️" },
        51: { label: "Light drizzle", day: "🌦️", night: "🌧️" },
        53: { label: "Drizzle", day: "🌦️", night: "🌧️" },
        55: { label: "Dense drizzle", day: "🌧️", night: "🌧️" },
        56: { label: "Freezing drizzle", day: "🌧️", night: "🌧️" },
        57: { label: "Dense freezing drizzle", day: "🌧️", night: "🌧️" },
        61: { label: "Light rain", day: "🌦️", night: "🌧️" },
        63: { label: "Rain", day: "🌧️", night: "🌧️" },
        65: { label: "Heavy rain", day: "🌧️", night: "🌧️" },
        66: { label: "Freezing rain", day: "🌧️", night: "🌧️" },
        67: { label: "Heavy freezing rain", day: "🌧️", night: "🌧️" },
        71: { label: "Light snow", day: "🌨️", night: "🌨️" },
        73: { label: "Snow", day: "❄️", night: "❄️" },
        75: { label: "Heavy snow", day: "❄️", night: "❄️" },
        77: { label: "Snow grains", day: "🌨️", night: "🌨️" },
        80: { label: "Light showers", day: "🌦️", night: "🌧️" },
        81: { label: "Showers", day: "🌧️", night: "🌧️" },
        82: { label: "Violent showers", day: "⛈️", night: "⛈️" },
        85: { label: "Snow showers", day: "🌨️", night: "🌨️" },
        86: { label: "Heavy snow showers", day: "❄️", night: "❄️" },
        95: { label: "Thunderstorm", day: "⛈️", night: "⛈️" },
        96: { label: "Thunderstorm with hail", day: "⛈️", night: "⛈️" },
        99: { label: "Severe thunderstorm", day: "⛈️", night: "⛈️" }
    };


    class WeatherSurfaceLayer extends L.Layer {
        constructor(options = {}) {
            super();
            L.setOptions(this, options);
            this._forecast = [];
            this._points = [];
            this._matrix = [];
            this._latitudes = [];
            this._longitudes = [];
            this._mode = "heat";
            this._hour = 0;
            this._frame = null;
            this._visible = true;
        }

        onAdd(map) {
            this._map = map;
            this._canvas = L.DomUtil.create("canvas", "leaflet-weather-surface");
            this._canvas.setAttribute("aria-hidden", "true");
            this._canvas.style.pointerEvents = "none";
            const weatherPane = map.getPane("weatherPane") || map.getPanes().overlayPane;
            weatherPane.appendChild(this._canvas);

            map.on("moveend zoomend resize viewreset", this._scheduleRedraw, this);
            this._scheduleRedraw();
        }

        onRemove(map) {
            map.off("moveend zoomend resize viewreset", this._scheduleRedraw, this);
            cancelAnimationFrame(this._frame);
            this._canvas?.remove();
            this._canvas = null;
            this._map = null;
        }

        setForecast(forecast, points, mode, hour) {
            this._forecast = forecast || [];
            this._points = points || [];
            this._mode = mode;
            this._hour = hour;
            this._buildMatrix();
            this._scheduleRedraw();
        }

        setVisible(visible) {
            this._visible = Boolean(visible);
            if (this._canvas) {
                this._canvas.style.display = this._visible ? "block" : "none";
            }
            if (this._visible) {
                this._scheduleRedraw();
            }
        }

        _buildMatrix() {
            this._latitudes = [...new Set(this._points.map(point => point.latitude))]
                .sort((a, b) => a - b);
            this._longitudes = [...new Set(this._points.map(point => point.longitude))]
                .sort((a, b) => a - b);

            const lookup = new Map();
            this._points.forEach((point, index) => {
                lookup.set(`${point.latitude}|${point.longitude}`, this._forecast[index]);
            });

            this._matrix = this._latitudes.map(latitude =>
                this._longitudes.map(longitude => {
                    const forecast = lookup.get(`${latitude}|${longitude}`);
                    const hourly = forecast?.hourly || {};
                    const hour = clamp(
                        this._hour,
                        0,
                        Math.max(0, (hourly.time?.length || 1) - 1)
                    );

                    return {
                        temperature: numberOrNull(hourly.temperature_2m?.[hour]),
                        rainProbability: numberOrNull(hourly.precipitation_probability?.[hour]),
                        rain: numberOrNull(hourly.rain?.[hour]),
                        snowfall: numberOrNull(hourly.snowfall?.[hour]),
                        wind: numberOrNull(hourly.wind_speed_10m?.[hour])
                    };
                })
            );
        }

        _scheduleRedraw() {
            if (!this._map || !this._canvas) {
                return;
            }

            cancelAnimationFrame(this._frame);
            this._frame = requestAnimationFrame(() => this._redraw());
        }

        _redraw() {
            if (!this._map || !this._canvas || !this._matrix.length || !this._visible) {
                return;
            }

            const size = this._map.getSize();
            if (!size.x || !size.y) {
                return;
            }

            const topLeft = this._map.containerPointToLayerPoint([0, 0]);
            L.DomUtil.setPosition(this._canvas, topLeft);

            const pixelRatio = Math.min(window.devicePixelRatio || 1, 2);
            this._canvas.width = Math.round(size.x * pixelRatio);
            this._canvas.height = Math.round(size.y * pixelRatio);
            this._canvas.style.width = `${size.x}px`;
            this._canvas.style.height = `${size.y}px`;

            const context = this._canvas.getContext("2d", { alpha: true });
            context.setTransform(pixelRatio, 0, 0, pixelRatio, 0, 0);
            context.clearRect(0, 0, size.x, size.y);

            const sampleSize = size.x <= 620 ? 8 : size.x <= 1100 ? 6 : 5;
            const sampleWidth = Math.max(1, Math.ceil(size.x / sampleSize));
            const sampleHeight = Math.max(1, Math.ceil(size.y / sampleSize));
            const buffer = document.createElement("canvas");
            buffer.width = sampleWidth;
            buffer.height = sampleHeight;
            const bufferContext = buffer.getContext("2d", { alpha: true });
            const image = bufferContext.createImageData(sampleWidth, sampleHeight);

            for (let y = 0; y < sampleHeight; y += 1) {
                for (let x = 0; x < sampleWidth; x += 1) {
                    const point = [
                        Math.min(size.x - 1, x * sampleSize + sampleSize / 2),
                        Math.min(size.y - 1, y * sampleSize + sampleSize / 2)
                    ];
                    const latlng = this._map.containerPointToLatLng(point);
                    const values = this._sample(latlng.lat, latlng.lng);
                    const pixel = surfacePixel(this._mode, values);
                    const offset = (y * sampleWidth + x) * 4;

                    image.data[offset] = pixel.r;
                    image.data[offset + 1] = pixel.g;
                    image.data[offset + 2] = pixel.b;
                    image.data[offset + 3] = pixel.a;
                }
            }

            bufferContext.putImageData(image, 0, 0);
            context.imageSmoothingEnabled = true;
            context.drawImage(buffer, 0, 0, size.x, size.y);
        }

        _sample(latitude, longitude) {
            if (!this._latitudes.length || !this._longitudes.length) {
                return null;
            }

            const lat = clamp(
                latitude,
                this._latitudes[0],
                this._latitudes[this._latitudes.length - 1]
            );
            const normalizedLongitude = normalizeLongitude(longitude);
            const lon = clamp(
                normalizedLongitude,
                this._longitudes[0],
                this._longitudes[this._longitudes.length - 1]
            );

            const latPosition = findGridPosition(this._latitudes, lat);
            const lonPosition = findGridPosition(this._longitudes, lon);

            const northWest = this._matrix[latPosition.lower]?.[lonPosition.lower];
            const northEast = this._matrix[latPosition.lower]?.[lonPosition.upper];
            const southWest = this._matrix[latPosition.upper]?.[lonPosition.lower];
            const southEast = this._matrix[latPosition.upper]?.[lonPosition.upper];

            return interpolateWeatherValues(
                northWest,
                northEast,
                southWest,
                southEast,
                lonPosition.ratio,
                latPosition.ratio
            );
        }
    }

    const LAYER_INFO = {
        heat: {
            title: "Temperature prediction",
            legend: [
                ["#4a66ff", "≤ 0°"],
                ["#24c9ff", "10°"],
                ["#42e69a", "20°"],
                ["#ffd166", "30°"],
                ["#ff4d4d", "≥ 40°"]
            ]
        },
        rain: {
            title: "Rain probability and intensity",
            legend: [
                ["#071a40", "0%"],
                ["#1766d2", "20%"],
                ["#18a9e6", "40%"],
                ["#48e0ff", "70%"],
                ["#e4fbff", "100%"]
            ]
        },
        snow: {
            title: "Snowfall prediction",
            legend: [
                ["#17324c", "0 cm"],
                ["#2e7ab6", "0.1"],
                ["#72c9f2", "0.5"],
                ["#c6f3ff", "1.5"],
                ["#ffffff", "3+"]
            ]
        },
        spring: {
            title: "Spring-like weather zones",
            legend: [
                ["#203b45", "Cold"],
                ["#4cb77c", "Mild"],
                ["#7ee49b", "Ideal"],
                ["#d3ef72", "Warm"],
                ["#e07a55", "Hot"]
            ]
        }
    };

    function initialize() {
        if (document.body.classList.contains("earth-mode")) {
            document.documentElement.style.overflow = "hidden";
        }
        createStars();
        updateClock();
        setInterval(updateClock, 1000);
        connectGatewayEvents();

        if (typeof window.L === "undefined") {
            elements.mapStatus.textContent = "Leaflet map library did not load";
            elements.mapLoading.classList.add("show");
            elements.mapLoading.innerHTML =
                "<strong>Unable to initialize the live world map</strong>" +
                "<small>Confirm that you are online, open this project through VS Code Live Server, and allow cdn.jsdelivr.net in browser or antivirus settings.</small>";
            showToast("Leaflet could not load from the CDN.", true);
            return;
        }

        initializeMap();
        connectEvents();
        setLegend();
        loadGlobalForecast();

        // Load Chennai weather without moving away from the full-world starting view.
        selectLocation(DEFAULT_LOCATION, { announce: false });

        setInterval(() => {
            refreshAll(false);
        }, 30 * 60 * 1000);
    }

    function connectGatewayEvents() {
        elements.enterMapButton?.addEventListener("click", enterInteractiveMap);
        elements.enterMapTextButton?.addEventListener("click", enterInteractiveMap);

        document.addEventListener("keydown", event => {
            if (event.key === "Escape" && !elements.earthGateway?.hidden) {
                enterInteractiveMap();
            }
        });
    }

    function enterInteractiveMap() {
        if (!elements.earthGateway || !elements.weatherApplication) {
            return;
        }

        elements.earthGateway.classList.add("leaving");
        document.body.classList.remove("earth-mode");
        document.documentElement.style.overflow = "";
        elements.weatherApplication.removeAttribute("inert");
        elements.weatherApplication.setAttribute("aria-hidden", "false");

        window.setTimeout(() => {
            elements.earthGateway.hidden = true;
            elements.earthGateway.classList.remove("leaving");
        }, 740);

        window.setTimeout(() => {
            if (!state.map) {
                return;
            }

            state.map.invalidateSize({ pan: false });
            state.map.fitBounds([[-62, -178], [78, 178]], {
                padding: [8, 8],
                animate: true,
                duration: 0.85
            });
            elements.placeReadout.textContent = "Choose any country, state, district, city or village to view its live forecast.";
        }, 220);
    }

    function showRealisticEarthView() {
        if (!elements.earthGateway || !elements.weatherApplication) {
            return;
        }

        window.scrollTo({ top: 0, behavior: "auto" });
        elements.earthGateway.hidden = false;
        elements.earthGateway.classList.remove("leaving");
        document.body.classList.add("earth-mode");
        document.documentElement.style.overflow = "hidden";
        elements.weatherApplication.setAttribute("inert", "");
        elements.weatherApplication.setAttribute("aria-hidden", "true");

        window.setTimeout(() => {
            elements.enterMapButton?.focus({ preventScroll: true });
        }, 120);
    }

    function createStars() {
        const count = window.innerWidth < 700 ? 45 : 90;
        const fragment = document.createDocumentFragment();

        for (let index = 0; index < count; index += 1) {
            const star = document.createElement("span");
            star.style.left = `${Math.random() * 100}%`;
            star.style.top = `${Math.random() * 100}%`;
            star.style.setProperty("--duration", `${2 + Math.random() * 5}s`);
            star.style.animationDelay = `${Math.random() * 4}s`;
            fragment.appendChild(star);
        }

        elements.stars.appendChild(fragment);
    }

    function updateClock() {
        const now = new Date();

        elements.utcClock.textContent = `${now.toLocaleTimeString("en-GB", {
            timeZone: "UTC",
            hour12: false
        })} UTC`;

        elements.utcDate.textContent = now.toLocaleDateString("en-GB", {
            timeZone: "UTC",
            weekday: "short",
            day: "2-digit",
            month: "short",
            year: "numeric"
        });
    }

    function initializeMap() {
        const worldBounds = L.latLngBounds([[-85.0511, -180], [85.0511, 180]]);

        state.map = L.map("weatherMap", {
            center: [18, 0],
            zoom: window.innerWidth < 620 ? 1.35 : 2.15,
            minZoom: 1,
            maxZoom: 19,
            zoomSnap: 0.25,
            zoomDelta: 0.5,
            worldCopyJump: false,
            maxBounds: worldBounds,
            maxBoundsViscosity: 0.85,
            zoomControl: true,
            preferCanvas: true
        });

        state.map.createPane("weatherPane");
        state.map.getPane("weatherPane").style.zIndex = "330";
        state.map.getPane("weatherPane").style.pointerEvents = "none";

        state.map.createPane("labelPane");
        state.map.getPane("labelPane").style.zIndex = "445";
        state.map.getPane("labelPane").classList.add("leaflet-label-pane");
        state.map.getPane("labelPane").style.pointerEvents = "none";

        state.map.createPane("boundaryPane");
        state.map.getPane("boundaryPane").style.zIndex = "460";
        state.map.getPane("boundaryPane").style.pointerEvents = "none";

        const sharedTileOptions = {
            noWrap: true,
            bounds: worldBounds,
            updateWhenIdle: true,
            updateWhenZooming: false,
            keepBuffer: 3,
            crossOrigin: true
        };

        const detailedOSM = L.tileLayer("https://tile.openstreetmap.org/{z}/{x}/{y}.png", {
            ...sharedTileOptions,
            maxZoom: 20,
            maxNativeZoom: 19,
            attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
        });

        const cartoVoyager = L.tileLayer(
            "https://{s}.basemaps.cartocdn.com/rastertiles/voyager/{z}/{x}/{y}{r}.png",
            {
                ...sharedTileOptions,
                subdomains: "abcd",
                maxZoom: 20,
                maxNativeZoom: 20,
                attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors &copy; <a href="https://carto.com/attributions">CARTO</a>'
            }
        );

        const satellite = L.tileLayer(
            "https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}",
            {
                ...sharedTileOptions,
                maxZoom: 19,
                attribution: "Tiles &copy; Esri and contributors"
            }
        );

        const terrain = L.tileLayer("https://{s}.tile.opentopomap.org/{z}/{x}/{y}.png", {
            ...sharedTileOptions,
            subdomains: "abc",
            maxZoom: 17,
            attribution: "Map data &copy; OpenStreetMap contributors, SRTM | Map style &copy; OpenTopoMap"
        });

        state.labelsOverlay = L.tileLayer(
            "https://{s}.basemaps.cartocdn.com/rastertiles/voyager_only_labels/{z}/{x}/{y}{r}.png",
            {
                ...sharedTileOptions,
                subdomains: "abcd",
                maxZoom: 20,
                maxNativeZoom: 20,
                pane: "labelPane",
                opacity: 0.96,
                attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors &copy; <a href="https://carto.com/attributions">CARTO</a>'
            }
        );

        state.baseLayers = {
            "OpenStreetMap detailed": detailedOSM,
            "CARTO Voyager backup": cartoVoyager,
            "Satellite imagery": satellite,
            "Terrain": terrain
        };

        state.activeBaseLayer = detailedOSM;
        detailedOSM.addTo(state.map);

        const layerControl = L.control.layers(
            state.baseLayers,
            { "Extra high-contrast labels": state.labelsOverlay },
            {
                position: "topright",
                collapsed: true
            }
        ).addTo(state.map);

        L.control.scale({
            position: "bottomright",
            imperial: false,
            metric: true,
            maxWidth: 140
        }).addTo(state.map);

        state.surfaceLayer = new WeatherSurfaceLayer().addTo(state.map);

        let primaryTileErrors = 0;
        detailedOSM.on("tileload", () => {
            primaryTileErrors = 0;
            if (!state.fallbackActivated) {
                elements.mapStatus.textContent = "Detailed OpenStreetMap online";
            }
        });

        detailedOSM.on("tileerror", () => {
            primaryTileErrors += 1;
            if (primaryTileErrors >= 4 && !state.fallbackActivated && state.map.hasLayer(detailedOSM)) {
                state.fallbackActivated = true;
                state.map.removeLayer(detailedOSM);
                cartoVoyager.addTo(state.map);
                state.activeBaseLayer = cartoVoyager;
                elements.mapStatus.textContent = "Using backup real-world map tiles";
                elements.mapServiceNotice.hidden = false;
                showToast("The primary map source was unavailable, so a backup map was activated.", true);
            }
        });

        state.map.on("baselayerchange", event => {
            state.activeBaseLayer = event.layer;
            elements.mapStatus.textContent = `${event.name} active`;
            setTimeout(() => state.map.invalidateSize({ pan: false }), 50);
        });

        state.map.on("mousemove", event => {
            elements.coordinateReadout.textContent =
                `Latitude ${formatCoordinate(event.latlng.lat, "N", "S")} · ` +
                `Longitude ${formatCoordinate(event.latlng.lng, "E", "W")} · Zoom ${state.map.getZoom().toFixed(1)}`;
        });

        state.map.on("mouseout", () => {
            elements.coordinateReadout.textContent = "Move over the map to inspect coordinates";
        });

        state.map.on("zoomend", () => {
            const zoom = state.map.getZoom();
            let detail = "countries and large regions";
            if (zoom >= 5) detail = "states, provinces and major cities";
            if (zoom >= 8) detail = "districts, towns and road networks";
            if (zoom >= 12) detail = "villages, neighbourhoods, streets and local places";
            elements.mapStatus.textContent = `Map detail: ${detail}`;
        });

        state.map.on("click", async event => {
            elements.placeReadout.textContent = "Identifying this point from OpenStreetMap…";

            const resolved = await reverseGeocode(event.latlng.lat, event.latlng.lng);
            const location = resolved || {
                name: "Selected map point",
                admin1: "",
                district: "",
                country: "",
                latitude: event.latlng.lat,
                longitude: event.latlng.lng
            };

            selectLocation(location, {
                drawBoundary: true
            });
        });

        state.map.whenReady(() => {
            state.map.fitBounds([[-62, -178], [78, 178]], {
                padding: [8, 8],
                animate: false
            });
            state.map.invalidateSize({ pan: false });
        });

        [100, 350, 900].forEach(delay => {
            setTimeout(() => state.map.invalidateSize({ pan: false }), delay);
        });
    }

    function connectEvents() {
        elements.searchForm.addEventListener("submit", event => {
            event.preventDefault();
            searchLocations(elements.citySearch.value.trim());
        });

        elements.citySearch.addEventListener("input", () => {
            if (!elements.citySearch.value.trim()) {
                elements.searchResults.innerHTML = "";
            }
        });

        elements.locationButton.addEventListener("click", useCurrentLocation);

        elements.worldViewButton.addEventListener("click", showRealisticEarthView);

        elements.weatherToggleButton.addEventListener("click", () => {
            state.surfaceVisible = !state.surfaceVisible;
            state.surfaceLayer.setVisible(state.surfaceVisible);
            elements.weatherToggleButton.classList.toggle("active", state.surfaceVisible);
            elements.weatherToggleButton.setAttribute("aria-pressed", String(state.surfaceVisible));
            elements.weatherToggleButton.textContent = state.surfaceVisible ? "☁ Weather on" : "☁ Weather off";
            showToast(state.surfaceVisible ? "Weather prediction surface enabled." : "Weather surface hidden for a clearer map view.");
        });

        elements.refreshButton.addEventListener("click", () => {
            refreshAll(true);
        });

        elements.layerButtons.forEach(button => {
            button.addEventListener("click", () => {
                state.selectedLayer = button.dataset.layer;
                elements.layerButtons.forEach(item => {
                    item.classList.toggle("active", item === button);
                });

                setLegend();
                renderGlobalLayer();
            });
        });

        elements.forecastHour.addEventListener("input", () => {
            state.selectedForecastHour = Number(elements.forecastHour.value);
            updateForecastHourLabel();
            renderGlobalLayer();
        });

        window.addEventListener("resize", debounce(() => {
            state.map?.invalidateSize({ pan: false });
            if (state.selectedForecast) {
                renderHourlyChart(state.selectedForecast);
            }
        }, 150));
    }

    async function refreshAll(announce = true) {
        const tasks = [
            loadGlobalForecast(true),
            fetchLocalForecast(state.selectedLocation)
        ];

        const results = await Promise.allSettled(tasks);
        const successful = results.filter(result => result.status === "fulfilled").length;

        elements.autoRefreshText.textContent = `Last refreshed ${new Date().toLocaleTimeString([], {
            hour: "2-digit",
            minute: "2-digit"
        })}`;

        if (announce) {
            showToast(
                successful === tasks.length
                    ? "Weather predictions refreshed."
                    : "Some forecast services could not be refreshed.",
                successful !== tasks.length
            );
        }
    }

    async function searchLocations(query) {
        if (!query) {
            showToast("Enter a country, state, district, city or village.", true);
            return;
        }

        if (state.searchController) {
            state.searchController.abort();
        }

        state.searchController = new AbortController();
        elements.searchResults.innerHTML = '<div class="search-result">Searching detailed OpenStreetMap places…</div>';

        try {
            const url = new URL("https://nominatim.openstreetmap.org/search");
            url.searchParams.set("q", query);
            url.searchParams.set("format", "jsonv2");
            url.searchParams.set("addressdetails", "1");
            url.searchParams.set("namedetails", "1");
            url.searchParams.set("limit", "8");
            url.searchParams.set("accept-language", "en");

            const results = await nominatimFetch(url, state.searchController.signal);

            if (!results.length) {
                elements.searchResults.innerHTML =
                    '<div class="search-result">No matching country, state, district, city or village found.</div>';
                return;
            }

            elements.searchResults.innerHTML = results.map((item, index) => {
                const location = buildLocationFromNominatim(item);
                return `
                    <button class="search-result" type="button" data-result-index="${index}">
                        <strong>${escapeHTML(location.name)}</strong>
                        <small>${escapeHTML(formatAddressHierarchy(location))}</small>
                        <span class="result-type">${escapeHTML(locationTypeLabel(item))}</span>
                    </button>
                `;
            }).join("");

            [...elements.searchResults.querySelectorAll("[data-result-index]")].forEach(button => {
                button.addEventListener("click", async () => {
                    const selected = results[Number(button.dataset.resultIndex)];
                    const location = buildLocationFromNominatim(selected);
                    elements.searchResults.innerHTML = "";
                    elements.citySearch.value = location.name;

                    await selectLocation(location, {
                        zoom: chooseLocationZoom(location),
                        drawBoundary: true
                    });
                });
            });
        } catch (error) {
            if (error.name === "AbortError") {
                return;
            }

            elements.searchResults.innerHTML =
                '<div class="search-result">Detailed place search is temporarily unavailable.</div>';
            showToast(error.message, true);
        }
    }

    function useCurrentLocation() {
        if (!navigator.geolocation) {
            showToast("Geolocation is not supported by this browser.", true);
            return;
        }

        elements.locationButton.disabled = true;
        elements.locationButton.textContent = "Locating…";

        navigator.geolocation.getCurrentPosition(
            async position => {
                elements.locationButton.disabled = false;
                elements.locationButton.innerHTML = "<span aria-hidden=\"true\">⌖</span> Use my current location";

                elements.placeReadout.textContent = "Identifying your local area…";
                const resolved = await reverseGeocode(
                    position.coords.latitude,
                    position.coords.longitude
                );

                await selectLocation(resolved || {
                    name: "Current location",
                    admin1: "",
                    district: "",
                    country: "",
                    latitude: position.coords.latitude,
                    longitude: position.coords.longitude
                }, {
                    zoom: 12,
                    drawBoundary: true
                });
            },
            error => {
                elements.locationButton.disabled = false;
                elements.locationButton.innerHTML = "<span aria-hidden=\"true\">⌖</span> Use my current location";
                showToast(`Unable to access location: ${error.message}`, true);
            },
            {
                enableHighAccuracy: true,
                timeout: 12000,
                maximumAge: 10 * 60 * 1000
            }
        );
    }

    async function selectLocation(location, options = {}) {
        state.selectedLocation = location;

        updateLocationHeader(location);
        placeMarker(location);

        if (options.drawBoundary) {
            await drawLocationBoundary(location);
        }

        if (options.zoom) {
            state.map.flyTo(
                [location.latitude, location.longitude],
                options.zoom,
                { duration: 1.25 }
            );
        }

        elements.placeReadout.textContent = formatAddressHierarchy(location) || displayLocationName(location);
        await fetchLocalForecast(location);

        if (options.announce !== false) {
            showToast(`Forecast loaded for ${displayLocationName(location)}.`);
        }
    }

    function updateLocationHeader(location) {
        elements.locationName.textContent = displayLocationName(location);
        elements.locationMeta.textContent =
            `${formatCoordinate(location.latitude, "N", "S")}, ` +
            `${formatCoordinate(location.longitude, "E", "W")}`;
        elements.locationHierarchy.textContent =
            formatAddressHierarchy(location) || "Administrative details unavailable for this coordinate";
    }

    function placeMarker(location) {
        const icon = L.divIcon({
            className: "",
            html: '<div class="weather-marker"><span></span></div>',
            iconSize: [28, 28],
            iconAnchor: [14, 28]
        });

        if (!state.marker) {
            state.marker = L.marker([location.latitude, location.longitude], { icon })
                .addTo(state.map);
        } else {
            state.marker.setLatLng([location.latitude, location.longitude]);
        }

        state.marker.bindPopup(`
            <div class="weather-popup">
                <strong>${escapeHTML(displayLocationName(location))}</strong>
                <span>${location.latitude.toFixed(3)}°, ${location.longitude.toFixed(3)}°</span>
                <span>Loading precise local forecast…</span>
            </div>
        `);
    }


    async function reverseGeocode(latitude, longitude) {
        if (state.reverseController) {
            state.reverseController.abort();
        }

        state.reverseController = new AbortController();

        try {
            const url = new URL("https://nominatim.openstreetmap.org/reverse");
            url.searchParams.set("lat", String(latitude));
            url.searchParams.set("lon", String(longitude));
            url.searchParams.set("format", "jsonv2");
            url.searchParams.set("addressdetails", "1");
            url.searchParams.set("namedetails", "1");
            url.searchParams.set("zoom", "16");
            url.searchParams.set("accept-language", "en");

            const result = await nominatimFetch(url, state.reverseController.signal);
            return result?.lat ? buildLocationFromNominatim(result) : null;
        } catch (error) {
            if (error.name !== "AbortError") {
                elements.placeReadout.textContent =
                    "Place name unavailable; weather is still available for the selected coordinates.";
            }
            return null;
        }
    }

    async function nominatimFetch(url, signal) {
        const elapsed = Date.now() - state.lastNominatimRequest;
        if (elapsed < 1050) {
            await wait(1050 - elapsed);
        }

        state.lastNominatimRequest = Date.now();
        const response = await fetch(url, {
            signal,
            headers: {
                "Accept": "application/json",
                "Accept-Language": "en"
            }
        });

        if (!response.ok) {
            throw new Error(`Detailed map search failed (${response.status})`);
        }

        return response.json();
    }

    function buildLocationFromNominatim(item) {
        const address = item.address || {};
        const name =
            item.namedetails?.name ||
            item.name ||
            address.village ||
            address.town ||
            address.city ||
            address.municipality ||
            address.county ||
            address.state ||
            address.country ||
            "Selected place";

        return {
            name,
            neighbourhood: address.neighbourhood || address.suburb || address.quarter || "",
            village: address.village || address.hamlet || address.isolated_dwelling || "",
            town: address.town || "",
            city: address.city || address.city_district || address.municipality || "",
            district: address.state_district || address.county || address.district || "",
            admin1: address.state || address.region || "",
            country: address.country || "",
            countryCode: String(address.country_code || "").toUpperCase(),
            postcode: address.postcode || "",
            latitude: Number(item.lat),
            longitude: Number(item.lon),
            osmType: item.osm_type || "",
            osmId: item.osm_id || "",
            category: item.category || item.class || "",
            placeType: item.type || address.place || "",
            boundingbox: item.boundingbox || null,
            geojson: item.geojson || null,
            displayName: item.display_name || ""
        };
    }

    function formatAddressHierarchy(location) {
        return [
            location.neighbourhood,
            location.village,
            location.town,
            location.city,
            location.district,
            location.admin1,
            location.country,
            location.postcode
        ]
            .filter(Boolean)
            .filter((item, index, array) => array.indexOf(item) === index)
            .join(" · ");
    }

    function locationTypeLabel(item) {
        const type = item.addresstype || item.type || item.category || item.class || "place";
        return String(type).replaceAll("_", " ");
    }

    function chooseLocationZoom(location) {
        const type = String(location.placeType || "").toLowerCase();
        if (["country"].includes(type)) return 5;
        if (["state", "region", "province"].includes(type)) return 7;
        if (["county", "state_district", "district"].includes(type)) return 9;
        if (["city", "municipality"].includes(type)) return 11;
        if (["town", "village", "hamlet", "suburb", "neighbourhood"].includes(type)) return 14;
        return 11;
    }

    async function drawLocationBoundary(location) {
        if (state.boundaryLayer) {
            state.map.removeLayer(state.boundaryLayer);
            state.boundaryLayer = null;
        }

        if (!Array.isArray(location.boundingbox) || location.boundingbox.length !== 4) {
            return;
        }

        const south = Number(location.boundingbox[0]);
        const north = Number(location.boundingbox[1]);
        const west = Number(location.boundingbox[2]);
        const east = Number(location.boundingbox[3]);

        if (![south, north, west, east].every(Number.isFinite)) {
            return;
        }

        state.boundaryLayer = L.rectangle([[south, west], [north, east]], {
            pane: "boundaryPane",
            className: "selected-admin-boundary",
            color: "#46e7ff",
            weight: 2,
            opacity: 0.9,
            dashArray: "7 6",
            fillColor: "#46e7ff",
            fillOpacity: 0.035,
            interactive: false
        }).addTo(state.map);
    }

    function wait(milliseconds) {
        return new Promise(resolve => setTimeout(resolve, milliseconds));
    }

    async function fetchLocalForecast(location) {
        if (state.forecastController) {
            state.forecastController.abort();
        }

        state.forecastController = new AbortController();
        setCurrentLoadingState();

        try {
            const url = new URL("https://api.open-meteo.com/v1/forecast");
            url.searchParams.set("latitude", String(location.latitude));
            url.searchParams.set("longitude", String(location.longitude));
            url.searchParams.set(
                "current",
                [
                    "temperature_2m",
                    "apparent_temperature",
                    "relative_humidity_2m",
                    "precipitation",
                    "weather_code",
                    "cloud_cover",
                    "pressure_msl",
                    "wind_speed_10m",
                    "wind_direction_10m",
                    "is_day"
                ].join(",")
            );
            url.searchParams.set(
                "hourly",
                [
                    "temperature_2m",
                    "apparent_temperature",
                    "relative_humidity_2m",
                    "precipitation_probability",
                    "rain",
                    "snowfall",
                    "weather_code",
                    "wind_speed_10m",
                    "cloud_cover",
                    "is_day"
                ].join(",")
            );
            url.searchParams.set(
                "daily",
                [
                    "weather_code",
                    "temperature_2m_max",
                    "temperature_2m_min",
                    "sunrise",
                    "sunset",
                    "precipitation_probability_max",
                    "rain_sum",
                    "snowfall_sum",
                    "wind_gusts_10m_max",
                    "uv_index_max"
                ].join(",")
            );
            url.searchParams.set("forecast_hours", "24");
            url.searchParams.set("forecast_days", "7");
            url.searchParams.set("timezone", "auto");

            const response = await fetch(url, { signal: state.forecastController.signal });
            if (!response.ok) {
                throw new Error(`Forecast request failed (${response.status})`);
            }

            const data = await response.json();
            state.selectedForecast = data;

            renderCurrentWeather(data, location);
            renderHourlyForecast(data);
            renderDailyForecast(data);
            renderHourlyChart(data);
            updateMarkerPopup(data, location);
        } catch (error) {
            if (error.name === "AbortError") {
                return;
            }

            elements.currentCondition.textContent = "Forecast unavailable";
            elements.currentUpdated.textContent = "Try refreshing the project";
            showToast(error.message, true);
        }
    }

    function setCurrentLoadingState() {
        elements.currentCondition.textContent = "Loading forecast…";
        elements.currentUpdated.textContent = "Contacting weather models";
        elements.forecastSummary.textContent = "Updating location";
    }

    function renderCurrentWeather(data, location) {
        const current = data.current || {};
        const weather = weatherInfo(current.weather_code, current.is_day);

        elements.currentIcon.textContent = weather.icon;
        elements.currentTemperature.textContent = formatTemperature(current.temperature_2m);
        elements.currentCondition.textContent = weather.label;
        elements.currentUpdated.textContent = `Updated ${formatLocalTime(current.time)}`;
        elements.feelsLike.textContent = formatTemperature(current.apparent_temperature);
        elements.humidity.textContent = formatNumber(current.relative_humidity_2m, "%");
        elements.windSpeed.textContent = formatNumber(current.wind_speed_10m, " km/h");
        elements.pressure.textContent = formatNumber(current.pressure_msl, " hPa");
        elements.cloudCover.textContent = formatNumber(current.cloud_cover, "%");
        elements.precipitation.textContent = formatNumber(current.precipitation, " mm", 1);

        const hourly = data.hourly || {};
        const temperatures = (hourly.temperature_2m || []).map(Number).filter(Number.isFinite);
        const rainProbabilities = (hourly.precipitation_probability || []).map(Number).filter(Number.isFinite);
        const maximum = temperatures.length ? Math.max(...temperatures) : null;
        const minimum = temperatures.length ? Math.min(...temperatures) : null;
        const rainPeak = rainProbabilities.length ? Math.max(...rainProbabilities) : null;

        elements.forecastSummary.textContent =
            Number.isFinite(maximum) && Number.isFinite(minimum)
                ? `${Math.round(minimum)}° to ${Math.round(maximum)}° · Rain peak ${Math.round(rainPeak || 0)}%`
                : "24-hour forecast loaded";

        elements.locationMeta.textContent =
            `${formatCoordinate(location.latitude, "N", "S")}, ` +
            `${formatCoordinate(location.longitude, "E", "W")} · ${data.timezone_abbreviation || ""}`;
    }

    function renderHourlyForecast(data) {
        const hourly = data.hourly || {};
        const count = Math.min(24, hourly.time?.length || 0);

        elements.hourlyForecast.innerHTML = Array.from({ length: count }, (_, index) => {
            const weather = weatherInfo(
                hourly.weather_code?.[index],
                hourly.is_day?.[index]
            );

            return `
                <article class="hour-card ${index === 0 ? "now" : ""}">
                    <time>${index === 0 ? "Now" : formatHour(hourly.time[index])}</time>
                    <span class="hour-icon" aria-hidden="true">${weather.icon}</span>
                    <strong>${formatTemperature(hourly.temperature_2m?.[index])}</strong>
                    <small>Rain ${Math.round(hourly.precipitation_probability?.[index] || 0)}%</small>
                    <small>Wind ${Math.round(hourly.wind_speed_10m?.[index] || 0)} km/h</small>
                </article>
            `;
        }).join("");
    }

    function renderDailyForecast(data) {
        const daily = data.daily || {};
        const count = Math.min(7, daily.time?.length || 0);

        elements.dailyForecast.innerHTML = Array.from({ length: count }, (_, index) => {
            const weather = weatherInfo(daily.weather_code?.[index], 1);
            const date = new Date(`${daily.time[index]}T12:00:00`);

            return `
                <article class="day-card">
                    <span class="day-name">${index === 0 ? "Today" : date.toLocaleDateString([], { weekday: "short" })}</span>
                    <span class="day-icon" aria-hidden="true">${weather.icon}</span>
                    <strong>${escapeHTML(weather.label)}</strong>
                    <div class="day-temperature">
                        <span>${formatTemperature(daily.temperature_2m_max?.[index])}</span>
                        <span>${formatTemperature(daily.temperature_2m_min?.[index])}</span>
                    </div>
                    <small>Rain ${Math.round(daily.precipitation_probability_max?.[index] || 0)}%</small>
                    <small>UV ${formatNumber(daily.uv_index_max?.[index], "", 1)}</small>
                    <small>Gust ${Math.round(daily.wind_gusts_10m_max?.[index] || 0)} km/h</small>
                </article>
            `;
        }).join("");
    }

    function renderHourlyChart(data) {
        const hourly = data.hourly || {};
        const temperatures = (hourly.temperature_2m || []).slice(0, 24);
        const rain = (hourly.precipitation_probability || []).slice(0, 24);
        const times = (hourly.time || []).slice(0, 24);

        if (!temperatures.length) {
            elements.hourlyChart.innerHTML = "";
            return;
        }

        const width = 1000;
        const height = 270;
        const padding = { left: 58, right: 34, top: 34, bottom: 45 };
        const chartWidth = width - padding.left - padding.right;
        const chartHeight = height - padding.top - padding.bottom;

        const minTemp = Math.floor(Math.min(...temperatures) - 2);
        const maxTemp = Math.ceil(Math.max(...temperatures) + 2);
        const tempRange = Math.max(1, maxTemp - minTemp);

        const x = index => padding.left + (index / Math.max(1, temperatures.length - 1)) * chartWidth;
        const yTemp = value => padding.top + ((maxTemp - value) / tempRange) * chartHeight;
        const yRain = value => padding.top + chartHeight - (value / 100) * chartHeight;

        const temperaturePoints = temperatures
            .map((value, index) => `${x(index).toFixed(1)},${yTemp(value).toFixed(1)}`)
            .join(" ");

        const horizontalGrid = Array.from({ length: 5 }, (_, index) => {
            const y = padding.top + (index / 4) * chartHeight;
            const value = Math.round(maxTemp - (index / 4) * tempRange);

            return `
                <line x1="${padding.left}" y1="${y}" x2="${width - padding.right}" y2="${y}" stroke="rgba(139,190,220,.12)" />
                <text x="${padding.left - 12}" y="${y + 4}" text-anchor="end" fill="#91a9bb" font-size="12">${value}°</text>
            `;
        }).join("");

        const rainBars = rain.map((value, index) => {
            const barWidth = Math.max(5, chartWidth / rain.length - 5);
            const top = yRain(value);
            const barHeight = padding.top + chartHeight - top;

            return `
                <rect
                    x="${x(index) - barWidth / 2}"
                    y="${top}"
                    width="${barWidth}"
                    height="${barHeight}"
                    rx="4"
                    fill="rgba(41,121,255,.24)"
                />
            `;
        }).join("");

        const labels = times.map((time, index) => {
            if (index % 3 !== 0 && index !== times.length - 1) {
                return "";
            }

            return `
                <text
                    x="${x(index)}"
                    y="${height - 16}"
                    text-anchor="middle"
                    fill="#91a9bb"
                    font-size="12"
                >${formatHour(time)}</text>
            `;
        }).join("");

        const points = temperatures.map((value, index) => `
            <circle cx="${x(index)}" cy="${yTemp(value)}" r="4" fill="#dffcff" stroke="#00c8ff" stroke-width="3">
                <title>${formatHour(times[index])}: ${Math.round(value)}°C, rain ${Math.round(rain[index] || 0)}%</title>
            </circle>
        `).join("");

        elements.hourlyChart.innerHTML = `
            <defs>
                <linearGradient id="temperatureLine" x1="0" x2="1">
                    <stop offset="0%" stop-color="#46e7ff" />
                    <stop offset="55%" stop-color="#ffd166" />
                    <stop offset="100%" stop-color="#ff5b5b" />
                </linearGradient>
                <filter id="lineGlow" x="-20%" y="-20%" width="140%" height="140%">
                    <feGaussianBlur stdDeviation="3" result="blur" />
                    <feMerge>
                        <feMergeNode in="blur" />
                        <feMergeNode in="SourceGraphic" />
                    </feMerge>
                </filter>
            </defs>
            ${horizontalGrid}
            ${rainBars}
            <polyline
                points="${temperaturePoints}"
                fill="none"
                stroke="url(#temperatureLine)"
                stroke-width="5"
                stroke-linecap="round"
                stroke-linejoin="round"
                filter="url(#lineGlow)"
            />
            ${points}
            ${labels}
            <text x="${width - 35}" y="22" text-anchor="end" fill="#46e7ff" font-size="12">Temperature</text>
            <text x="${width - 35}" y="40" text-anchor="end" fill="#6d9fff" font-size="12">Rain probability bars</text>
        `;
    }

    function updateMarkerPopup(data, location) {
        if (!state.marker || !data.current) {
            return;
        }

        const current = data.current;
        const weather = weatherInfo(current.weather_code, current.is_day);

        state.marker.bindPopup(`
            <div class="weather-popup">
                <strong>${escapeHTML(displayLocationName(location))}</strong>
                <span>${escapeHTML(formatAddressHierarchy(location))}</span>
                <span>${weather.icon} ${escapeHTML(weather.label)}</span>
                <span>${formatTemperature(current.temperature_2m)} · Feels ${formatTemperature(current.apparent_temperature)}</span>
                <span>Wind ${Math.round(current.wind_speed_10m || 0)} km/h · Humidity ${Math.round(current.relative_humidity_2m || 0)}%</span>
            </div>
        `);
    }

    function buildGlobalPoints() {
        // A stable global sampling grid. Requests are sent in small batches to avoid
        // browser URL limits and API failures from one oversized request.
        const latitudes = [-75, -55, -35, -15, 5, 25, 45, 65, 80];
        const longitudes = Array.from({ length: 13 }, (_, index) => -180 + index * 30);
        const points = [];

        latitudes.forEach(latitude => {
            longitudes.forEach(longitude => {
                points.push({ latitude, longitude });
            });
        });

        return points;
    }

    async function fetchGlobalForecastBatch(points, signal) {
        const url = new URL("https://api.open-meteo.com/v1/forecast");
        url.searchParams.set("latitude", points.map(point => point.latitude).join(","));
        url.searchParams.set("longitude", points.map(point => point.longitude).join(","));
        url.searchParams.set(
            "hourly",
            [
                "temperature_2m",
                "precipitation_probability",
                "rain",
                "snowfall",
                "wind_speed_10m"
            ].join(",")
        );
        url.searchParams.set("forecast_hours", "25");
        url.searchParams.set("timezone", "UTC");
        url.searchParams.set("cell_selection", "nearest");

        const response = await fetch(url, { signal });
        if (!response.ok) {
            throw new Error(`Global forecast batch failed (${response.status})`);
        }

        const data = await response.json();
        return Array.isArray(data) ? data : [data];
    }

    async function loadGlobalForecast(force = false) {
        const age = Date.now() - state.lastGlobalRefresh;

        if (!force && state.globalForecast.length && age < 55 * 60 * 1000) {
            renderGlobalLayer();
            return;
        }

        if (state.globalAbortController) {
            state.globalAbortController.abort();
        }
        state.globalAbortController = new AbortController();

        elements.mapLoading.classList.add("show");
        elements.mapStatus.textContent = "Loading worldwide forecast grid…";

        state.globalPoints = buildGlobalPoints();
        const batchSize = 30;
        const batches = [];

        for (let start = 0; start < state.globalPoints.length; start += batchSize) {
            batches.push({
                start,
                points: state.globalPoints.slice(start, start + batchSize)
            });
        }

        try {
            const settled = await Promise.allSettled(
                batches.map(async batch => ({
                    start: batch.start,
                    data: await fetchGlobalForecastBatch(
                        batch.points,
                        state.globalAbortController.signal
                    )
                }))
            );

            const merged = Array(state.globalPoints.length).fill(null);
            let successfulBatches = 0;

            settled.forEach(result => {
                if (result.status !== "fulfilled") {
                    return;
                }

                successfulBatches += 1;
                result.value.data.forEach((forecast, offset) => {
                    merged[result.value.start + offset] = forecast;
                });
            });

            if (!successfulBatches) {
                throw new Error("All worldwide forecast batches failed");
            }

            state.globalForecast = merged;
            state.lastGlobalRefresh = Date.now();

            renderGlobalLayer();
            elements.mapStatus.textContent =
                successfulBatches === batches.length
                    ? "Worldwide forecast surface online"
                    : "Worldwide forecast partially available";
        } catch (error) {
            if (error.name === "AbortError") {
                return;
            }
            elements.mapStatus.textContent = "Global prediction layer unavailable";
            showToast(
                "The world weather surface could not load. The real map and local forecasts still work.",
                true
            );
            console.error(error);
        } finally {
            elements.mapLoading.classList.remove("show");
        }
    }

    function renderGlobalLayer() {
        if (!state.surfaceLayer || !state.globalForecast.length) {
            return;
        }

        const referenceForecast = state.globalForecast.find(Boolean);
        const hour = clamp(
            state.selectedForecastHour,
            0,
            Math.max(0, (referenceForecast?.hourly?.time?.length || 1) - 1)
        );

        state.surfaceLayer.setForecast(
            state.globalForecast,
            state.globalPoints,
            state.selectedLayer,
            hour
        );

        elements.mapModeTitle.textContent = LAYER_INFO[state.selectedLayer].title;
        updateForecastHourLabel();
    }

    function surfacePixel(layer, values) {
        if (!values) {
            return { r: 0, g: 0, b: 0, a: 0 };
        }

        if (layer === "heat") {
            if (!Number.isFinite(values.temperature)) {
                return { r: 0, g: 0, b: 0, a: 0 };
            }
            const color = colorToRGB(temperatureColor(values.temperature));
            return { ...color, a: 112 };
        }

        if (layer === "rain") {
            const probability = clamp((values.rainProbability || 0) / 100, 0, 1);
            const intensity = clamp((values.rain || 0) / 5, 0, 1);
            const strength = Math.max(probability * 0.78, intensity);

            if (strength < 0.035) {
                return { r: 0, g: 0, b: 0, a: 0 };
            }

            const color = colorToRGB(rainColor(strength * 100));
            return { ...color, a: Math.round(35 + Math.pow(strength, 0.68) * 190) };
        }

        if (layer === "snow") {
            const snowfall = Math.max(0, values.snowfall || 0);
            const strength = clamp(snowfall / 3, 0, 1);

            if (strength < 0.01) {
                return { r: 0, g: 0, b: 0, a: 0 };
            }

            const color = colorToRGB(snowColor(snowfall));
            return { ...color, a: Math.round(45 + Math.pow(strength, 0.58) * 205) };
        }

        const temperature = values.temperature;
        if (!Number.isFinite(temperature)) {
            return { r: 0, g: 0, b: 0, a: 0 };
        }
        const comfort = clamp(1 - Math.abs(temperature - 18) / 14, 0, 1);
        const dryScore = clamp(1 - (values.rainProbability || 0) / 85, 0, 1);
        const windScore = clamp(1 - Math.max(0, (values.wind || 0) - 10) / 45, 0, 1);
        const score = comfort * (0.55 + dryScore * 0.25 + windScore * 0.2);

        if (score < 0.12) {
            return { r: 0, g: 0, b: 0, a: 0 };
        }

        const color = colorToRGB(springColor(temperature));
        return { ...color, a: Math.round(30 + score * 175) };
    }

    function findGridPosition(grid, value) {
        if (value <= grid[0]) {
            return { lower: 0, upper: 0, ratio: 0 };
        }

        const last = grid.length - 1;
        if (value >= grid[last]) {
            return { lower: last, upper: last, ratio: 0 };
        }

        for (let index = 0; index < last; index += 1) {
            const start = grid[index];
            const end = grid[index + 1];

            if (value >= start && value <= end) {
                return {
                    lower: index,
                    upper: index + 1,
                    ratio: (value - start) / Math.max(0.0001, end - start)
                };
            }
        }

        return { lower: 0, upper: 0, ratio: 0 };
    }

    function interpolateWeatherValues(a, b, c, d, xRatio, yRatio) {
        const fields = ["temperature", "rainProbability", "rain", "snowfall", "wind"];
        const result = {};

        fields.forEach(field => {
            const top = interpolateNullable(a?.[field], b?.[field], xRatio);
            const bottom = interpolateNullable(c?.[field], d?.[field], xRatio);
            result[field] = interpolateNullable(top, bottom, yRatio);
        });

        return result;
    }

    function interpolateNullable(first, second, ratio) {
        const a = first === null || first === undefined ? Number.NaN : Number(first);
        const b = second === null || second === undefined ? Number.NaN : Number(second);
        const hasA = Number.isFinite(a);
        const hasB = Number.isFinite(b);

        if (hasA && hasB) {
            return a + (b - a) * ratio;
        }

        if (hasA) {
            return a;
        }

        if (hasB) {
            return b;
        }

        return null;
    }

    function numberOrNull(value) {
        if (value === null || value === undefined || value === "") {
            return null;
        }
        const number = Number(value);
        return Number.isFinite(number) ? number : null;
    }

    function normalizeLongitude(value) {
        let longitude = Number(value);
        while (longitude < -180) longitude += 360;
        while (longitude > 180) longitude -= 360;
        return longitude;
    }

    function colorToRGB(color) {
        if (color.startsWith("#")) {
            return hexToRGB(color);
        }

        const values = color.match(/[\d.]+/g)?.map(Number) || [0, 0, 0];
        return {
            r: Math.round(values[0] || 0),
            g: Math.round(values[1] || 0),
            b: Math.round(values[2] || 0)
        };
    }

    function setLegend() {
        const info = LAYER_INFO[state.selectedLayer];
        elements.mapLegend.innerHTML = info.legend.map(([color, label]) => `
            <span class="legend-item" style="background:${color}">${label}</span>
        `).join("");
        elements.mapModeTitle.textContent = info.title;
    }

    function updateForecastHourLabel() {
        const hour = state.selectedForecastHour;
        const forecastTime = state.globalForecast.find(Boolean)?.hourly?.time?.[hour];

        elements.forecastHourLabel.textContent =
            hour === 0
                ? "Now"
                : forecastTime
                    ? `${formatUTCHour(forecastTime)} (+${hour}h)`
                    : `+${hour} hours`;
    }

    function weatherInfo(code, isDay = 1) {
        const weather = WMO[code] || {
            label: "Weather data",
            day: "🌐",
            night: "🌐"
        };

        return {
            label: weather.label,
            icon: Number(isDay) === 0 ? weather.night : weather.day
        };
    }

    function temperatureColor(value) {
        const stops = [
            [-30, "#5538d9"],
            [-10, "#4a66ff"],
            [0, "#2d9cff"],
            [10, "#24c9ff"],
            [20, "#42e69a"],
            [30, "#ffd166"],
            [40, "#ff7b42"],
            [50, "#ff354d"]
        ];

        return interpolateStops(value, stops);
    }

    function rainColor(value) {
        const stops = [
            [0, "#071a40"],
            [20, "#1766d2"],
            [40, "#18a9e6"],
            [70, "#48e0ff"],
            [100, "#e4fbff"]
        ];

        return interpolateStops(value, stops);
    }

    function snowColor(value) {
        const stops = [
            [0, "#17324c"],
            [0.1, "#2e7ab6"],
            [0.5, "#72c9f2"],
            [1.5, "#c6f3ff"],
            [3, "#ffffff"]
        ];

        return interpolateStops(value, stops);
    }

    function springColor(temperature) {
        const stops = [
            [-10, "#203b45"],
            [8, "#4cb77c"],
            [17, "#7ee49b"],
            [24, "#d3ef72"],
            [35, "#e07a55"]
        ];

        return interpolateStops(temperature, stops);
    }

    function interpolateStops(value, stops) {
        const numeric = Number(value);

        if (!Number.isFinite(numeric)) {
            return stops[0][1];
        }

        if (numeric <= stops[0][0]) {
            return stops[0][1];
        }

        if (numeric >= stops[stops.length - 1][0]) {
            return stops[stops.length - 1][1];
        }

        for (let index = 0; index < stops.length - 1; index += 1) {
            const [startValue, startColor] = stops[index];
            const [endValue, endColor] = stops[index + 1];

            if (numeric >= startValue && numeric <= endValue) {
                const ratio = (numeric - startValue) / (endValue - startValue);
                return mixHexColors(startColor, endColor, ratio);
            }
        }

        return stops[0][1];
    }

    function mixHexColors(first, second, ratio) {
        const a = hexToRGB(first);
        const b = hexToRGB(second);

        const mixed = {
            r: Math.round(a.r + (b.r - a.r) * ratio),
            g: Math.round(a.g + (b.g - a.g) * ratio),
            b: Math.round(a.b + (b.b - a.b) * ratio)
        };

        return `rgb(${mixed.r}, ${mixed.g}, ${mixed.b})`;
    }

    function hexToRGB(hex) {
        const value = hex.replace("#", "");
        const parsed = Number.parseInt(value, 16);

        return {
            r: (parsed >> 16) & 255,
            g: (parsed >> 8) & 255,
            b: parsed & 255
        };
    }

    function displayLocationName(location) {
        return [
            location.name,
            location.district,
            location.admin1,
            location.country
        ]
            .filter(Boolean)
            .filter((item, index, array) => array.indexOf(item) === index)
            .slice(0, 4)
            .join(", ");
    }

    function formatTemperature(value) {
        return Number.isFinite(Number(value))
            ? `${Math.round(Number(value))}°`
            : "--°";
    }

    function formatNumber(value, suffix = "", decimals = 0) {
        return Number.isFinite(Number(value))
            ? `${Number(value).toFixed(decimals)}${suffix}`
            : `--${suffix}`;
    }

    function formatHour(value) {
        if (!value) {
            return "--";
        }

        return new Date(value).toLocaleTimeString([], {
            hour: "2-digit",
            minute: "2-digit",
            hour12: false
        });
    }

    function formatUTCHour(value) {
        if (!value) {
            return "--:-- UTC";
        }

        const time = value.includes("T") ? value.split("T")[1] : value;
        return `${time.slice(0, 5)} UTC`;
    }

    function formatLocalTime(value) {
        if (!value) {
            return "--";
        }

        return value.includes("T")
            ? value.split("T")[1].slice(0, 5)
            : value;
    }

    function formatCoordinate(value, positive, negative) {
        const direction = value >= 0 ? positive : negative;
        return `${Math.abs(value).toFixed(2)}°${direction}`;
    }

    function escapeHTML(value) {
        return String(value ?? "")
            .replaceAll("&", "&amp;")
            .replaceAll("<", "&lt;")
            .replaceAll(">", "&gt;")
            .replaceAll('"', "&quot;")
            .replaceAll("'", "&#039;");
    }

    function clamp(value, minimum, maximum) {
        return Math.min(maximum, Math.max(minimum, value));
    }

    function debounce(callback, delay) {
        let timer;

        return (...args) => {
            clearTimeout(timer);
            timer = setTimeout(() => callback(...args), delay);
        };
    }

    function showToast(message, isError = false) {
        clearTimeout(state.toastTimer);
        elements.toast.textContent = message;
        elements.toast.classList.toggle("error", isError);
        elements.toast.classList.add("show");

        state.toastTimer = setTimeout(() => {
            elements.toast.classList.remove("show");
        }, 3500);
    }

    if (document.readyState === "loading") {
        document.addEventListener("DOMContentLoaded", initialize, { once: true });
    } else {
        initialize();
    }
})();
