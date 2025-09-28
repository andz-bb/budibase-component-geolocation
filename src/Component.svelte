<script>
  import { getContext, onDestroy, onMount } from "svelte";
  import { writable } from "svelte/store";
  import L from "leaflet"
  import sanitizeHtml from "sanitize-html"
  import "leaflet/dist/leaflet.css"
  import {
    FullScreenControl,
    LocationControl,
    initMapControls,
  } from "./EmbeddedMapControls"
  import { v4 as uuidv4 } from 'uuid';
  
  export let automode;
  export let automodeDelay;
  export let disabled;
  export let label;
  export let showButton;
  export let showOutput;

  export let latitudeField;
  export let longitudeField;
  export let latitudeValidation;
  export let longitudeValidation;

  const { styleable, notificationStore } = getContext("sdk");
  const component = getContext("component");
  const formContext = getContext("form");
  const formStepContext = getContext("form-step");
  const fieldGroupContext = getContext("field-group");

  let latitude = "";
  let longitude = "";
  let error = "";
  let isLoading = false;
  let latFieldState;
  let latFieldApi;
  let longFieldState;
  let longFieldApi;
  let unsubscribeLatitude;
  let unsubscribeLongitude;

  let initialLatStore = writable(null);
  let initialLongStore = writable(null);

  async function getLocation() {
    if (!navigator.geolocation) {
      error = "Geolocation is not supported by your browser";
    } else {
      isLoading = true;
      await navigator.geolocation.getCurrentPosition(
        (pos) => {
          setLatLng(pos.coords.latitude, pos.coords.longitude);
          isLoading = false;
        },
        (err) => {
          error = `Error(${err.code}): ${err.message}`;
          isLoading = false;
        }
      );
    }
  }

  function setLatLng(lat, lng) {
    mapMarkerGroup.clearLayers()

    latitude = lat;
    longitude = lng;
    latFieldApi.setValue(lat);
    longFieldApi.setValue(lng);

    let candidateMarker = L.marker(
      [lat, lng],
      mapMarkerOptions
    )
    candidateMarker
      .addTo(mapMarkerGroup)
    resetView()
  }

  const formApi = formContext?.formApi;
  const labelPos = fieldGroupContext?.labelPosition || "above";
  $: formStep = formStepContext ? $formStepContext || 1 : 1;
  $: labelClass =
    labelPos === "above" ? "" : `spectrum-FieldLabel--${labelPos}`;

  $: if (formApi && latitudeField) {
    const formFieldLatitude = formApi.registerField(
      latitudeField,
      "number",
      latitude,
      false,
      latitudeValidation,
      formStep
    );

    unsubscribeLatitude = formFieldLatitude.subscribe((value) => {
      latFieldState = value?.fieldState;
      latFieldApi = value?.fieldApi;
    });
  }

  $: if (formApi && longitudeField) {
    const formFieldLongitude = formApi.registerField(
      longitudeField,
      "number",
      longitude,
      false,
      longitudeValidation,
      formStep
    );

    unsubscribeLongitude = formFieldLongitude.subscribe((value) => {
      longFieldState = value?.fieldState;
      longFieldApi = value?.fieldApi;
    });
  }

  $: {
    if (latFieldState?.value !== undefined && $initialLatStore === null) {
      initialLatStore.set(latFieldState.value);
    }

    if (longFieldState?.value !== undefined && $initialLongStore === null) {
      initialLongStore.set(longFieldState.value);
    }
  }

  onMount(async () => {
    mounted = true
    initMap(tileURL, mapAttribution, safeZoomLevel)
    if (automode && !disabled) {
      if (automodeDelay) {
        setTimeout(getLocation, automodeDelay * 1000);
      } else {
        await getLocation();
      }
    }
  });

  onDestroy(() => {
    unsubscribeLatitude?.();
    unsubscribeLongitude?.();
  });

  initMapControls()

  export let showMap;
  export let zoomLevel
  export let zoomEnabled = true
  export let fullScreenEnabled = false
  export let locationEnabled = true
  export let defaultLocation
  export let tileURL = "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
  export let mapAttribution

  const embeddedMapId = `${uuidv4()}-wrapper`

  // Map Button Controls
  const locationControl = new LocationControl({
    position: "bottomright",
    onLocationFail: err => {
      if (err.code === GeolocationPositionError.PERMISSION_DENIED) {
        notificationStore.actions.error(
          "Location requests not permitted. Ensure location is enabled"
        )
      } else if (err.code === GeolocationPositionError.POSITION_UNAVAILABLE) {
        notificationStore.actions.warning(
          "Location could not be retrieved. Try again"
        )
      } else if (err.code === GeolocationPositionError.TIMEOUT) {
        notificationStore.actions.warning(
          "Location request timed out. Try again"
        )
      } else {
        notificationStore.actions.error("Unknown location error")
      }
    },
    onLocationSuccess: pos => {
      cachedDeviceCoordinates = pos
      if (typeof mapInstance === "object") {
        mapInstance.setView(cachedDeviceCoordinates, 15)
      }
    },
  })
  const fullScreenControl = new FullScreenControl({
    position: "topright",
  })
  const zoomControl = L.control.zoom({
    position: "bottomright",
  })

  // Map and marker configuration
  const defaultMarkerOptions = {
    html: '<div><i class="ph ph-map-pin ph-fill" style="font-size: 26px; color: #b12b27;" aria-hidden="true"></i></div>',
    className: "embedded-map-marker",
    iconSize: [26, 26],
    iconAnchor: [13, 26],
    popupAnchor: [0, -13],
  }
  const mapMarkerOptions = {
    icon: L.divIcon(defaultMarkerOptions),
    draggable: false,
    alt: "Location Marker",
  }
  const mapOptions = {
    fullScreen: false,
    zoomControl: false,
    scrollWheelZoom: zoomEnabled,
    minZoomLevel,
    maxZoomLevel,
  }
  const fallbackCoordinates = [51.5072, -0.1276] //London

  let mapInstance
  let mapMarkerGroup = new L.FeatureGroup()
  let mounted = false
  let minZoomLevel = 0
  let maxZoomLevel = 18
  let cachedDeviceCoordinates

  $: safeZoomLevel = parseZoomLevel(zoomLevel)
  $: defaultCoordinates = parseDefaultLocation(defaultLocation)
  $: initMap(tileURL, mapAttribution, safeZoomLevel)
  $: zoomControlUpdated(mapInstance, zoomEnabled)
  $: locationControlUpdated(mapInstance, locationEnabled)
  $: fullScreenControlUpdated(mapInstance, fullScreenEnabled)
  $: width = $component.styles.normal.width
  $: height = $component.styles.normal.height
  $: width, height, mapInstance?.invalidateSize()
  $: defaultCoordinates, resetView()

  const isValidLatitude = value => {
    return !isNaN(value) && value > -90 && value < 90
  }

  const isValidLongitude = value => {
    return !isNaN(value) && value > -180 && value < 180
  }

  const parseZoomLevel = zoomLevel => {
    let zoom = zoomLevel
    if (zoom == null || isNaN(zoom)) {
      zoom = 50
    } else {
      zoom = parseFloat(zoom)
      zoom = Math.max(0, Math.min(100, zoom))
    }
    return Math.round((zoom * maxZoomLevel) / 100)
  }

  const parseDefaultLocation = defaultLocation => {
    if (typeof defaultLocation !== "string") {
      return fallbackCoordinates
    }
    let defaultLocationParts = defaultLocation.split(",")
    if (defaultLocationParts.length !== 2) {
      return fallbackCoordinates
    }

    let parsedDefaultLatitude = parseFloat(defaultLocationParts[0].trim())
    let parsedDefaultLongitude = parseFloat(defaultLocationParts[1].trim())

    return isValidLatitude(parsedDefaultLatitude) === true &&
      isValidLongitude(parsedDefaultLongitude) === true
      ? [parsedDefaultLatitude, parsedDefaultLongitude]
      : fallbackCoordinates
  }

  const resetView = () => {
    if (!mapInstance) {
      return
    }
    if (mapMarkerGroup.getLayers().length) {
      mapInstance.setZoom(0)
      mapInstance.fitBounds(mapMarkerGroup.getBounds(), {
        paddingTopLeft: [0, 24],
      })
    } else {
      mapInstance.setView(defaultCoordinates, safeZoomLevel)
    }
  }

  const locationControlUpdated = (mapInstance, locationEnabled) => {
    if (typeof mapInstance !== "object") {
      return
    }
    if (locationEnabled) {
      locationControl.addTo(mapInstance)
    } else {
      mapInstance.removeControl(locationControl)
    }
  }

  const fullScreenControlUpdated = (mapInstance, fullScreenEnabled) => {
    if (typeof mapInstance !== "object") {
      return
    }
    if (fullScreenEnabled) {
      fullScreenControl.addTo(mapInstance)
    } else {
      mapInstance.removeControl(fullScreenControl)
    }
  }

  const zoomControlUpdated = (mapInstance, zoomEnabled) => {
    if (typeof mapInstance !== "object") {
      return
    }
    if (zoomEnabled) {
      zoomControl.addTo(mapInstance)
      mapInstance.scrollWheelZoom.enable()
    } else {
      mapInstance.removeControl(zoomControl)
      mapInstance.scrollWheelZoom.disable()
    }
  }

  const initMap = (tileURL, attribution, zoom) => {
    if (!mounted) {
      return
    }
    if (mapInstance) {
      mapInstance.remove()
    }

    if(showMap) {
      try {
        mapInstance = L.map(embeddedMapId, mapOptions)
        mapMarkerGroup.addTo(mapInstance)

        // Add attribution
        const cleanAttribution = sanitizeHtml(attribution, {
          allowedTags: ["a"],
          allowedAttributes: {
            a: ["href", "target"],
          },
        })
        L.tileLayer(tileURL, {
          attribution: "&copy; " + cleanAttribution,
          zoom,
        }).addTo(mapInstance)

        // Add click handler
        mapInstance.on("click", handleMapClick)
        if(latitude && longitude) {
          setLatLng(latitude, longitude)
        }

        // Reset view
        resetView()
      } catch (e) {
        console.error("There was a problem with the map", e)
      }
    }
  }

  const handleMapClick = e => {
    console.log(e)
    setLatLng(e.latlng.lat, e.latlng.lng)
  }
</script>

<label
  class:hidden={!label}
  for={longFieldState?.fieldId}
  class={`spectrum-FieldLabel spectrum-FieldLabel--sizeM spectrum-Form-itemLabel ${labelClass}`}
>
  {label || " "}
</label>

{#if showMap}
  <div class="embedded-map-wrapper map-default" use:styleable={$component.styles}>
    {#if error}
      <div>{error}</div>
    {/if}

    <div id={embeddedMapId} class="embedded embedded-map" />
  </div>
{:else}
  <div use:styleable={$component.styles}>
    {#if error}
      <div>{error}</div>
    {/if}
  </div>
{/if}

<div class="button-container">
  <div class="spectrum-Form-item" use:styleable={$component.styles}>
    {#if !formContext}
      <div class="placeholder">Form components need to be wrapped in a form</div>
    {:else}
      {#if !latitudeField || !longitudeField}
        <div class="error">Please select latitude and longitude fields</div>
      {/if}
      {#if error}
        <div class="error">{error}</div>
      {/if}
      {#if latFieldState?.error}
        <div class="error">{latFieldState.error}</div>
      {/if}
      {#if longFieldState?.error}
        <div class="error">{longFieldState.error}</div>
      {/if}


      {#if !disabled && showButton}
        <div class="spectrum-Form-itemField">
          <div class="container">
            <div class="actions">
              {#if !isLoading}
                <button
                  on:click={getLocation}
                  class="spectrum-Button spectrum-Button--fill spectrum-Button--sizeM spectrum-Button--primary"
                  >Get Location</button
                >
              {:else}
                <div class="spinner">
                  <div
                    class="spectrum-ProgressCircle spectrum-ProgressCircle--indeterminate spectrum-ProgressCircle--small"
                  >
                    <div class="spectrum-ProgressCircle-track" />
                    <div class="spectrum-ProgressCircle-fills">
                      <div class="spectrum-ProgressCircle-fillMask1">
                        <div class="spectrum-ProgressCircle-fillSubMask1">
                          <div class="spectrum-ProgressCircle-fill" />
                        </div>
                      </div>
                      <div class="spectrum-ProgressCircle-fillMask2">
                        <div class="spectrum-ProgressCircle-fillSubMask2">
                          <div class="spectrum-ProgressCircle-fill" />
                        </div>
                      </div>
                    </div>
                  </div>
                </div>
              {/if}
            </div>
            {#if showOutput}
              <div class="results">
                <div class="spectrum-FieldLabel">
                  {latitude || $initialLatStore}
                </div>
                <div class="spectrum-FieldLabel">
                  {longitude || $initialLongStore}
                </div>
              </div>
            {/if}
          </div>
        </div>
      {/if}
    {/if}
  </div>
</div>

<style>
  .container {
    display: flex;
    height: 42px;
    align-items: center;
  }
  .actions {
    width: 135px;
  }
  .spinner {
    display: flex;
    justify-content: center;
  }
  .placeholder {
    color: var(--spectrum-global-color-gray-600);
  }
  label {
    white-space: nowrap;
  }
  label.hidden {
    padding: 0;
  }
  .spectrum-Form-itemField {
    position: relative;
    width: 100%;
    align-items: center;
    gap: 8px;
  }
  .error {
    color: var(
      --spectrum-semantic-negative-color-default,
      var(--spectrum-global-color-red-500)
    );
    font-size: var(--spectrum-global-dimension-font-size-75);
    margin-top: var(--spectrum-global-dimension-size-75);
  }
  .spectrum-FieldLabel--right,
  .spectrum-FieldLabel--left {
    padding-right: var(--spectrum-global-dimension-size-200);
  }

  .embedded-map-wrapper {
    background-color: #f1f1f1;
    height: 320px;
  }
  .map-default {
    min-height: 180px;
    min-width: 200px;
  }
  .embedded-map :global(a.map-svg-button) {
    display: flex;
    justify-content: center;
    align-items: center;
  }
  .embedded-map :global(.leaflet-top),
  .embedded-map :global(.leaflet-bottom) {
    z-index: 998;
  }
  .embedded-map :global(.embedded-map-marker) {
    color: #ee3b35;
  }
  .embedded-map :global(.embedded-map-marker--candidate) {
    color: var(--primaryColor);
  }
  .embedded-map :global(.embedded-map-control) {
    font-size: 22px;
  }
  .embedded-map {
    height: 100%;
    width: 100%;
  }
  .button-container {
    display: flex;
    flex-direction: row;
    justify-content: center;
    align-items: center;
    gap: var(--spacing-xl);
    margin-top: var(--spacing-xl);
  }
</style>
