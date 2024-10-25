<!-- <script> tag includes JavaScript code -->
<script>
    import { onMount } from 'svelte'
    import Geolocation from 'svelte-geolocation'
    import {
        Control,
        ControlButton,
        ControlGroup,
        FillLayer,
        GeoJSON,
        hoverStateFilter,
        LineLayer,
        MapLibre,
        Marker,
        Popup,
    } from 'svelte-maplibre' // DoNotChange
    /**
     * You can put functions you need for multiple components in a js file in
     * the lib folder, export them in lib/index.js and then import them like this
     */
    import { getBuffer, getMapBounds } from '$lib'
    import * as turf from '@turf/turf'
    import StartModal from '../lib/StartModal.svelte'
    /**
     * Declare variables
     * let decalres an immutable variable
     * const declares a constant
     *
     * Note the format of markers
     */

    let markers = [
        {
            lngLat: {
                lng: 144.9638347277324,
                lat: -37.80967960080751,
            },
            label: 'Marker 1',
            name: 'This is a marker',
        },
    ]

    let totems = [
        {
            lngLat: {
                lng: 144.96383,
                lat: -37.80967,
            },
            label: 'Tower 1',
            name: 'Base Tower',
            attackRange: 0.01,
        },
    ]

    let getPosition = false
    let success = false
    let error = ''
    // let debugInfo = ''
    // let hoveredLandmark = null
    let lastUpdateTime = 0
    const UPDATE_INTERVAL = 0.5 * 60 * 1000 // minutes in milliseconds
    const limitTotems = 5
    let countTotems = 0
    const playerRange = 0.5 // KM
    let countSummons = 0
    let manaPool = 0

    const minEnemies = 5
    const maxEnemies = 50
    const minTotemRangeMetres = 10
    const maxTotemRangeMetres = 25

    // buttons and events
    let disableTracking = true
    let disableMultipleGeolocation = false
    let disableConjureTotem = true
    let disableRespawnEnemies = true
    const disableConjureExtraTotem = true
    // empty placeholders
    let position = {}
    let coords = []
    let familiarEntities = []
    let zoneData
    let randomEnemies = []
    let currentTargets = null
    // Extent of the map
    let bounds = getMapBounds(totems)
    let closestFamiliar = null
    let closestFamiliarEntity

    let showPopup = true
    let showModal = false
    const enemyIcons = ['👻', '🧛', '🧟', '👽', '👾', '👹']
    const statusIcons = ['🔥', '❄️', '⚡', '☠️', '💤', '🩸']

    // Add this function to check for the closest landmark in range
    function checkTotemsNearby(watchedMarkerLngLat) {
        if (!watchedMarkerLngLat || !familiarEntities) {
            return
        }

        const bufferRadius = playerRange
        let minDistance = Infinity
        closestFamiliar = null

        familiarEntities.forEach((feature) => {
            const [lng, lat] = feature.geometry.coordinates
            const dx = lng - watchedMarkerLngLat.lng
            const dy = lat - watchedMarkerLngLat.lat
            const distance = Math.sqrt(dx * dx + dy * dy)

            if (distance <= bufferRadius && distance < minDistance) {
                minDistance = distance
                closestFamiliarEntity = feature
                closestFamiliar = {
                    feature_na: feature.properties.feature_na || 'Unnamed',
                    type: feature.properties.theme || 'No type',
                    subtype: feature.properties.sub_theme || 'No subtype',
                }
            }
        })
    }

    /**
     * Declaring a function
     *
     * Functions declared in <script> can only be used in this component
     */

    // eslint-disable-next-line no-unused-vars, unused-imports/no-unused-vars
    function addMarker(e, label, name) {
        markers = [
            ...markers,
            {
                lngLat: e.detail.lngLat,
                label,
                name,
            },
        ]
    }

    /**
     * Adds a tower on the watchedMarker location, and generates a random attack range
     * @param t
     * @param label
     * @param name
     */
    function addTotem(t, label, name) {
        if (countTotems >= limitTotems) {
            disableConjureTotem = true
        }
        else {
            totems = [
                ...totems,
                {
                    lngLat: t.lngLat,
                    label,
                    name,
                    attackRange: Math.floor(minTotemRangeMetres + Math.random() * (maxTotemRangeMetres + minTotemRangeMetres + 1)) / 1000,
                },
            ]
            countTotems = countTotems + 1
        }
    }

    /**
     * Generates 15 random points around the given location within a 200-meter radius
     * @param {number} lat - Latitude of the current location
     * @param {number} lng - Longitude of the current location
     * @returns {Array} Array of objects containing lat and lng of random points
     */
    function generateRandomPoints(lat, lng) {
        const points = []
        const earthRadius = 6371000 // Earth's radius in meters
        const maxDistance = 100 // Maximum distance in meters
        const countTargets = Math.floor(minEnemies + Math.random() * (maxEnemies - minEnemies + 1))

        for (let i = 0; i < countTargets; i++) {
            // Generate random distance and angle
            const randomDistance = Math.random() * maxDistance
            const randomAngle = Math.random() * 2 * Math.PI

            // Calculate offset in latitude and longitude
            const latOffset = (randomDistance / earthRadius) * (180 / Math.PI)
            const lngOffset = (randomDistance / earthRadius) * (180 / Math.PI) / Math.cos(lat * Math.PI / 180)

            // Calculate new lat and lng
            const newLat = lat + latOffset * Math.cos(randomAngle)
            const newLng = lng + lngOffset * Math.sin(randomAngle)

            points.push({
                lngLat: { lng: newLng, lat: newLat },
                label: `Random ${i + 1}`,
                name: `Random point ${i + 1}`,
            })
        }

        return points
    }

    function updateRandomPoints(wm) {
        randomEnemies = generateRandomPoints(wm.lngLat.lat, wm.lngLat.lng)
        if (currentTargets.features.length) {
            manaPool = manaPool + currentTargets.features.length
        }
    }

    // Geolocation API related
    const options = {
        enableHighAccuracy: true,
        timeout: 10000, // milliseconds Infinity
        maximumAge: 500, // milliseconds, 0 disables cached positions
    }

    let accuracy = ''
    let heading = ''
    let speed = ''
    /**
     * $: indicates a reactive statement, meaning that this block of code is
     * executed whenever the variable used as the condition changes its value
     *
     * In this case: whenever success is set to true, a Position object
     * has been successfully obtained. Immediately update the relevant variables
     */
    $: if (success || error) {
        // reset the flag
        getPosition = false
    }

    $: if (success) {
        coords = [position.coords.longitude, position.coords.latitude]
        accuracy = position.coords.accuracy
        heading = position.coords.heading
        speed = position.coords.speed
    /*
        markers = [
            ...markers,
            {
                lngLat: { lng: coords[0], lat: coords[1] },
                label: 'Current',
                name: 'Current Position',
            },
        ]
    */
    }

    $: if (error) {
        showModal = true
        disableMultipleGeolocation = false
        disableTracking = true
    }
    // Watch a position using Geolocation API if you need continuous updates
    let watchPosition = false
    let watchedPosition = {}
    let watchedMarker = {}
    /**
     * Trigger an action when getting close to a marker
     */

    let countFamiliars = 0
    $: if (watchedPosition.coords) { // this block is triggered when watchedPosition is updated
        // The tracked position in marker format
        watchedMarker = {
            lngLat: {
                lng: watchedPosition.coords.longitude,
                lat: watchedPosition.coords.latitude,
            },
        }

        checkTotemsNearby(watchedMarker.lngLat)
        countFamiliars = 0
        familiarEntities.forEach((landmark) => {
            const pl = turf.point([landmark.geometry.coordinates[0], landmark.geometry.coordinates[1]])
            const loc = turf.point([watchedPosition.coords.longitude, watchedPosition.coords.latitude])
            const pr = turf.buffer(loc, playerRange, { units: 'kilometers' })

            if (turf.booleanPointInPolygon(pl, pr)) {
                countFamiliars = countFamiliars + 1
            }
        })

        const currentTime = Date.now()
        if (currentTime - lastUpdateTime >= UPDATE_INTERVAL) {
            const { latitude, longitude } = watchedPosition.coords
            randomEnemies = generateRandomPoints(latitude, longitude)
            lastUpdateTime = currentTime
        }
    }
    /**
     * Point in Polygon
     */
    let countTargets = 0 // number of markers found

    $: if (watchedPosition.coords && totems.length && randomEnemies.length) {
        watchedMarker = {
            lngLat: {
                lng: watchedPosition.coords.longitude,
                lat: watchedPosition.coords.latitude,
            },
        }

        const towerRanges = []
        totems.forEach((tower) => {
            const pt = turf.point([tower.lngLat.lng, tower.lngLat.lat])
            const bf = turf.buffer(pt, tower.attackRange, { units: 'kilometers' })
            towerRanges.push(bf)
        })

        const pa = []
        const totemRangeFeature = turf.featureCollection(towerRanges)
        const totemArea = turf.dissolve(totemRangeFeature)

        randomEnemies.forEach((enemy) => {
            pa.push([enemy.lngLat.lng, enemy.lngLat.lat])
        })
        const pe = turf.points(pa)
        currentTargets = turf.pointsWithinPolygon(pe, totemArea)

        if (currentTargets.features.length) {
            countTargets = currentTargets.features.length
        }
        else {
            countTargets = 0
        }
    }

    /**
     * Variables can be initialised without a value and populated later
     * WARNING: this can lead to errors if the variable is used before being
     * assigned a value
     */

    /**
     * onMount is executed immediately after the component is mounted, it can be
     * used to load large datasets or to execute code required after the page
     * has been loaded
     *
     * async/await indicate an asynchronous function (i.e., program is paused
     * when a line marked with await starts and resumes when it is resolved)
     *
     * Asset files (e.g., data files, images) can be put in static folder
     *
     * Another way to load data files is to use a URL to the file hosted
     * on a remote server. Try this by replacing 'melbourne.geojson' with
     * 'https://raw.githubusercontent.com/codeforgermany/click_that_hood/main/public/data/melbourne.geojson'
     */
    onMount(async () => {
        showModal = true
        try {
            const landmarksResponse = await fetch('landmarks1011.geojson')
            const landmarksData = await landmarksResponse.json()
            familiarEntities = landmarksData.features
        // debugInfo = `Loaded ${familiarEntities.length} features`
        }
        catch (error) {
            console.error('Error loading landmark GeoJSON data:', error)
            debugInfo = `Error: ${error.message}`
        }

        try {
            const boundsResponse = await fetch('melbourne.geojson')
            zoneData = await boundsResponse.json()
            const boundsFeatures = zoneData.features
            // debugInfo = `Loaded ${boundsFeatures.length} zones`
            console.log('Coordinates of first zone:', boundsFeatures[0]?.geometry?.coordinates)
        }
        catch (error) {
            console.error('Error loading bounds GeoJSON data:', error)
        // debugInfo = `Error: ${error.message}`
        }
    })

/*
    function handleMouseEnter(feature) {
        hoveredLandmark = feature.properties.feature_na || 'Unnamed Landmark'
    }

    function handleMouseLeave() {
        hoveredLandmark = null
    }
        */
</script>

<StartModal
    class="modal-middle"
    bind:showModal
    bind:error>

    <h1
        class="font-bold text-center"
        slot="header">
        POSITION AND LOCATION TRACKING
    </h1>
    {#if error}
        <div class="text-center font-medium text-red-500">An error occurred. Error code {error.code}: {error.message}.</div>
        <div class="text-center font-medium text-red-500">Please Enable Location. Refresh Page if No Prompt from Device</div>
    {/if}

    <!-- on:click declares what to do when the button is clicked -->
    <!-- In the HTML part, {} tells the framework to treat what's inside as code (variables or functions), instead of as strings -->
    <!-- () => {} is an arrow function, almost equivalent to function foo() {} -->
    <h1 class="font-bold">Enable Position</h1>
    <button
        class="btn btn-xs sm:btn-sm md:btn-md lg:btn-lg btn-accent"
        disabled={disableMultipleGeolocation}
        on:click={() => {
            getPosition = true
            disableTracking = false
            disableMultipleGeolocation = true
        }}
    >
        Enable Position
    </button>

    <h1 class="font-bold">Enable Location Tracking</h1>

    <button
        class="btn btn-xs sm:btn-sm md:btn-md lg:btn-lg btn-secondary"
        disabled={disableTracking}
        on:click={() => {
            watchPosition = true
            disableTracking = true
            disableRespawnEnemies = false
            if (countTotems >= limitTotems) { disableConjureTotem = true }
            else { disableConjureTotem = false }
        }}
    >
        Track Location
    </button>

</StartModal>

<!-- Everything after <script> will be HTML for rendering -->
<!-- This section demonstrates how to get the current user location -->
<div class="flex flex-col h-[calc(100vh-80px)] w-full">

    <!-- This section demonstrates how to make a web map using MapLibre -->
    <!-- More basemap options -->
    <!-- "https://basemaps.cartocdn.com/gl/positron-gl-style/style.json" -->
    <!-- "https://tiles.basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json" -->
    <!-- "https://tiles.basemaps.cartocdn.com/gl/voyager-gl-style/style.json" -->
    <MapLibre
        center={[144.97, -37.81]}
        class="map flex-grow min-h-[500px]"
        standardControls
        style="https://tiles.basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json"
        bind:bounds
        zoom={12}
    >
        <!-- Custom control buttons -->
        <Control class="flex flex-col gap-y-2">
            <ControlGroup>
                <ControlButton
                    on:click={() => {
                        bounds = getMapBounds(randomEnemies)
                    }}
                >
                    Fly
                </ControlButton>
            </ControlGroup>
        </Control>

        <!-- Totems -->
        {#if totems.length > 0}
            {#each totems as { lngLat, attackRange }, i (i)}
                <Marker
                    {lngLat}
                    class="grid h-8 w-8 place-items-center rounded-full
                        border border-white-200
                        bg-orange-500 text-white shadow-2xl focus:outline-2 focus:outline-white"
                >
                    <span class="text-xl">⚚</span>
                    <Popup
                        openOn="click"
                        offset={[0, -10]}>
                        <div class="break-words text-lg font-bold">
                            Totem {i} Conjured <br />
                            Effect Range: {attackRange * 1000}
                        </div>
                    </Popup>
                </Marker>
                <GeoJSON
                    id="towerBuffer{i}"
                    data={getBuffer(lngLat, attackRange)}
                >
                    <FillLayer
                        paint={{
                            'fill-color': hoverStateFilter('red', 'yellow'),
                            'fill-opacity': 0.1,
                        }}
                        beforeLayerType="symbol"
                        manageHoverState
                    >
                    </FillLayer>
                    <LineLayer
                        layout={{ 'line-cap': 'round', 'line-join': 'round' }}
                        paint={{ 'line-color': 'red', 'line-width': 0.3 }}
                        beforeLayerType="symbol"
                    />
                </GeoJSON>

            {/each}
        {/if}

        <!-- Targets -->

        <!-- Closest Familiar Entity Landmark -->
        {#if closestFamiliarEntity}
            <Marker
                lngLat={closestFamiliarEntity.geometry.coordinates}
                class="flex items-center justify-center rounded-full w-8 h-8
                    border border-white-200
                    bg-yellow-500 text-white shadow-2xl focus:outline-4 focus:outline-white"
            >
                <span class="items-center text-2xl">🗿</span>
            </Marker>
        {/if}

        <!-- Targeted features -->
        {#if currentTargets}
            {#each currentTargets.features as feature (feature)}
                <Marker
                    lngLat={feature.geometry.coordinates}
                    class="grid h-12 w-12 place-items-center rounded-full"
                >
                    <span class="flex items-center w-12 h-12">{statusIcons[Math.floor(0 + Math.random() * (5 + 0 + 1))]}</span>
                </Marker>
            {/each}
        {/if}

        <!-- if Suburb Data is Loaded -->
        {#if zoneData}
            <GeoJSON
                id="zoneData"
                data={zoneData}
                promoteId="name"
            >
                <FillLayer
                    paint={{
                        'fill-color': hoverStateFilter('white', 'yellow'),
                        'fill-opacity': 0.1,
                    }}
                    beforeLayerType="symbol"
                    manageHoverState
                >
                    <Popup
                        openOn="click"
                        let:data
                    >
                        {@const props = data?.properties}
                        {#if props}
                            <div class="flex flex-col gap-2">
                                <p>Enemy Zone {props.name}</p>
                            </div>
                        {/if}
                    </Popup>
                </FillLayer>
                <LineLayer
                    layout={{ 'line-cap': 'round', 'line-join': 'round' }}
                    paint={{ 'line-color': 'white', 'line-width': 5 }}
                    beforeLayerType="symbol"
                />
            </GeoJSON>
        {/if}

        <!-- if watched position is updated -->
        {#if watchedMarker.lngLat}

            <Marker
                lngLat={watchedMarker.lngLat}
                class="grid h-12 w-12 place-items-center rounded-full
                    border border-white-200
                    bg-gray-400 text-white shadow-2xl focus:outline-2 focus:outline-white"
            >
                <span class="text-2xl">🪄</span>
                <Popup
                    openOn="click"
                    offset={[0, -10]}>
                    <div class="break-words text-lg text-orange-900 font-bold">Exorcist</div>
                </Popup>
            </Marker>

            <GeoJSON
                id="watchedMarkerBuffer"
                data={getBuffer(watchedMarker.lngLat, playerRange)}
            >
                <FillLayer
                    paint={{
                        'fill-color': hoverStateFilter('yellow', 'orange'),
                        'fill-opacity': 0.1,
                    }}
                    beforeLayerType="symbol"
                    manageHoverState
                >
                    {#if closestFamiliar && showPopup}
                        <Popup
                            lngLat={watchedMarker.lngLat}
                            closeButton={false}
                            closeOnClick={false}
                        >
                            <div class="text-lg font-bold">✨ Summonable Familiar:</div>
                            <div class="mt-2">
                                <div class="text-lg text-green-700"><strong>Name:</strong> {closestFamiliar.feature_na}</div>
                            </div>
                            <button
                                class="btn btn-accent"
                                on:click={() => {
                                    showPopup = false
                                    countSummons = countSummons + 1
                                }}
                            >
                                🕯️ Summon Familiar
                            </button>
                        </Popup>
                    {/if}
                </FillLayer>
                <LineLayer
                    layout={{ 'line-cap': 'round', 'line-join': 'round' }}
                    paint={{ 'line-color': 'white', 'line-width': 3 }}
                    beforeLayerType="symbol"
                />
            </GeoJSON>
        {/if}
        <!-- A map event to add a marker when clicked -->
        <!-- MapEvents on:click={event => addMarker(event, 'Added', 'This is an added marker')} / -->

        <!-- This is how GeoJSON datasets are rendered -->
        <!-- promoteId must be a unique ID field in properties of each feature -->
        <!-- Displaying markers, this is reactive -->
        <!-- For-each loop syntax -->
        <!-- markers is an object, lngLat, label, name are the fields in the object -->
        <!-- i is the index, () indicates the unique ID for each item, duplicate IDs will lead to errors -->

        {#each randomEnemies as { lngLat }, i (i)}
            <Marker
                {lngLat}
                class="flex items-center justify-center w-8 h-8"
            >
                <span class="text-2xl">{enemyIcons[Math.floor(0 + Math.random() * (5 + 0 + 1))]}</span>
                <Popup
                    openOn="click"
                    offset={[0, -10]}>
                    <div class="text-lg">Enemy {i}</div>
                </Popup>
            </Marker>
        {/each}

        <!--
            on:mouseenter={() => handleMouseEnter(feature)}
            on:mouseleave={handleMouseLeave}
        -->
        {#each familiarEntities as feature (feature.id)}
            <Marker
                lngLat={feature.geometry.coordinates}
                class="w-6 h-6 rounded-full items-center justify-center bg-black cursor-pointer"
            >
                <span class="text-lg">🗿</span>
                <Popup
                    openOn="click"
                    offset={[0, -10]}>
                    <div class="text-lg">Familiar {feature.id} </div>
                </Popup>
            </Marker>
        {/each}

    </MapLibre>

    <!-- grid, grid-cols-#, col-span-#, md:xxxx are some Tailwind utilities you can use for responsive design -->
    <div class="grid grid-cols-4">
        <div class="col-span-1 md:col-span-1 text-center">
            <button
                class="btn btn-s sm:btn-sm md:btn-md lg:btn-lg btn-primary"
                disabled={disableConjureTotem}
                on:click={() => {
                    addTotem(watchedMarker, 'label', 'name', Math.floor(minTotemRangeMetres + Math.random() * (maxTotemRangeMetres - minTotemRangeMetres + 1)) / 1000)
                }}
            >
                Conjure Totems
            </button>
        </div>

        <div class="col-span-1 md:col-span-1 text-center">
            <button
                class="btn btn-s sm:btn-sm md:btn-md lg:btn-lg btn-secondary"
                disabled={disableRespawnEnemies}
                on:click={() => {
                    updateRandomPoints(watchedMarker)
                    if (currentTargets.features.length) {
                        manaPool = manaPool + currentTargets.features.length
                    }
                }}
            >
                Respawn Enemies
            </button>
        </div>

        <div class="col-span-1 md:col-span-1 text-center">
            <button
                class="btn btn-s sm:btn-sm md:btn-md lg:btn-lg btn-accent"
                disabled={disableConjureExtraTotem}
                on:click={() => {
                }}
            >
                Conjure Extra Totems
            </button>
        </div>

        <div class="col-span-1 md:col-span-1 text-center">
            <button
                class="btn btn-s sm:btn-sm md:btn-md lg:btn-lg btn-neutral"
                on:click={() => {
                    showModal = !showModal
                    disableTracking = false
                }}
            >
                Show Location Modal
            </button>
        </div>

        <div class="col-span-2 md:col-span-1 text-center">
            <h1 class="font-bold text-center">COUNTS</h1>
            <hr />
            <h2 class="font-semibold text-orange-300">{countTotems} of {limitTotems} Totems Conjured</h2>
            <h2 class="font-semibold text-purple-300">{countTargets} of {randomEnemies.length} Enemies Affected</h2>
            <h2 class="font-semibold text-green-300">{countFamiliars} Familiars Nearby</h2>
        </div>
        <div class="col-span-2 md:col-span-1 text-center">
            <h1 class="font-bold text-center">RESOURCES</h1>
            <hr />
            <h2 class="font-semibold text-blue-300">{manaPool} Mana Recovered </h2>
            <h2 class="font-semibold text-green-300">{countSummons} Familiars Summoned </h2>
            <h2 class="font-semibold text-orange-300">0 Extra Totems to Conjure </h2>
        </div>
        <!-- This section demonstrates how to get automatically updated user location -->
        <div class="col-span-4 md:col-span-1 text-center">

            <Geolocation
                getPosition={watchPosition}
                options={options}
                watch={true}
                bind:watchedPosition
                on:position={(e) => {
                    watchedPosition = e.detail
                }}
            />
            {#if watchedPosition.coords}
                <h1 class="font-bold text-left">Current Position: {Number.parseFloat(watchedPosition.coords.longitude)}, {Number.parseFloat(watchedPosition.coords.latitude)}</h1>
                <h1 class="font-bold text-left"> Accuracy: {Number.parseFloat(watchedPosition.coords.accuracy).toFixed(4)} m </h1>
                {#if Number.parseFloat(watchedPosition.coords.accuracy) > 100.0}
                    <p class="break-words text-left text-red-500">Device may be using WIFI/5G Network</p>
                {:else}
                    <p class="break-words text-left text-green-500">Device using GNSS</p>
                {/if}
                {#if watchedPosition.coords.altitude}
                    <h1 class="font-bold break-words text-left">Updated Altitude: {watchedPosition.coords.altitude} | Altitude Accuracy: {Number.parseFloat(watchedPosition.coords.altitudeAccuracy).toFixed(4)} m </h1>
                {/if}
                {#if watchedPosition.coords.heading || watchedPosition.coords.speed}
                    <h1 class="font-bold break-words text-left">Heading: {Number.parseFloat(watchedPosition.coords.heading).toFixed(4)} | Speed: {Number.parseFloat(watchedPosition.coords.speed).toFixed(4)}</h1>
                    <!-- p class="break-words text-left">{JSON.stringify(watchedPosition)}</p -->
                {/if}
            {/if}

        </div>
        <div class="col-span-4 md:col-span-1 bg-white text-center">
            <!-- <Geolocation> tag is used to access the Geolocation API -->
            <!-- {getPosition} is equivalent to getPosition={getPosition} -->
            <!-- bind:variable associates the parameter with the variable with the same name declared in <script> reactively -->
            <!-- let:variable creates a variable for use from the component's return -->
            <Geolocation
                {getPosition}
                options={options}
                bind:position
                bind:accuracy
                bind:speed
                bind:heading
                bind:success
                bind:error
                let:loading
                let:notSupported
            >
                <!-- If-else block syntax -->
                {#if notSupported}
                    Your browser does not support the Geolocation API.
                {:else}
                    {#if loading}
                        Loading...
                    {/if}
                    {#if success}
                        Success!
                    {/if}
                    {#if error}
                        An error occurred. Error code {error.code}: {error.message}.
                    {/if}
                {/if}
            </Geolocation>

            <p class="break-words text-left">Initial Position: {coords}</p>
            <p class="break-words text-left">Accuracy: {Number.parseFloat(accuracy).toFixed(4)}m</p>
            {#if Number.parseFloat(accuracy) > 100.0}
                <p class="break-words text-left text-red-500">ONSTART: Device may be using WIFI/5G Network</p>
            {:else}
                <p class="break-words text-left text-green-500">ONSTART: Device using GNSS</p>
            {/if}
            {#if heading || speed}
                <p class="break-words text-left">Heading: {heading.toFixed(4)} | Speed: {speed.toFixed(4)} </p>
            {/if}
        </div>

    </div>

    <!-- div class="absolute top-4 right-4 bg-white p-2 rounded shadow">
        Debug: {debugInfo}
    </div>

    {#if hoveredLandmark}
        <div class="absolute top-4 left-4 bg-white p-2 rounded shadow">
            {hoveredLandmark}
        </div>
    {/if}
-->
</div>

<!-- Optionally, you can have a <style> tag for CSS at the end, but with TailwindCSS it is usually not necessary -->
