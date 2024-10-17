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
    import { getBuffer, getMapBounds, getRhumbDistance } from '$lib'
    import StartModal from './StartModal.svelte'
    /**
     * Declare variables
     * let decalres an immutable variable
     * const declares a constant
     *
     * Note the format of markers
     */
    let showModal = false
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
            attackRange: 100,
        },
    ]

    let countTowers = 5
    let disableTracking = true
    let disableMultipleGeolocation = false
    let disableDropTower = true
    let disableRespawnEnemies = true

    // Extent of the map
    let bounds = getMapBounds(towers)

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

    function addTower(t, label, name, attackRange) {
        if (countTowers !== 0) {
            towers = [
                ...towers,
                {
                    lngLat: t.lngLat,
                    label,
                    name,
                    attackRange,
                },
            ]
            countTowers -= 1
        }
        if (countTowers === 0) {
            disableDropTower = true
        }
    }

    // Geolocation API related
    const options = {
        enableHighAccuracy: true,
        timeout: Infinity, // milliseconds
        maximumAge: 0, // milliseconds, 0 disables cached positions
    }
    let getPosition = false
    let success = false
    let error = ''
    let position = {}
    let coords = []

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
    // Watch a position using Geolocation API if you need continuous updates
    let watchPosition = false
    let watchedPosition = {}
    let watchedMarker = {}

    /**
     * Trigger an action when getting close to a marker
     */
    let countEnemies = 0 // number of markers found
    $: if (watchedPosition.coords) { // this block is triggered when watchedPosition is updated
        // The tracked position in marker format
        watchedMarker = {
            lngLat: {
                lng: watchedPosition.coords.longitude,
                lat: watchedPosition.coords.latitude,
            },
        }

        countEnemies = 0
        // Whenever the watched position is updated, check if it is within 10 meters of any marker
        markers.forEach((marker) => {
            const rhumbDistance = getRhumbDistance([watchedMarker, marker])
            const threshold = 50

            if (rhumbDistance <= threshold) {
                countEnemies += 1
            }
        })
    }

    let randomPoints = []
    let lastUpdateTime = 0
    const UPDATE_INTERVAL = 1 * 60 * 1000 // 20 minutes in milliseconds

    // Reactive statement to generate random points when watchedPosition changes
    $: if (watchedPosition.coords) {
        const currentTime = Date.now()
        if (currentTime - lastUpdateTime >= UPDATE_INTERVAL) {
            const { latitude, longitude } = watchedPosition.coords
            randomPoints = generateRandomPoints(latitude, longitude)
            lastUpdateTime = currentTime
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
        const maxDistance = 200 // Maximum distance in meters
        const minEnemies = 1
        const maxEnemies = 20
        const countEnemies = Math.floor(minEnemies + Math.random() * (maxEnemies - minEnemies + 1))

        for (let i = 0; i < countEnemies; i++) {
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
        randomPoints = generateRandomPoints(wm.lngLat.lat, wm.lngLat.lng)
    }

    /**
     * Variables can be initialised without a value and populated later
     * WARNING: this can lead to errors if the variable is used before being
     * assigned a value
     */

    const showGeoJSON = false
    let geojsonData

    let landmarkFeatures = []
    let hoveredLandmark = null
    let debugInfo = ''
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
            console.log('First landmark feature:', landmarkFeatures[0])
            console.log('Coordinates of first feature:', landmarkFeatures[0]?.geometry?.coordinates)
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
        class="btn btn-neutral"
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
        class="btn btn-neutral"
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
        <div class="col-span-4 md:col-span-1 text-center">
            <!-- <Geolocation> tag is used to access the Geolocation API -->
            <!-- {getPosition} is equivalent to getPosition={getPosition} -->
            <!-- bind:variable associates the parameter with the variable with the same name declared in <script> reactively -->
            <!-- let:variable creates a variable for use from the component's return -->
            <Geolocation
                {getPosition}
                options={options}
                bind:position
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

        <div class="col-span-4 md:col-span-1 text-center">
            <button
                class="btn btn-primary transition ease-in-out delay-150 bg-orange-400 hover:-translate-y-1 hover:scale-110 hover:bg-orange-600 duration-300"
                disabled={disableDropTower}
                on:click={() => {
                    addTower(watchedMarker, 'label', 'name', Math.floor(50 + Math.random() * (200 - 50 + 1)) / 1000)
                    console.log(towers)
                }}
            >
                Drop Towers. Remaining : {countTowers}
            </button>
        </div>

        <div class="col-span-4 md:col-span-1 text-center">
            <button
                class="btn transition ease-in-out delay-150 bg-pink-400 hover:-translate-y-1 hover:scale-110 hover:bg-pink-600 duration-300"
                disabled={disableRespawnEnemies}
                on:click={() => {
                    updateRandomPoints(watchedMarker)
                    console.log(randomPoints)
                }}
            >
                Respawn Enemies. Current: {randomPoints.length}
            </button>
        </div>

        <div class="col-span-4 md:col-span-1 text-center">
            <h1 class="font-bold">Found {countEnemies} Enemies in Range</h1>
            Counts enemies in tower range
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
        zoom={10}
    >
        <Marker
            lngLat={[144.97, -37.81]}
            class="w-8 h-8 rounded-full bg-blue-500 cursor-pointer"
        />

        <!-- ... other markers ... -->

        <!-- Custom control buttons -->
        <Control class="flex flex-col gap-y-2">
            <ControlGroup>
                <ControlButton
                    on:click={() => {
                        bounds = getMapBounds(randomPoints)
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

        {#if showGeoJSON}
            <GeoJSON
                id="geojsonData"
                data={geojsonData}
                promoteId="name"
            >
                <FillLayer
                    paint={{
                        'fill-color': hoverStateFilter('purple', 'yellow'),
                        'fill-opacity': 0.3,
                    }}
                    beforeLayerType="symbol"
                    manageHoverState
                >
                    <Popup
                        openOn="hover"
                        let:data
                    >
                        {@const props = data?.properties}
                        {#if props}
                            <div class="flex flex-col gap-2">
                                <p>{props.name}</p>
                            </div>
                        {/if}
                    </Popup>
                </FillLayer>
                <LineLayer
                    layout={{ 'line-cap': 'round', 'line-join': 'round' }}
                    paint={{ 'line-color': 'purple', 'line-width': 3 }}
                    beforeLayerType="symbol"
                />
            </GeoJSON>
        {/if}

        <!-- Displaying markers, this is reactive -->
        <!-- For-each loop syntax -->
        <!-- markers is an object, lngLat, label, name are the fields in the object -->
        <!-- i is the index, () indicates the unique ID for each item, duplicate IDs will lead to errors -->

        {#each markers as { lngLat, label, name }, i (i)}
            <Marker
                {lngLat}
                class="grid h-8 w-14 place-items-center rounded-md border
                    border-gray-200 bg-red-300 text-black shadow-2xl
                    focus:outline-2 focus:outline-black"
            >
                <span>
                    {label}
                </span>

                <Popup
                    openOn="hover"
                    offset={[0, -10]}>
                    <div class="text-lg font-bold">{name}</div>
                </Popup>
            </Marker>
        {/each}

        {#each towers as { lngLat, attackRange }, i (i)}
            <Marker
                {lngLat}
                class="grid h-8 w-8 place-items-center rounded-full
                    border border-white-200
                    bg-orange-900 text-white shadow-2xl focus:outline-2 focus:outline-white"
            >
                <span>
                    {i}
                </span>
                <Popup
                    openOn="hover"
                    offset={[0, -10]}>
                    <div class="text-lg font-bold">Tower {i} Initialized</div>
                </Popup>
            </Marker>
            <GeoJSON
                id="towerBuffer{i}"
                data={getBuffer(lngLat, attackRange)}
            >
                <FillLayer
                    paint={{
                        'fill-color': hoverStateFilter('red', 'orange'),
                        'fill-opacity': 0.3,
                    }}
                    beforeLayerType="symbol"
                    manageHoverState
                >
                </FillLayer>
                <LineLayer
                    layout={{ 'line-cap': 'round', 'line-join': 'round' }}
                    paint={{ 'line-color': 'red', 'line-width': 0.2 }}
                    beforeLayerType="symbol"
                />
            </GeoJSON>

        {/each}

        {#each randomPoints as { lngLat, name }, i (i)}
            <Marker
                {lngLat}
                class="w-4 h-4 rounded-full bg-purple-500"
            >
                <Popup
                    openOn="hover"
                    offset={[0, -10]}>
                    <div class="text-sm">{name}</div>
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
                        'fill-opacity': 0.3,
                    }}
                    beforeLayerType="symbol"
                    manageHoverState
                >
                    <Popup
                        openOn="hover"
                        let:data
                    >
                        {@const props = data?.properties}
                        {#if props}
                            <div class="flex flex-col gap-2">
                                <p>50 meter range</p>
                            </div>
                        {/if}
                    </Popup>
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
