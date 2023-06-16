<script>
  import { getContext, onDestroy } from "svelte";
  import { writable } from "svelte/store";

  export let disabled;
  export let label;

  export let latitudeField;
  export let longitudeField;
  export let latitudeValidation;
  export let longitudeValidation;

  const { styleable } = getContext("sdk");
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
          latitude = pos.coords.latitude;
          longitude = pos.coords.longitude;
          latFieldApi.setValue(latitude);
          longFieldApi.setValue(longitude);
          isLoading = false;
        },
        (err) => {
          error = `Error(${err.code}): ${err.message}`;
          isLoading = false;
        }
      );
    }
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

  onDestroy(() => {
    unsubscribeLatitude?.();
    unsubscribeLongitude?.();
  });
</script>

<div class="spectrum-Form-item" use:styleable={$component.styles}>
  {#if !formContext}
    <div class="placeholder">Form components need to be wrapped in a form</div>
  {:else}
    <label
      class:hidden={!label}
      for={longFieldState?.fieldId}
      class={`spectrum-FieldLabel spectrum-FieldLabel--sizeM spectrum-Form-itemLabel ${labelClass}`}
    >
      {label || " "}
    </label>
    <div class="spectrum-Form-itemField">
      <div class="container">
        {#if !disabled}
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
        {/if}

        <div class="results">
          <div class="spectrum-FieldLabel">
            {latitude || $initialLatStore}
          </div>
          <div class="spectrum-FieldLabel">
            {longitude || $initialLongStore}
          </div>
        </div>
      </div>

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
    </div>
  {/if}
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
</style>
