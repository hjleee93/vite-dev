<template>
    <div id="map"></div>
    <MapInfo @onSelectedReset="onSelectedReset"></MapInfo>
</template>

<script>
import { ref, onMounted } from "vue";
import mapboxgl from "mapbox-gl";
import "mapbox-gl/dist/mapbox-gl.css";
import * as turf from "@turf/turf";
import SphericalMercator from "../scripts/sphericalmercator";
import tilebelt from "@mapbox/tilebelt";
import MapInfo from "./MapInfo.vue";

export default {
    components: {
        MapInfo,
    },
    
    setup() {
        let xPoint = ref(0);
        let yPoint = ref(0);

        let lat = ref(0);
        let lng = ref(0);

    
        function calMinMax(start, end) {
            const minX = Math.min(start.tile[0], end.tile[0]);
            const minY = Math.min(start.tile[1], end.tile[1]);
            const maxX = Math.max(start.tile[0], end.tile[0]);
            const maxY = Math.max(start.tile[1], end.tile[1]);

            const returnJson = {
                x: {
                    min: minX,
                    max: maxX,
                },
                y: {
                    min: minY,
                    max: maxY,
                },
            };
            return returnJson;
        }

        const setClickTileGroupStart = async (onGroupSelect) => {
            const clickAreaTileFeatures = setGeometryJson(
                onGroupSelect.coordinates
            );
            const clickAreaFlagFeatures = setGeometryJsonMultiPoint(
                onGroupSelect.center
            );

            // 구매된 tile 레이어 올리기
            if (map.getSource("buyedData") === undefined) {
                map.addSource("buyedData", {
                    type: "geojson",
                    data: clickAreaTileFeatures,
                });

                map.addLayer({
                    id: "buyed_layer",
                    type: "fill",
                    source: "buyedData",
                    paint: {
                        "fill-color": "#79da19",
                        "fill-opacity": 0.5,
                    },
                    minzoom: 17,
                    layout: {
                        visibility: "visible",
                    },
                });
            } else {
                map.getSource("buyedData").setData(clickAreaTileFeatures);

                //국기의 경우 이전 깃발사진으로 된 레이어가 남아있기 때문에 지우고 다시 만들어야 한다. 다시 만드는건 아랫줄에...
                map.removeLayer("buyedFlag_layer");
                map.removeSource("buyedFlagData");
            }

            // 국기 레이어 생성 및 재생성
            map.addSource("buyedFlagData", {
                type: "geojson",
                data: clickAreaFlagFeatures,
            });

            map.addLayer({
                id: "buyedFlag_layer",
                type: "symbol",
                source: "buyedFlagData",
                minzoom: 17,
                layout: {
                    "icon-image": onGroupSelect.country,
                    "icon-size": 1.3,
                },
                paint: {
                    "icon-opacity": 1,
                },
            });

            setSelectedCellList((prevState) => ({
                ...prevState,
                groupSelect: {
                    quadKeys: onGroupSelect.quadKeys,
                    coordinates: onGroupSelect.coordinates,
                    isGroupSelect: true,
                },
            }));

            onSelect.onGroupSelect = onGroupSelect;
            selectTile.groupClick = true;
        };
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

        let onSelect = {
            mode: "select",
            selected: {
                // 화면상에 선택
                quadKeys: [],
                coordinates: [],
            },
            onSelect: {
                // 선택중인 셀
                start: {},
                end: {},
                quadKeys: [],
                coordinates: [],
            },
            onDelete: {
                // 삭제중인 셀
                start: {},
                end: {},
                quadKeys: [],
                coordinates: [],
            },
            onGroupSelect: {
                // 그룹 셀렉트
                id: -1,
                quadKeys: [],
                coordinates: [],
            },
        };

        const onScreen = {
            quadKeys: [],
            coordinates: [],
            onBuyed: [],
            countryList: [],
        };

        const selectTile = {
            tileClick: false,
            groupClick: false,
        };

        const selectedCellList = {
            boxSelect: {
                isBoxSelect: false,
                quadKeys: [],
                coordinates: [],
            },
            groupSelect: {
                isGroupSelect: false,
                groupId: -1,
                quadKeys: [],
                coordinates: [],
            },
        };

        let setSelectedCellList = {
            boxSelect: {
                isBoxSelect: false,
                quadKeys: [],
                coordinates: [],
            },
            groupSelect: {
                isGroupSelect: false,
                groupId: -1,
                quadKeys: [],
                coordinates: [],
            },
        };

        const onSelectedReset = () => {
            console.log("?");
            onSelect = {
                mode: "select",
                selected: {
                    // 화면상에 선택
                    quadKeys: [],
                    coordinates: [],
                },
                onSelect: {
                    // 선택중인 셀
                    start: {},
                    end: {},
                    quadKeys: [],
                    coordinates: [],
                },
                onDelete: {
                    // 삭제중인 셀
                    start: {},
                    end: {},
                    quadKeys: [],
                    coordinates: [],
                },
                onGroupSelect: {
                    // 그룹 셀렉트
                    id: -1, // 그룹 아이디
                    quadKeys: [],
                    coordinates: [],
                },
            };
            selectTile.tileClick = false;
            selectTile.groupClick = false;
        };

        onMounted(() => {
            let tileStart = false;
            let sphericalmercator = new SphericalMercator();
            //지도 시작 포인트
            const startPos = [126.9548, 37.55];

            let onScreenJson = {
                quadKeys: [],
                coordinates: [],
            };
            mapboxgl.accessToken =
                "pk.eyJ1IjoiaGpsZWVlOTMiLCJhIjoiY2t6Nno1Zm1zMHc2eDJ5cHI4cmp6NWk1OCJ9.Ai1Gmiu--ecVJ6DEf73nMQ";

            const map = new mapboxgl.Map({
                container: "map", // container ID
                style: "mapbox://styles/mapbox/streets-v11", // style URL
                center: startPos, // starting position [lng, lat]
                zoom: 18, // starting zoom
            });

            const bounds = map.getBounds();
            const ne = bounds.getNorthEast();
            const sw = bounds.getSouthWest();

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

            map.on("click", async (event) => {
                const _self = {};
                // if (map.getZoom() < 17) return; // 줌 레벨 17이하 이벤트막기
                console.log(event);

                const latLng = event.lngLat;
                const lng = latLng.lng,
                    lat = latLng.lat;

                //selected tile
                const tile = tilebelt.pointToTile(lng, lat, 21);

                //quadkey : 지도 index key
                const quadKey = tilebelt.tileToQuadkey(tile);
                const coordinate = tilebelt.tileToGeoJSON(tile).coordinates;

                /**
                 * [그룹 타일 클릭]
                 * desc: 구매된 그룹 클릭시 작동
                 * */
                _self.setClickTileGroupStart = async () => {
                    const findIndex = onScreen.onBuyed.findIndex(
                        (item) => item.quadKey === quadKey
                    );

                    // 선택 레이어 삭제 == 모두 초기화후 시작
                    if (map.getSource("selectData") !== undefined) {
                        onSelectedReset();
                        selectTile.tileClick = false;
                    }

                    // TODO : 나중에 조회 api로 변경 == START
                    let clickAreaJson = await callBuyedMapClick(
                        onScreen.onBuyed[findIndex].id
                    );
                    // TODO : 나중에 조회 api로 변경 == END

                    let onGroupSelect = {
                        id: onScreen.onBuyed[findIndex].id,
                        quadKeys: [],
                        coordinates: [],
                        center: [],
                        country: "kr",
                    };

                    clickAreaJson.map((item) => {
                        onGroupSelect.coordinates.push(item.coordinates);
                        onGroupSelect.quadKeys.push(item.quadKey);
                        onGroupSelect.center.push(item.center);
                        onGroupSelect.country = item.country;
                    });

                    await setClickTileGroupStart(onGroupSelect);
                };

                /**
                 * [영역선택 - 시작]
                 * 선택모드 => 클릭시
                 * */
                setClickTileSelectBoxStart = (quadKey, coordinate, tile) => {
                    const geometryJson = setGeometryJson([coordinate]);
                    console.log("geometryJson", geometryJson);
                    console.log("onSelect", onSelect);
                    onSelect.onSelect.start = { tile: tile }; // 최초 클릭시 타일 저장

                    // 선택모드 => 클릭시 => 새로운 레이어생성
                    if (map.getSource("selectData") === undefined) {
                        console.log("new layer", map.getSource("selectData"));

                        map.addSource("selectData", {
                            type: "geojson",
                            data: geometryJson,
                        });

                        map.addLayer({
                            id: "select_layer",
                            type: "fill",
                            source: "selectData",
                            paint: {
                                "fill-color": "#000",
                                "fill-opacity": 0.5,
                            },
                            minzoom: 18,
                        });
                        onSelect.selected.quadKeys.push(quadKey); // id값
                        onSelect.selected.coordinates.push(coordinate); // geojson

                        map.getSource("selectData").setData(geometryJson);
                    } else {
                        /**
                         * 선택모드 => 클릭시 => 소스 있을때
                         * 소스가 있다 === 아직 레이어 선택중 임을 인지
                         * map.getSource("selectData") !== undefined
                         * */
                        // 100개 넘어가면 true
                        const isMax = onSelect.selected.quadKeys.length >= 100;
                        // 기존 맵에 없는지 체크
                        const isExist =
                            onSelect.selected.quadKeys.findIndex(
                                (item) => item === quadKey
                            ) !== -1;
                        // 이미 구입된 맵에 있는지 체크
                        const isBuyedTile =
                            onScreen.onBuyed.findIndex(
                                (item) => item.quadKey === quadKey
                            ) !== -1;

                        if (!isMax && !isExist && !isBuyedTile) {
                            onSelect.selected.quadKeys.push(quadKey); // id값
                            onSelect.selected.coordinates.push(coordinate); // geojson
                            const geometryJsonPlus = setGeometryJson(
                                onSelect.selected.coordinates
                            );
                            map.getSource("selectData").setData(
                                geometryJsonPlus
                            );
                        } //!isMax && isExist
                    } // map.getSource("selectData") === undefined
                };

                /**
                 * [영역선택 - 종료]
                 * 선택모드 => 클릭종료시
                 * desc: 선택정보를 선택에 저장후, 초기화 한다.
                 * */
                function setClickTileSelectBoxEnd() {
                    console.log("end");
                    onSelect.selected.quadKeys = onSelect.onSelect.quadKeys;
                    onSelect.selected.coordinates =
                        onSelect.onSelect.coordinates;

                    // 블록정보 창을 위한 렌더링 정보
                    setSelectedCellList = {
                        groupSelect: {
                            isGroupSelect: false,
                            groupId: -1,
                            quadKeys: [],
                            coordinates: [],
                        },
                        boxSelect: {
                            quadKeys: onSelect.selected.quadKeys,
                            coordinates: onSelect.selected.coordinates,
                            isBoxSelect: true,
                        },
                    };

                    // 초기화
                    onSelect.onSelect.quadKeys = [];
                    onSelect.onSelect.coordinates = [];
                    onSelect.onSelect.start = {};
                    onSelect.onSelect.end = {};
                }

                /**
                 * [영역삭제 - 시작]
                 * 삭제모드 => 클릭시
                 * */
                _self.setClickTileDeleteBoxStart = (
                    quadKey,
                    coordinate,
                    tile
                ) => {
                    const geometryJson = setGeometryJson([coordinate]);

                    onSelect.onDelete.start = { tile: tile };

                    map.addSource("deleteData", {
                        type: "geojson",
                        data: geometryJson,
                    });

                    map.addLayer({
                        id: "delete_layer",
                        type: "fill",
                        source: "deleteData",
                        paint: {
                            "fill-outline-color": "#9e5296",
                            "fill-color": "#ffb5cc",
                            "fill-opacity": 0.5,
                        },
                    });

                    onSelect.onDelete.quadKeys.push(quadKey);
                    onSelect.onDelete.coordinates.push(coordinate);

                    map.getSource("deleteData").setData(geometryJson);
                };

                /**
                 * [영역삭제 - 종료]
                 * 삭제모드 => 클릭끝날때
                 * */
                _self.setClickTileDeleteBoxEnd = () => {
                    let deleteIndex = [];

                    //삭제할 index 추출
                    onSelect.selected.quadKeys.map((item, i) =>
                        onSelect.onDelete.quadKeys.includes(item) === true
                            ? deleteIndex.push(i)
                            : ""
                    );

                    //index 번호로 포함되지 않는 것만 뺀다. 삭제되고 나머지만 추출한다.
                    let onDeletedQuadKeys = onSelect.selected.quadKeys.filter(
                        (item, idx) => !deleteIndex.includes(idx)
                    );
                    let onDeletedCoordinates =
                        onSelect.selected.coordinates.filter(
                            (item, idx) => !deleteIndex.includes(idx)
                        );

                    onSelect.selected.quadKeys = onDeletedQuadKeys;
                    onSelect.selected.coordinates = onDeletedCoordinates;

                    // 구매창 상태변화 용
                    setSelectedCellList((prevState) => ({
                        ...prevState,
                        boxSelect: {
                            quadKeys: onSelect.selected.quadKeys,
                            coordinates: onSelect.selected.coordinates,
                            isBoxSelect:
                                onSelect.selected.quadKeys.length !== 0
                                    ? true
                                    : false,
                        },
                    }));

                    const geometryJson = setGeometryJson(
                        onSelect.selected.coordinates
                    );

                    if (map.getSource("selectData") !== undefined)
                        map.getSource("selectData").setData(geometryJson);

                    onSelect.onDelete.quadKeys = [];
                    onSelect.onDelete.coordinates = [];
                    onSelect.onDelete.start = {};
                    onSelect.onDelete.end = {};

                    map.removeLayer("delete_layer");
                    map.removeSource("deleteData");
                };

                /**
                 * [클릭동작]
                 * */
                async function setClickTile(quadKey, coordinate, tile) {
                    console.log("onSelect", onSelect);
                    switch (onSelect.mode) {
                        case "select" /** =================[선택모드]================= **/:
                            console.log("onScreen", onScreen);
                            const isBuyedTile =
                                onScreen.onBuyed.findIndex(
                                    (item) => item.quadKey === quadKey
                                ) !== -1;

                            if (onScreen.onBuyed.length !== 0 && isBuyedTile) {
                                _self.setClickTileGroupStart();
                                return; // ===================구입된 그룹클릭이면 여기서 멈춤===================
                            } //=============================================[그룹셀 클릭 종료]==========================================

                            // 빈 타일 클릭인 경우 => 레이어 초기화하기
                            if (map.getSource("buyedData") !== undefined) {
                                map.removeLayer("buyed_layer");
                                map.removeSource("buyedData");
                            }
                            if (map.getSource("buyedFlagData") !== undefined) {
                                map.removeLayer("buyedFlag_layer");
                                map.removeSource("buyedFlagData");
                            }

                            selectTile.groupClick = false;

                            /**
                             * 선택모드 => 클릭시
                             * desc: 선택 레이어의 셀 Tile을 만든다.
                             * */
                            if (!selectTile.tileClick) {
                                setSelectedCellList = {
                                    boxSelect: {
                                        isBoxSelect: false,
                                        quadKeys: [],
                                        coordinates: [],
                                    },
                                    groupSelect: {
                                        isGroupSelect: false,
                                        groupId: -1,
                                        quadKeys: [],
                                        coordinates: [],
                                    },
                                };

                                console.log("coordinate222", coordinate);
                                setClickTileSelectBoxStart(
                                    quadKey,
                                    coordinate,
                                    tile
                                );
                            } else {
                                setClickTileSelectBoxEnd();
                            }
                            break;

                        case "delete" /** =================[삭제모드]================= **/:
                            if (!selectTile.tileClick) {
                                _self.setClickTileDeleteBoxStart(
                                    quadKey,
                                    coordinate,
                                    tile
                                );
                            } else {
                                _self.setClickTileDeleteBoxEnd();
                            }

                            break;
                    }
                    selectTile.tileClick = !selectTile.tileClick; // 클릭 or 클릭아님 모드 변경
                }

                // 로직 시작
                await setClickTile(quadKey, coordinate, tile);
            });

            /**
             * [마우스오버 이벤트]
             * desc : 클릭 이후에 작동, selectBox 기능으로만 사용
             * 현재 위경도를 지속적으로 입력
             * https://docs.mapbox.com/mapbox-gl-js/api/events/#mapmouseevent#type
             * */
            map.on("mousemove", (event) => {
                const _self = {};
                if (map.getZoom() < 17) return; // 줌레벨 17이하 이벤트막기
                if (selectTile.groupClick === true) return; // 그룹 클릭상태는 막기
                if (selectTile.tileClick === false) return; // 클릭안한 상태 이벤트 막기

                /**
                 * 선택모드
                 * */
                _self.setMouseMoveSelect = (_tile) => {
                    let onSelectTemp = {
                        quadKeys: [],
                        coordinates: [],
                    };
                    let geometryJson, calMinMaxXY, rightMode;
                    onSelect.onSelect.end = { tile: _tile };

                    calMinMaxXY = calMinMax(
                        onSelect.onSelect.start,
                        onSelect.onSelect.end
                    ); // x y 의 최소 최대값 반환
                    rightMode =
                        onSelect.onSelect.end.tile[0] >
                        onSelect.onSelect.start.tile[0]; // 오른쪽 모드인지 체크

                    for (
                        let i = rightMode
                            ? calMinMaxXY.x.min
                            : calMinMaxXY.x.max;
                        rightMode
                            ? i <= calMinMaxXY.x.max
                            : i >= calMinMaxXY.x.min;
                        rightMode ? i++ : i--
                    ) {
                        // 가로
                        for (
                            let j = rightMode
                                ? calMinMaxXY.y.min
                                : calMinMaxXY.y.max;
                            rightMode
                                ? j <= calMinMaxXY.y.max
                                : j >= calMinMaxXY.y.min;
                            rightMode ? j++ : j--
                        ) {
                            // 세로
                            const _tile = [i, j, 21];
                            const _quadKey = tilebelt.tileToQuadkey(_tile);
                            const _coordinate =
                                tilebelt.tileToGeoJSON(_tile).coordinates;
                            const _mergeQuadKeys =
                                onSelect.selected.quadKeys.length +
                                onSelectTemp.quadKeys.length;

                            // 100개 넘어가면 선택불가
                            const isMax = _mergeQuadKeys >= 100;
                            // 기존 맵에 있는지 체크
                            const isExist =
                                onSelect.selected.quadKeys.findIndex(
                                    (item) => item === _quadKey
                                ) !== -1;
                            // 이미 구입된 맵에 있는지 체크
                            const isBuyedTile =
                                onScreen.onBuyed.findIndex(
                                    (item) => item.quadKey === _quadKey
                                ) !== -1;

                            if (!isExist && !isMax && !isBuyedTile) {
                                onSelectTemp.quadKeys.push(_quadKey); // id값
                                onSelectTemp.coordinates.push(_coordinate); // geojson
                            }
                        }
                    }

                    // 기존 선택한 CELL들과 합치는 과정을 해야 한다.
                    const mergeCoordinatesResult = [
                        ...onSelectTemp.coordinates,
                        ...onSelect.selected.coordinates,
                    ];
                    const mergeQuadKeysResult = [
                        ...onSelectTemp.quadKeys,
                        ...onSelect.selected.quadKeys,
                    ];

                    onSelect.onSelect.coordinates = mergeCoordinatesResult;
                    onSelect.onSelect.quadKeys = mergeQuadKeysResult;

                    geometryJson = setGeometryJson(mergeCoordinatesResult);
                    map.getSource("selectData").setData(geometryJson);
                };

                /**
                 * 삭제모드
                 * */
                _self.setMouseMoveDelete = (_tile) => {
                    let onSelectTemp = {
                        quadKeys: [],
                        coordinates: [],
                    };
                    let geometryJson, calMinMaxXY, rightMode;

                    onSelect.onDelete.end = { tile: _tile };
                    calMinMaxXY = calMinMax(
                        onSelect.onDelete.start,
                        onSelect.onDelete.end
                    ); // x y 의 최소 최대값 반환
                    rightMode =
                        onSelect.onDelete.end.tile[0] >
                        onSelect.onDelete.start.tile[0]; // 오른쪽 모드인지 체크

                    for (
                        let i = rightMode
                            ? calMinMaxXY.x.min
                            : calMinMaxXY.x.max;
                        rightMode
                            ? i <= calMinMaxXY.x.max
                            : i >= calMinMaxXY.x.min;
                        rightMode ? i++ : i--
                    ) {
                        // 가로
                        for (
                            let j = rightMode
                                ? calMinMaxXY.y.min
                                : calMinMaxXY.y.max;
                            rightMode
                                ? j <= calMinMaxXY.y.max
                                : j >= calMinMaxXY.y.min;
                            rightMode ? j++ : j--
                        ) {
                            // 세로
                            const _tile = [i, j, 21];
                            const _quadKey = tilebelt.tileToQuadkey(_tile);
                            const _coordinates =
                                tilebelt.tileToGeoJSON(_tile).coordinates;
                            onSelectTemp.quadKeys.push(_quadKey);
                            onSelectTemp.coordinates.push(_coordinates);
                        }
                    }

                    geometryJson = setGeometryJson(onSelectTemp.coordinates);

                    onSelect.onDelete.quadKeys = onSelectTemp.quadKeys;
                    onSelect.onDelete.coordinates = onSelectTemp.coordinates;

                    map.getSource("deleteData").setData(geometryJson);
                };

                const latLng = event.lngLat;
                const lng = latLng.lng,
                    lat = latLng.lat;
                const _tile = tilebelt.pointToTile(lng, lat, 21); // xyz return

                switch (onSelect.mode) {
                    case "select":
                        _self.setMouseMoveSelect(_tile);
                        break;
                    case "delete":
                        console.log();
                        _self.setMouseMoveDelete(_tile);
                        break;
                }
            }); // ======[mousemove]======

            /**
             * [선택 영역 색칠 레이어]
             * desc: 스타일 변경시 타일 선택 재구성용
             * */
            async function setClickTileSelectBoxStart() {
                const geometryJson = setGeometryJson(
                    onSelect.selected.coordinates
                );

                if (map.getSource("selectData") === undefined) {
                    map.addSource("selectData", {
                        type: "geojson",
                        data: geometryJson,
                    });

                    map.addLayer({
                        id: "select_layer",
                        type: "fill",
                        source: "selectData",
                        paint: {
                            "fill-color": "#ffffff",
                            "fill-opacity": 0.5,
                        },
                        minzoom: 17,
                    });
                } else {
                    map.getSource("selectData").setData(geometryJson);
                }

                // 블록정보 창을 위한 렌더링 정보
                setSelectedCellList = {
                    groupSelect: {
                        isGroupSelect: false,
                        groupId: -1,
                        quadKeys: [],
                        coordinates: [],
                    },
                    boxSelect: {
                        quadKeys: onSelect.selected.quadKeys,
                        coordinates: onSelect.selected.coordinates,
                        isBoxSelect: true,
                    },
                };
            }

            /**
             * [그룹선택 영역 색칠 레이어]
             * desc: 그룹 선택시 & 스타일 변경시 타일 선택 재구성용
             * */
            const setClickTileGroupStart = async (onGroupSelect) => {
                const clickAreaTileFeatures = setGeometryJson(
                    onGroupSelect.coordinates
                );
                const clickAreaFlagFeatures = setGeometryJsonMultiPoint(
                    onGroupSelect.center
                );

                // 구매된 tile 레이어 올리기
                if (map.getSource("buyedData") === undefined) {
                    map.addSource("buyedData", {
                        type: "geojson",
                        data: clickAreaTileFeatures,
                    });

                    map.addLayer({
                        id: "buyed_layer",
                        type: "fill",
                        source: "buyedData",
                        paint: {
                            "fill-color": "#79da19",
                            "fill-opacity": 0.5,
                        },
                        minzoom: 17,
                        layout: {
                            visibility: "visible",
                        },
                    });
                } else {
                    map.getSource("buyedData").setData(clickAreaTileFeatures);

                    //국기의 경우 이전 깃발사진으로 된 레이어가 남아있기 때문에 지우고 다시 만들어야 한다. 다시 만드는건 아랫줄에...
                    map.removeLayer("buyedFlag_layer");
                    map.removeSource("buyedFlagData");
                }

                // 국기 레이어 생성 및 재생성
                map.addSource("buyedFlagData", {
                    type: "geojson",
                    data: clickAreaFlagFeatures,
                });

                map.addLayer({
                    id: "buyedFlag_layer",
                    type: "symbol",
                    source: "buyedFlagData",
                    minzoom: 17,
                    layout: {
                        "icon-image": onGroupSelect.country,
                        "icon-size": 1.3,
                    },
                    paint: {
                        "icon-opacity": 1,
                    },
                });

                setSelectedCellList((prevState) => ({
                    ...prevState,
                    groupSelect: {
                        quadKeys: onGroupSelect.quadKeys,
                        coordinates: onGroupSelect.coordinates,
                        isGroupSelect: true,
                    },
                }));

                onSelect.onGroupSelect = onGroupSelect;
                selectTile.groupClick = true;
            };

            map.on("load", async () => {
                console.log('load map')
                // 타일 라인 레이어 생성
                if (map.getSource("tileData") === undefined) {
                    map.addSource("tileData", {
                        type: "geojson",
                        data: geometryJson,
                    });

                    map.addLayer({
                        id: "tile_layer",
                        type: "line",
                        source: "tileData",
                        layout: {
                            "line-join": "round",
                            "line-cap": "round",
                        },
                        paint: {
                            "line-color": "#222222",
                            "line-width": 1,
                            "line-opacity": 0.3,
                        },
                        minzoom: 17,
                    });
                } else {
                    // map.getSource("tileData") !== undefined
                    map.getSource("tileData").setData(geometryJson);
                }

                onScreen.quadKeys = onScreenJson.quadKeys;
                onScreen.coordinates = onScreenJson.coordinates;
                tileStart = true;
                if (onSelect.onGroupSelect.quadKeys.length !== 0)
                    await setClickTileGroupStart(onSelect.onGroupSelect);
                if (onSelect.selected.quadKeys.length !== 0)
                    await setClickTileSelectBoxStart();
            });
        });
        return {
            xPoint,
            yPoint,
            lat,
            lng,
            onSelectedReset,
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
