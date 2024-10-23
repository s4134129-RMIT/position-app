<!-- <script> tag includes JavaScript code -->
<script>
    import { onMount } from 'svelte'
    import Geolocation from 'svelte-geolocation'
    import {
        Control,
        ControlButton,
        ControlGroup,
        DefaultMarker,
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

    let towers = [
        {
            lngLat: {
                lng: 144.96383,
                lat: -37.80967,
            },
            label: 'Tower 1',
            name: 'My Tower # 1',
            attackRange: 0.01,
        },
    ]

    let getPosition = false
    let success = false
    let error = ''
    let debugInfo = ''

    let hoveredLandmark = null
    let lastUpdateTime = 0
    const UPDATE_INTERVAL = 1 * 60 * 1000 // 20 minutes in milliseconds
    const limitTowers = 5
    let countTowers = 0

    const minEnemies = 5
    const maxEnemies = 50
    const minTowerRangeMetres = 5
    const maxTowerRangeMetres = 10

    // buttons and events
    let disableTracking = true
    let disableMultipleGeolocation = false
    let disableDropTower = true
    let disableRespawnEnemies = true
    // empty placeholders
    let position = {}
    let coords = []
    let landmarkFeatures = []
    let randomEnemies = []
    // Extent of the map
    let bounds = getMapBounds(towers)
    let closestLandmark = null

    let showPopup = true
    let showModal = false
    const enemyIcons = ['👻', '🧛', '🧟', '👽', '👾', '👹']

    function closePopup() {
        showPopup = false
    }

    // Add this function to check for the closest landmark in range
    function checkLandmarksInRange(watchedMarkerLngLat) {
        if (!watchedMarkerLngLat || !landmarkFeatures) {
            return
        }

        const bufferRadius = 1 // 50 meters in degrees, matching your buffer size
        let minDistance = Infinity
        closestLandmark = null

        landmarkFeatures.forEach((feature) => {
            const [lng, lat] = feature.geometry.coordinates
            const dx = lng - watchedMarkerLngLat.lng
            const dy = lat - watchedMarkerLngLat.lat
            const distance = Math.sqrt(dx * dx + dy * dy)

            if (distance <= bufferRadius && distance < minDistance) {
                minDistance = distance
                closestLandmark = {
                    feature_na: feature.properties.feature_na || 'Unnamed',
                    theme: feature.properties.theme || 'No theme',
                    subtheme: feature.properties.sub_theme || 'No subtheme',
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
    function addTower(t, label, name) {
        if (limitTowers !== 0) {
            towers = [
                ...towers,
                {
                    lngLat: t.lngLat,
                    label,
                    name,
                    attackRange: Math.floor(minTowerRangeMetres + Math.random() * (maxTowerRangeMetres + minTowerRangeMetres + 1)) / 1000,
                },
            ]
            countTowers = countTowers + 1
        }
        if (countTowers === limitTowers) {
            disableDropTower = true
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
    }

    // Geolocation API related
    const options = {
        enableHighAccuracy: true,
        timeout: Infinity, // milliseconds
        maximumAge: 0, // milliseconds, 0 disables cached positions
    }

    let accuracy = ''
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
    }
    // Watch a position using Geolocation API if you need continuous updates
    let watchPosition = false
    let watchedPosition = {}
    let watchedMarker = {}
    /**
     * Trigger an action when getting close to a marker
     */

    let countLandmarks = 0
    $: if (watchedPosition.coords) { // this block is triggered when watchedPosition is updated
        // The tracked position in marker format
        watchedMarker = {
            lngLat: {
                lng: watchedPosition.coords.longitude,
                lat: watchedPosition.coords.latitude,
            },
        }

        checkLandmarksInRange(watchedMarker.lngLat)
        countLandmarks = 0
        landmarkFeatures.forEach((landmark) => {
            const pl = turf.point([landmark.geometry.coordinates[0], landmark.geometry.coordinates[1]])
            const loc = turf.point([watchedPosition.coords.longitude, watchedPosition.coords.latitude])
            const pr = turf.buffer(loc, 20, { units: 'kilometers' })

            if (turf.booleanPointInPolygon(pl, pr)) {
                countLandmarks = countLandmarks + 1
            }
        })

        const currentTime = Date.now()
        if (currentTime - lastUpdateTime >= UPDATE_INTERVAL) {
            const { latitude, longitude } = watchedPosition.coords
            randomEnemies = generateRandomPoints(latitude, longitude)
            lastUpdateTime = currentTime
        }
    }
    let countTargets = 0 // number of markers found
    $: if (watchedPosition.coords && towers.length && randomEnemies.length) {
        watchedMarker = {
            lngLat: {
                lng: watchedPosition.coords.longitude,
                lat: watchedPosition.coords.latitude,
            },
        }

        countTargets = 0
        const towerRanges = []

        towers.forEach((tower) => {
            const pt = turf.point([tower.lngLat.lng, tower.lngLat.lat])
            const bf = turf.buffer(pt, tower.attackRange, { units: 'kilometers' })
            towerRanges.push(bf)
        })

        const towerRangeFeature = turf.featureCollection(towerRanges)
        const towerCoverage = turf.dissolve(towerRangeFeature)
        // console.log(towerCoverage)

        randomEnemies.forEach((enemy) => {
            const pe = turf.point([enemy.lngLat.lng, enemy.lngLat.lat])

            if (turf.booleanPointInPolygon(pe, towerCoverage.features[0])) {
                countTargets += 1
            }
        })
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
            const response = await fetch('landmarks1011.geojson')
            const data = await response.json()
            landmarkFeatures = data.features
            debugInfo = `Loaded ${landmarkFeatures.length} features`
        // console.log('First landmark feature:', landmarkFeatures[0])
            // console.log('Coordinates of first feature:', landmarkFeatures[0]?.geometry?.coordinates)
        }
        catch (error) {
            console.error('Error loading landmark GeoJSON data:', error)
            debugInfo = `Error: ${error.message}`
        }
    })

    function handleMouseEnter(feature) {
        hoveredLandmark = feature.properties.feature_na || 'Unnamed Landmark'
    }

    function handleMouseLeave() {
        hoveredLandmark = null
    }
</script>

<StartModal
    class="modal-middle"
    bind:showModal>

    <h1
        class="font-bold"
        slot="header">Get your current position on the map.</h1>
    <div class="text-center font-medium text-red-500">Track Location will be disabled until this is done.</div>

    <!-- on:click declares what to do when the button is clicked -->
    <!-- In the HTML part, {} tells the framework to treat what's inside as code (variables or functions), instead of as strings -->
    <!-- () => {} is an arrow function, almost equivalent to function foo() {} -->
    <button
        class="btn btn-xs sm:btn-sm md:btn-md lg:btn-lg btn-primary"
        disabled={disableMultipleGeolocation}
        on:click={() => {
            getPosition = true
            disableTracking = false
            disableMultipleGeolocation = true
        }}
    >
        Get Position
    </button>

    <h1 class="font-bold">Track position while moving</h1>

    <button
        class="btn btn-xs sm:btn-sm md:btn-md lg:btn-lg btn-primary"
        disabled={disableTracking}
        on:click={() => {
            watchPosition = true
            disableDropTower = false
            disableTracking = true
            disableRespawnEnemies = false
        }}
    >
        Track
    </button>

</StartModal>

<!-- Everything after <script> will be HTML for rendering -->
<!-- This section demonstrates how to get the current user location -->
<div class="flex flex-col h-[calc(100vh-80px)] w-full">
    <!-- grid, grid-cols-#, col-span-#, md:xxxx are some Tailwind utilities you can use for responsive design -->
    <div class="grid grid-cols-4">
        <div class="col-span-2 md:col-span-1 text-center">
            <button
                class="btn sm:btn-sm md:btn-md lg:btn-lg btn-accent"
                disabled={disableDropTower}
                on:click={() => {
                    addTower(watchedMarker, 'label', 'name', Math.floor(minTowerRangeMetres + Math.random() * (maxTowerRangeMetres - minTowerRangeMetres + 1)) / 1000)
                // console.log(towers)
                }}
            >
                Drop Towers. Remaining : {limitTowers - countTowers}
            </button>
        </div>

        <div class="col-span-2 md:col-span-1 text-center">
            <button
                class="btn sm:btn-sm md:btn-md lg:btn-lg btn-secondary"
                disabled={disableRespawnEnemies}
                on:click={() => {
                    updateRandomPoints(watchedMarker)
                }}
            >
                Respawn Enemies. Current: {randomEnemies.length}
            </button>
        </div>

        <div class="col-span-4 md:col-span-1 text-center">
            <h1 class="font-bold">Engaging {countTargets} Enemies </h1>
            <h1 class="font-bold">{countTowers} of {limitTowers} Towers Deployed</h1>
            <h1 class="font-bold">{countLandmarks} Landmarks Nearby</h1>
        </div>
        <div class="col-span-4 md:col-span-1 text-center">
            <!-- <Geolocation> tag is used to access the Geolocation API -->
            <!-- {getPosition} is equivalent to getPosition={getPosition} -->
            <!-- bind:variable associates the parameter with the variable with the same name declared in <script> reactively -->
            <!-- let:variable creates a variable for use from the component's return -->
            <Geolocation
                {getPosition}
                options={options}
                bind:position
                bind:accuracy
                let:loading
                bind:success
                bind:error
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

            <p class="break-words text-left">Coordinates: {coords}</p>
            <p class="break-words text-left">Accuracy: {accuracy}</p>
            <!-- Objects cannot be directly rendered, use JSON.stringify() to convert it to a string -->
            <p class="break-words text-left">Position: {JSON.stringify(position)}</p>
        </div>

        <!-- This section demonstrates how to get automatically updated user location -->
        <div class="col-span-4 md:col-span-1 text-center">

            <Geolocation
                getPosition={watchPosition}
                options={options}
                watch={true}
                on:position={(e) => {
                    watchedPosition = e.detail
                }}
            />

            <p class="break-words text-left">watchedPosition: {JSON.stringify(watchedPosition)}</p>
        </div>

    </div>

    <!-- This section demonstrates how to make a web map using MapLibre -->
    <!-- More basemap options -->
    <!-- "https://basemaps.cartocdn.com/gl/positron-gl-style/style.json" -->
    <!-- "https://tiles.basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json" -->
    <!-- "https://tiles.basemaps.cartocdn.com/gl/voyager-gl-style/style.json" -->
    <MapLibre
        center={[144.97, -37.81]}
        class="map flex-grow min-h-[500px]"
        standardControls
        style="https://tiles.basemaps.cartocdn.com/gl/voyager-gl-style/style.json"
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
                    Fit
                </ControlButton>
            </ControlGroup>
        </Control>

        <!-- A map event to add a marker when clicked -->
        <!-- MapEvents on:click={event => addMarker(event, 'Added', 'This is an added marker')} / -->

        <!-- This is how GeoJSON datasets are rendered -->
        <!-- promoteId must be a unique ID field in properties of each feature -->
        <!-- Displaying markers, this is reactive -->
        <!-- For-each loop syntax -->
        <!-- markers is an object, lngLat, label, name are the fields in the object -->
        <!-- i is the index, () indicates the unique ID for each item, duplicate IDs will lead to errors -->
        {#if towers.length > 0}
            {#each towers as { lngLat, attackRange }, i (i)}
                <Marker
                    {lngLat}
                    class="flex items-center justify-center w-12 h-12"
                >
                    <span class="text-3xl">🗼</span>
                    <Popup
                        openOn="click"
                        offset={[0, -10]}>
                        <div class="break-words text-lg font-bold">
                            Tower {i} Initialized
                            KM Range: {attackRange * 1000}
                        </div>
                    </Popup>
                </Marker>
                <GeoJSON
                    id="towerBuffer{i}"
                    data={getBuffer(lngLat, attackRange)}
                >
                    <FillLayer
                        paint={{
                            'fill-color': hoverStateFilter('green', 'yellow'),
                            'fill-opacity': 0.3,
                        }}
                        beforeLayerType="symbol"
                        manageHoverState
                    >
                    </FillLayer>
                    <LineLayer
                        layout={{ 'line-cap': 'round', 'line-join': 'round' }}
                        paint={{ 'line-color': 'green', 'line-width': 0.2 }}
                        beforeLayerType="symbol"
                    />
                </GeoJSON>

            {/each}
        {/if}

        {#each randomEnemies as { lngLat }, i (i)}
            <Marker
                {lngLat}
                class="flex items-center justify-center w-8 h-8"
            >
                <span class="text-2xl">{enemyIcons[Math.floor(1 + Math.random() * (6 + 1 + 1))]}</span>
                <Popup
                    openOn="click"
                    offset={[0, -10]}>
                    <div class="text-sm">Zombie {i}</div>
                </Popup>
            </Marker>
        {/each}

        {#each landmarkFeatures as feature (feature.id)}
            <Marker
                lngLat={feature.geometry.coordinates}
                class="w-4 h-4 rounded-full bg-red-500 cursor-pointer"
                on:mouseenter={() => handleMouseEnter(feature)}
                on:mouseleave={handleMouseLeave}
            />
        {/each}
        <!-- Display the watched position as a marker -->
        {#if watchedMarker.lngLat}

            <DefaultMarker lngLat={watchedMarker.lngLat}>
                <Popup offset={[0, -10]}>
                    <div class="text-lg font-bold">You</div>
                </Popup>
            </DefaultMarker>

            <GeoJSON
                id="watchedMarkerBuffer"
                data={getBuffer(watchedMarker.lngLat, 0.1)}
            >
                <FillLayer
                    paint={{
                        'fill-color': hoverStateFilter('blue', 'yellow'),
                        'fill-opacity': 0.2,
                    }}
                    beforeLayerType="symbol"
                    manageHoverState
                >
                    {#if closestLandmark && showPopup}
                        <Popup
                            lngLat={watchedMarker.lngLat}
                            closeButton={false}
                            closeOnClick={false}
                        >
                            <div class="text-lg font-bold">🔧Repairable Tower:</div>
                            <div class="mt-2">
                                <div><strong>Name:</strong> {closestLandmark.feature_na}</div>
                                <div><strong>Theme:</strong> {closestLandmark.theme}</div>
                                <div><strong>Sub_theme:</strong> {closestLandmark.subtheme}</div>
                            </div>
                            <button
                                class="mt-2 px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600"
                                on:click={closePopup}
                            >
                                Repair
                            </button>
                        </Popup>
                    {/if}
                </FillLayer>
                <LineLayer
                    layout={{ 'line-cap': 'round', 'line-join': 'round' }}
                    paint={{ 'line-color': 'blue', 'line-width': 3 }}
                    beforeLayerType="symbol"
                />
            </GeoJSON>
        {/if}

    </MapLibre>

    <div class="absolute top-4 right-4 bg-white p-2 rounded shadow">
        Debug: {debugInfo}
    </div>

    {#if hoveredLandmark}
        <div class="absolute top-4 left-4 bg-white p-2 rounded shadow">
            {hoveredLandmark}
        </div>
    {/if}
</div>

<!-- Optionally, you can have a <style> tag for CSS at the end, but with TailwindCSS it is usually not necessary -->
