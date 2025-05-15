<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import * as d3 from "d3";

    mapboxgl.accessToken = "pk.eyJ1IjoicGVkcm9waGMyNSIsImEiOiJjbWFwaDFvdngwaHBnMmxwb3E2MjlhMGhhIn0.TTDmNO_-H8EeIdYnMCO4LA";

    import { onMount } from "svelte";

    let map;
    let stations = [];
    let mapViewChanged = 0;
    let trips = [];
    let departures;
    let arrivals;
    let radiusScale;

    $: radiusScale = d3.scaleSqrt()
        .domain([0, d3.max(stations, d => d.totalTraffic) || 0])
        .range([0, 25]);
    
    $: map?.on("move", evt => mapViewChanged++);

    function getCoords(station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let { x, y } = map.project(point);
        return { cx: x, cy: y };
    }

    async function loadStationData() {
        try {
            const csvUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-stations.csv';
            const data = await d3.csv(csvUrl);
            
            stations = data.map(station => ({
                id: station.Number,
                name: station.NAME,
                Lat: +station.Lat,
                Long: +station.Long,
            }));
            // console.log('Stations loaded:', stations.length);
        } catch (error) {
            console.error('Error loading station data:', error);
        }
    }

    async function loadStationDemand() {
        try {
            const csvUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv';
            const data = await d3.csv(csvUrl);
            
            trips = data.map(trip => {
                return {
                    id: trip.ride_id,
                    name: trip.NAME,
                    started_at: new Date(trip.started_at),
                    ended_at: new Date(trip.ended_at),
                    start_station_id: trip.start_station_id,
                    end_station_id: trip.end_station_id
                };
            });
            
            // console.log('Trips loaded:', trips.length);
        } catch (error) {
            console.error('Error loading station data:', error);
        }
    }

    async function initMap() {
        map = new mapboxgl.Map({
            container: 'map',
            center: [-71.09415, 42.36027],
            zoom: 12,
            style: "mapbox://styles/pedrophc25/cmaphl15u00cs01r2bqsl8xu1",
        });
        await new Promise(resolve => map.on("load", resolve));
        map.addSource("boston_route", {
            type: "geojson",
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
        });
        map.addSource("cambridge-route", {
            type: "geojson",
            data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
        });
        map.addLayer({
            id: "BOSTON", // A name for our layer (up to you)
            type: "line", // one of the supported layer types, e.g. line, circle, etc.
            source: "boston_route", // The id we specified in `addSource()`
            paint: {
                "line-color": "green",
                "line-width": 3,
                "line-opacity": 0.4
            },
        });
        map.addLayer({
            id: "CAMBRIDGE", // A name for our layer (up to you)
            type: "line", // one of the supported layer types, e.g. line, circle, etc.
            source: "cambridge-route", // The id we specified in `addSource()`
            paint: {
                "line-color": "orange",
                "line-width": 3,
                "line-opacity": 0.4
            },
        });
    }

    // Executa ao montar o componente
    onMount(async () => {
        await initMap();
        await loadStationData();
        await loadStationDemand();

        const departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
        const arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);

        stations = stations.map(station => {
            const id = station.id;
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;
            return station;
        });
    });
</script>


<h1>Bikes</h1>

<div id="map">
	<svg>
        {#key mapViewChanged}
            {#each stations as station}
                <circle
                    {...getCoords(station)}
                    r={radiusScale(station.totalTraffic)}
                    fill="steelblue"
                    fill-opacity="0.7"
                    stroke="white"
                    stroke-width="1"
                />
            {/each}
        {/key}
    </svg>
</div>


<style>
    @import url("$lib/global.css");

    #map {
        flex: 1;
    }
    #map svg {
        /* background: yellow;
        opacity: 50%; */
        z-index: 1;
        position: absolute;
        width: 100%;
        height: 100%;
        pointer-events: none;
    }
</style>