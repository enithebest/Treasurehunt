<script>
    import { onMount } from "svelte";
    import { getDistance, getGreatCircleBearing, isPointWithinRadius } from "geolib";

    // --- Target circle center & radius ---
    const target = { latitude: 42.05913363079434, longitude: 19.51725368912721 };
    const radiusMeters = 5; // 5 m circle

    // --- Reactive state ---
    let userLat = null;
    let userLon = null;
    let distance = null;
    let bearing = null;
    let insideCircle = false;

    let watchId;

    function updatePosition(pos) {
        userLat = pos.coords.latitude;
        userLon = pos.coords.longitude;

        // Calculate distance
        distance = getDistance(
            { latitude: userLat, longitude: userLon },
            target
        );

        // Bearing from user to target (degrees from North)
        bearing = getGreatCircleBearing(
            { latitude: userLat, longitude: userLon },
            target
        );

        // Inside radius?
        insideCircle = isPointWithinRadius(
            { latitude: userLat, longitude: userLon },
            target,
            radiusMeters
        );
    }

    function geoError(err) {
        console.error("Geolocation error:", err);
    }

    onMount(() => {
        if ("geolocation" in navigator) {
            watchId = navigator.geolocation.watchPosition(updatePosition, geoError, {
                enableHighAccuracy: true,
                maximumAge: 0,
                timeout: 10000
            });
        }
        return () => {
            if (watchId) navigator.geolocation.clearWatch(watchId);
        };
    });
</script>

<div class="flex flex-col items-center justify-center min-h-screen bg-gray-50 dark:bg-gray-900 text-gray-800 dark:text-gray-100">
    <h1 class="text-2xl font-bold mb-4">Navigation to Target</h1>

    <div class="mb-6">
        {#if userLat && userLon}
            <p>Distance to target: {distance} m</p>
            <p>Bearing to target: {bearing}° (turn until your heading matches)</p>
            <p>Status:
                {insideCircle
                    ? "✅ Inside 5 m radius"
                    : "❌ Outside radius"}
            </p>
        {:else}
            <p>Waiting for GPS signal…</p>
        {/if}
    </div>

    <!-- Visual indicator -->
    <div
        class="w-16 h-16 rounded-full"
        style="background-color: {insideCircle ? 'green' : 'red'};"
    ></div>
</div>
