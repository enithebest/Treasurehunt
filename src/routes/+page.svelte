<script>
  import { onMount } from "svelte";
  import { writable, derived } from "svelte/store";
  import {
    isPointWithinRadius,
    getPreciseDistance,
    getGreatCircleBearing,
  } from "geolib";

  // 🎯 Target coordinates
  const target = { latitude: 42.05913363079434, longitude: 19.51725368912721 };

  // ── Reactive stores
  const position = writable(null);
  const heading = writable(0);
  const beta = writable(0);
  const gamma = writable(0);

  // ── Derived values
  const distance = derived(position, ($pos) =>
    $pos ? getPreciseDistance($pos, target) : null
  );
  const inRadius = derived(position, ($pos) =>
    $pos ? isPointWithinRadius($pos, target, 5) : false
  );
  const bearing = derived(position, ($pos) =>
    $pos ? getGreatCircleBearing($pos, target) : 0
  );
  const arrowRotation = derived([bearing, heading], ([$b, $h]) => {
    let rot = $b - $h;
    return (rot + 360) % 360;
  });

  let errorMsg = "";
  let permissionGranted = false;

  async function requestPermission() {
    if (
      typeof DeviceOrientationEvent !== "undefined" &&
      typeof DeviceOrientationEvent.requestPermission === "function"
    ) {
      try {
        const resp = await DeviceOrientationEvent.requestPermission();
        if (resp === "granted") {
          permissionGranted = true;
          setupOrientation();
        } else {
          errorMsg = "Keine Berechtigung für Kompassdaten.";
        }
      } catch (e) {
        errorMsg = "Kompass-Berechtigung fehlgeschlagen.";
      }
    } else {
      permissionGranted = true;
      setupOrientation();
    }
  }

  function setupOrientation() {
    function handleOrientation(e) {
      if (e.webkitCompassHeading !== undefined) {
        heading.set(e.webkitCompassHeading); // iOS
      } else if (e.absolute === true) {
        heading.set(e.alpha || 0); // Android
      }
      beta.set(e.beta || 0);
      gamma.set(e.gamma || 0);
    }
    window.addEventListener("deviceorientationabsolute", handleOrientation, true);
    window.addEventListener("deviceorientation", handleOrientation, true);
  }

  onMount(() => {
    if ("geolocation" in navigator) {
      const watchId = navigator.geolocation.watchPosition(
        (pos) => {
          if (pos.coords.accuracy <= 20) {
            position.set({
              latitude: pos.coords.latitude,
              longitude: pos.coords.longitude,
            });
          }
        },
        (err) => {
          console.error("Standortfehler:", err);
          errorMsg = "Standort konnte nicht ermittelt werden.";
        },
        { enableHighAccuracy: true, maximumAge: 0, timeout: 10000 }
      );
      return () => navigator.geolocation.clearWatch(watchId);
    }
  });
</script>

<!-- Layout container -->
<div class="flex flex-col items-center justify-between w-screen h-screen p-4 text-center">
  <!-- Circle shows if inside radius -->
  <div class="rounded-full shadow-md w-[40vw] h-[40vw] max-w-40 max-h-40 mt-8
              transition-colors duration-300
              { $inRadius ? 'bg-green-500' : 'bg-red-500' }"></div>

  <!-- Distance display -->
  {#if $distance !== null}
    <p class="mt-4 text-lg">Abstand: {$distance.toFixed(1)} m</p>
  {:else}
    <p class="mt-4 text-lg">Abstand: wird berechnet …</p>
  {/if}

  <!-- Compass -->
  <div class="flex flex-col items-center mb-8">
    {#if !permissionGranted}
      <button
        on:click={requestPermission}
        class="px-4 py-2 mb-4 text-white bg-blue-600 rounded-lg shadow hover:bg-blue-700">
        📍 Kompass aktivieren
      </button>
    {/if}

    <!-- Compass circle -->
    <div class="relative flex items-center justify-center w-32 h-32 border-4 border-gray-800 rounded-full bg-gray-100">
      <!-- Arrow -->
      <div
        class="absolute w-0 h-0 border-l-[12px] border-r-[12px] border-l-transparent border-r-transparent border-b-[35px] border-b-red-600 origin-[50%_90%] transition-transform duration-150"
        style="transform: rotate({$arrowRotation}deg);">
      </div>
    </div>

    <p class="mt-2 text-base">➡ Ziel: {$bearing.toFixed(0)}°</p>
    <p class="text-base">🧭 Heading: {$heading.toFixed(0)}°</p>
    <p class="text-sm">β: {$beta.toFixed(0)}° | γ: {$gamma.toFixed(0)}°</p>
  </div>

  {#if errorMsg}
    <p class="text-red-600 font-semibold mt-2">{errorMsg}</p>
  {/if}
</div>
