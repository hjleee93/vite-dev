<template>
    <div id="map"></div>

    <div class="map-overlay top">
        <div class="map-overlay-inner">
            <label>x point : {{ xPoint }}</label>
            <label>y point : {{ yPoint }}</label>
            <label>lat : {{ lat }}</label>
            <label>lng : {{ lng }}</label>
        </div>
    </div>
</template>

<script>
import { ref, onMounted } from "vue";
import mapboxgl from "mapbox-gl";
import "mapbox-gl/dist/mapbox-gl.css";
import testJson from "../data/test.json";
import L from "leaflet";
import axios from "axios";
import Maplibre from "maplibre-gl";
import * as MaplibreGrid from "maplibre-grid";
import * as turf from "@turf/turf";
import _ from "lodash";
import tilebelt from "@mapbox/tilebelt";

import SphericalMercator from "../scripts/sphericalmercator";

export default {
    setup() {
        let xPoint = ref(0);
        let yPoint = ref(0);

        let lat = ref(0);
        let lng = ref(0);

        function fCode() {
            axios
                .get(
                    "https://api.mapbox.com/geocoding/v5/mapbox.places/-73.989,40.733.json?access_token=pk.eyJ1IjoiaGpsZWVlOTMiLCJhIjoiY2t6Nno1Zm1zMHc2eDJ5cHI4cmp6NWk1OCJ9.Ai1Gmiu--ecVJ6DEf73nMQ"
                )
                .then((res) => {
                    console.log("res", res);
                });
        }

        function setGeometryJson(coordinates) {
            return {
                type: "FeatureCollection",
                features: [
                    {
                        type: "Feature",
                        geometry: {
                            type: "MultiPolygon",
                            coordinates: coordinates,
                        },
                    },
                ],
            };
        }
        function drawGrid() {
            mapboxgl.accessToken =
                "pk.eyJ1IjoiaGpsZWVlOTMiLCJhIjoiY2t6Nno1Zm1zMHc2eDJ5cHI4cmp6NWk1OCJ9.Ai1Gmiu--ecVJ6DEf73nMQ";

            const map = new mapboxgl.Map({
                container: "map", // container ID
                style: "mapbox://styles/mapbox/streets-v11", // style URL
                center: [-98.23942393345851, 29.844585070439663], // starting position [lng, lat]
                zoom: 18, // starting zoom
            });

            console.log("map", map);

            const zoom = map.getZoom();
            const loadFeatures = zoom > 17;

            console.log(loadFeatures);

            if (loadFeatures) {
                // Zoom level is high enough
                var ne = map.getBounds().getNorthEast();
                var sw = map.getBounds().getSouthWest();

                console.log(what3words);

                // Call the what3words Grid API to obtain the grid squares within the current visble bounding box
                what3words.api
                    .gridSectionGeoJson({
                        southwest: {
                            lat: sw.lat,
                            lng: sw.lng,
                        },
                        northeast: {
                            lat: ne.lat,
                            lng: ne.lng,
                        },
                    })
                    .then(function (data) {
                        console.log("data", data);
                        // Get the grid source from the map (it won't exist initally)
                        var grid = map.getSource("grid");

                        if (typeof grid === "undefined") {
                            // Create a source of type 'geojson' which loads the GeoJSON returned from the what3words API
                            map.addSource("grid", {
                                type: "geojson",
                                data: data,
                            });

                            // Create a new layer, which loads data from the newly created data source
                            map.addLayer({
                                id: "grid_layer",
                                type: "line",
                                source: "grid",
                                layout: {
                                    "line-join": "round",
                                    "line-cap": "round",
                                },
                                paint: {
                                    "line-color": "#777",
                                    "line-width": 0.5,
                                },
                            });
                        } else {
                            // The source and map layer already exist, so just update the source data to be the new
                            // GeoJSON returned from the what3words API
                            map.getSource("grid").setData(data);
                        }
                    })
                    .catch(console.error);

                //    map.on("load", drawGrid).on("move", drawGrid);
            }

            // If we have reached the required zoom level, set the 'grid_layer' to be visible
            var grid_layer = map.getLayer("grid_layer");
            if (typeof grid_layer !== "undefined") {
                map.setLayoutProperty(
                    "grid_layer",
                    "visibility",
                    loadFeatures ? "visible" : "none"
                );
            }
        }

        function turfGrid() {
            const map = new mapboxgl.Map({
                container: "map", // container ID
                style: "mapbox://styles/mapbox/streets-v11", // style URL
                center: [-98.5, 35], // starting position [lng, lat]
                zoom: 20, // starting zoom
            });
            map.boxZoom.disable();
            const bounds = map.getBounds();

            const NE = bounds.getNorthEast();
            const SW = bounds.getSouthWest();
            var bbox = [SW.lng, SW.lat, NE.lng, NE.lat];

            console.log(bbox);
            var cellSide = 0.01;

            var squareGrid = turf.squareGrid(bbox, cellSide);

            return squareGrid;
        }

        // When the map is either loaded or moved, check to see if the grid should be draw
        // if the appropriate zoom level has been met, and if so draw it on.

        function getTileScreenCoords(x, y, zoom) {
            // Given tile coordinates, returns x and y pixel positions of the upper-left corner of the tile.
            let { lng, lat } = XYToLngLat(x, y, zoom);
            let screenCoords = map.project([lng, lat]);
            const pixelsX = Math.round(screenCoords.x * 100) / 100;
            const pixelsY = Math.round(screenCoords.y * 100) / 100;
            return { pixelsX, pixelsY };
        }
        function XYToLngLat(x, y, zoom) {
            // Converts x and y tile coordinates to lng and lat coordinates. This returns the NW-corner of the tile.
            let lng = (x / Math.pow(2, zoom)) * 360 - 180;
            let n = Math.PI - (2 * Math.PI * y) / Math.pow(2, zoom);
            let lat = -1 * (radiansToDegrees(Math.atan(Math.sinh(n))) * -1);
            return { lng, lat };
        }
        function radiansToDegrees(radiansVal) {
            return radiansVal * (180 / Math.PI);
        }
        
        onMounted(() => {
            let sphericalmercator = new SphericalMercator();

            var tile = [10, 15, 8]; // x,y,z

            console.log("tilebelt", tilebelt.bboxToTile(tile));

            mapboxgl.accessToken =
                "pk.eyJ1IjoiaGpsZWVlOTMiLCJhIjoiY2t6Nno1Zm1zMHc2eDJ5cHI4cmp6NWk1OCJ9.Ai1Gmiu--ecVJ6DEf73nMQ";
            // const map = new mapboxgl.Map({
            //     container: "map", // container ID
            //     style: "mapbox://styles/mapbox/streets-v11", // style URL
            //     center: [-91.73155103968728, 30.830091716712417], // starting position [lng, lat]
            //     zoom: 9, // starting zoom
            // });

            // drawGrid();

            // const grid = new MaplibreGrid.Grid({
            //     gridWidth: 10,
            //     gridHeight: 10,
            //     units: "degrees",
            //     paint: {
            //         "line-opacity": 0.2,
            //     },
            // });
            // map.addControl(grid);

            // Disable default box zooming.

            const popup = new mapboxgl.Popup({
                closeButton: false,
            });

            let onScreenJson = {
                quadKeys: [],
                coordinates: [],
            };

            //지도 시작 포인트
            const startPos = [126.9548, 37.55];

            const map = new mapboxgl.Map({
                container: "map", // container ID
                style: "mapbox://styles/mapbox/satellite-v9", // style URL
                center: startPos, // starting position [lng, lat]
                zoom: 20, // starting zoom
            });

            const bounds = map.getBounds();
            const ne = bounds.getNorthEast();
            const sw = bounds.getSouthWest();

            // xyz
            const xyz = sphericalmercator.xyz(startPos, 21);

            const box = sphericalmercator.bbox(xyz.minX, xyz.maxY, 21);

            const offset = [box[2] - box[0], box[3] - box[1]];

            for (
                let i = sw.lng - offset[0];
                i <= ne.lng + offset[0];
                i += offset[0]
            ) {
                // 0: longitude, 1: latitude
                for (
                    let j = sw.lat - offset[1];
                    j <= ne.lat + offset[1];
                    j += offset[1]
                ) {
                    const _tile = tilebelt.pointToTile(i, j, 21);
                    const _geoJson = tilebelt.tileToGeoJSON(_tile).coordinates;
                    onScreenJson.coordinates.push(_geoJson);
                    onScreenJson.quadKeys.push(tilebelt.tileToQuadkey(_tile));
                }
            }
            const geometryJson = setGeometryJson(onScreenJson.coordinates);

            console.log("offset", geometryJson);

            // 줌 크기별로 전체 영역 잡기
            var poly = [turf.bboxPolygon(box)];

            console.log("poly", poly);
            console.log("poly.box", poly[0].bbox);
            console.log("box", box);

            //              Double = 0.00008982987
            // public static let MAPCOORD_UNIT_LENGTH_LONGITUDE: Double = 0.00010729588

            const polyBbox = poly[0].bbox;
            console.log("polyBbox", polyBbox);
            console.log("polyBbox", turf.bboxPolygon(polyBbox));
            // poly = poly.con + turf.bboxPolygon(polyBbox)

            console.log(poly.push(turf.bboxPolygon(polyBbox)));

            var options = { units: "kilometers" };

            // var bbox = [SW.lng, SW.lat, NE.lng, NE.lat];

            // console.log("bbox2", bbox);

            // var cellSide = 0.01;

            // poly = turf.squareGrid(poly.bbox, cellSide, options);

            // console.log("squareGrid", squareGrid);
            map.boxZoom.disable();
            map.on("load", () => {
                // 영역 드래그

                //              map.addLayer({
                //     id: 'rpd_parks',
                //     type: 'fill',
                //     source: {
                //         type: 'vector',
                //         url: 'mapbox://mapbox.3o7ubwm8'
                //     },
                //     'source-layer': 'RPD_Parks',
                //     layout: {
                //         visibility: 'visible'
                //     },
                //     paint: {
                //         'fill-color': 'rgba(61,153,80,0.55)'
                //     }
                // });
                const canvas = map.getCanvasContainer();

                let start;

                // Variable to hold the current xy coordinates
                // when `mousemove` or `mouseup` occurs.
                let current;

                // Variable for the draw box element.
                let box;

                // const graticule = {
                //     type: "FeatureCollection",
                //     features: [],
                // };
                // for (let lng = 50; lng < 500; lng += 0.00008982987) {
                //     graticule.features.push({
                //         type: "Feature",
                //         geometry: {
                //             type: "LineString",
                //             coordinates: [
                //                 [lng, -90],
                //                 [lng, 90],
                //             ],
                //         },
                //         properties: { value: lng, id : lng },
                //     });
                // }
                // for (let lat = -80; lat <= 80; lat += 10) {
                //     graticule.features.push({
                //         type: "Feature",
                //         geometry: {
                //             type: "LineString",
                //             coordinates: [
                //                 [-180, lat],
                //                 [180, lat],
                //             ],
                //         },
                //         properties: { value: lat,id : lat  },
                //     });
                // }

                // map.addSource("maine", {
                //     type: "geojson",
                //     data: graticule,
                // });
                // map.addLayer({
                //     id: "maine-layer",
                //     type: "line",
                //     source: "maine",
                // });
                // grid 그리기
                map.addSource("maine", {
                    type: "geojson",
                    data: geometryJson,
                    // generateId: true,
                    // data: {
                    //     type: "FeatureCollection",
                    //     features: poly,
                    // },
                });

                map.addLayer({
                    id: "maine-layer",
                    type: "fill",
                    source: "maine", // reference the data source
                    layout: {},
                    paint: {
                        "fill-color": "#0080ff", // blue color fill
                        "fill-opacity": 0.5,
                    },
                });

                map.addLayer({
                    id: "maine-highlighted",
                    type: "fill",
                    source: "maine",
                    paint: {
                        "fill-outline-color": "#484896",
                        "fill-color": "#6e599f",
                        "fill-opacity": 0.75,
                    },
                    filter: ["in", "id", ""],
                }); // Place polygon under these labels.

                canvas.addEventListener("mousedown", mouseDown, true);

                function mousePos(e) {
                    const rect = canvas.getBoundingClientRect();

                    console.log("rect", rect);
                    return new mapboxgl.Point(
                        e.clientX - rect.left - canvas.clientLeft,
                        e.clientY - rect.top - canvas.clientTop
                    );
                }

                function mouseDown(e) {
                    console.log(e);
                    // Continue the rest of the function if the shiftkey is pressed.
                    if (!(e.shiftKey && e.button === 0)) return;

                    // Disable default drag zooming when the shift key is held down.
                    map.dragPan.disable();

                    // Call functions for the following events
                    document.addEventListener("mousemove", onMouseMove);
                    document.addEventListener("mouseup", onMouseUp);
                    document.addEventListener("keydown", onKeyDown);

                    // Capture the first xy coordinates
                    start = mousePos(e);
                }

                function onMouseMove(e) {
                    // Capture the ongoing xy coordinates
                    current = mousePos(e);

                    // Append the box element if it doesnt exist
                    if (!box) {
                        box = document.createElement("div");
                        box.classList.add("boxdraw");
                        canvas.appendChild(box);
                    }

                    const minX = Math.min(start.x, current.x),
                        maxX = Math.max(start.x, current.x),
                        minY = Math.min(start.y, current.y),
                        maxY = Math.max(start.y, current.y);

                    // Adjust width and xy position of the box element ongoing
                    const pos = `translate(${minX}px, ${minY}px)`;
                    box.style.transform = pos;
                    box.style.width = maxX - minX + "px";
                    box.style.height = maxY - minY + "px";
                }

                function onMouseUp(e) {
                    // Capture xy coordinates
                    finish([start, mousePos(e)]);
                }

                function onKeyDown(e) {
                    // If the ESC key is pressed
                    if (e.keyCode === 27) finish();
                }

                function finish(bbox) {
                    // Remove these events now that finish has been called.
                    document.removeEventListener("mousemove", onMouseMove);
                    document.removeEventListener("keydown", onKeyDown);
                    document.removeEventListener("mouseup", onMouseUp);

                    if (box) {
                        box.parentNode.removeChild(box);
                        box = null;
                    }

                    // If bbox exists. use this value as the argument for `queryRenderedFeatures`
                    if (bbox) {
                        const features = map.queryRenderedFeatures(bbox, {
                            layers: ["maine-layer"],
                        });

                        if (features.length >= 1000) {
                            return window.alert(
                                "Select a smaller number of features"
                            );
                        }

                        // Run through the selected features and set a filter
                        // to match features with unique FIPS codes to activate
                        // the `counties-highlighted` layer.
                        const id = features.map(
                            (feature) => feature.properties.id
                        );
                        map.setFilter("maine-highlighted", ["in", "id", ...id]);
                    }

                    map.dragPan.enable();
                }

                map.on("mousemove", (e) => {
                    const features = map.queryRenderedFeatures(e.point, {
                        layers: ["maine-highlighted"],
                    });

                    // Change the cursor style as a UI indicator.
                    map.getCanvas().style.cursor = features.length
                        ? "pointer"
                        : "";

                    if (!features.length) {
                        popup.remove();
                        return;
                    }

                    // popup
                    //     .setLngLat(e.lngLat)
                    //     .setText(features[0].properties.COUNTY)
                    //     .addTo(map);
                });

                // map.on("zoom", (e) => {
                //     console.log("A zoom event occurred.", e);
                // });

                map.on("click", (e) => {
                    console.log("e", e);
                    xPoint.value = e.point.x;
                    yPoint.value = e.point.y;

                    lat.value = e.lngLat.lat;
                    lng.value = e.lngLat.lng;
                    // Set `bbox` as 5px reactangle area around clicked point.
                    const bbox = [
                        [e.point.x - 5, e.point.y - 5],
                        [e.point.x + 5, e.point.y + 5],
                    ];
                    // Find features intersecting the bounding box.
                    const selectedFeatures = map.queryRenderedFeatures(bbox, {
                        layers: ["maine-layer"],
                    });
                    const id = selectedFeatures.map((feature) =>
                        feature.id.toString()
                    );

                    map.setFilter("maine-highlighted", ["in", "id", ...id]);
                });
            });
        });

        return {
            xPoint,
            yPoint,
            lat,
            lng,
        };
    },
};
</script>


<style>
body {
    margin: 0;
    padding: 0;
}
#map {
    position: absolute;
    top: 0;
    bottom: 0;
    width: 100%;
}
.boxdraw {
    background: rgba(56, 135, 190, 0.1);
    border: 2px solid #3887be;
    position: absolute;
    top: 0;
    left: 0;
    width: 0;
    height: 0;
}

.map-overlay {
    font: bold 12px/20px "Helvetica Neue", Arial, Helvetica, sans-serif;
    position: absolute;
    width: 25%;
    top: 0;
    left: 0;
    padding: 10px;
}

.map-overlay .map-overlay-inner {
    background-color: #fff;
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
    border-radius: 3px;
    padding: 10px;
    margin-bottom: 10px;
}

.map-overlay label {
    display: block;
    margin: 0 0 10px;
}

.map-overlay input {
    background-color: transparent;
    display: inline-block;
    width: 100%;
    position: relative;
    margin: 0;
    cursor: ew-resize;
}
</style>
