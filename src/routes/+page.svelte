<script>
    import { onMount } from "svelte";
    import { getDistance, getGreatCircleBearing, isPointWithinRadius } from "geolib";

    const target = { latitude: 42.05913363079434, longitude: 19.51725368912721 };
    const radiusMeters = 5;

    let userLat = null;
    let userLon = null;
    let distance = null;
    let bearing = null;
    let insideCircle = false;
    let heading = 0;           // Compass heading of the phone (degrees)
    let arrowRotation = 0;     // Final rotation for the arrow

    let watchId;

    function updatePosition(pos) {
        userLat = pos.coords.latitude;
        userLon = pos.coords.longitude;

        distance = getDistance(
            { latitude: userLat, longitude: userLon },
            target
        );

        bearing = getGreatCircleBearing(
            { latitude: userLat, longitude: userLon },
            target
        );

        insideCircle = isPointWithinRadius(
            { latitude: userLat, longitude: userLon },
            target,
            radiusMeters
        );

        updateArrowRotation();
    }

    function updateArrowRotation() {
        if (bearing !== null && heading !== null) {
            // difference between where we need to go and where device is facing
            let diff = bearing - heading;
            // Normalize to 0–360
            arrowRotation = (diff + 360) % 360;
        }
    }

    function geoError(err) {
        console.error("Geolocation error:", err);
    }

    function handleOrientation(e) {
        // alpha is degrees clockwise from magnetic north
        // Some browsers use 'webkitCompassHeading' for iOS
        const rawHeading =
            e.webkitCompassHeading !== undefined
                ? e.webkitCompassHeading
                : e.alpha;

        if (rawHeading != null) {
            heading = rawHeading;
            updateArrowRotation();
        }
    }

    onMount(() => {
        if ("geolocation" in navigator) {
            watchId = navigator.geolocation.watchPosition(updatePosition, geoError, {
                enableHighAccuracy: true,
                maximumAge: 0,
                timeout: 10000
            });
        }
        if (window.DeviceOrientationEvent) {
            window.addEventListener("deviceorientationabsolute", handleOrientation, true);
            // Fallback for browsers not supporting 'absolute'
            window.addEventListener("deviceorientation", handleOrientation, true);
        }

        return () => {
            if (watchId) navigator.geolocation.clearWatch(watchId);
            window.removeEventListener("deviceorientationabsolute", handleOrientation);
            window.removeEventListener("deviceorientation", handleOrientation);
        };
    });
</script>

<main>
<div class="flex flex-col items-center justify-center min-h-screen bg-gray-50 dark:bg-gray-900 text-gray-800 dark:text-gray-100">
    <h1 class="text-2xl font-bold mb-4">Navigation to Target</h1>

    <div class="mb-6 text-center">
        {#if userLat && userLon}
            <p>Distance to target: {distance} m</p>
            <p>Bearing to target: {bearing}°</p>
            <p>Phone heading: {Math.round(heading)}°</p>
            <p>Status:
                {insideCircle
                    ? "Inside 5 m radius"
                    : "Outside radius"}
            </p>
        {:else}
            <p>Waiting for GPS signal…</p>
        {/if}
    </div>

    <div
        class="w-16 h-16 mb-6 rounded-full"
        style="background-color: {insideCircle ? 'green' : 'red'};"
    ></div>

    <!-- Arrow that rotates toward the target -->
    <p class="text-4xl font-bold inline-block"
       style="transform: rotate({arrowRotation}deg); transition: transform 0.1s linear;">
        ➡️
    </p>
</div>
</main>
