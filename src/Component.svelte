<script>
  import { getContext, onDestroy } from "svelte";

  export let disabled;
  export let label;

  export let latitudeField;
  export let longitudeField;

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

  async function getLocation() {
    if (!navigator.geolocation) {
      error = "Geolocation is not supported by your browser";
    } else {
      isLoading = true;
      navigator.geolocation.getCurrentPosition(
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

  const formFieldLatitude = formApi?.registerField(
    latitudeField,
    "number",
    latitude,
    false,
    null,
    formStep
  );

  const formFieldLongitude = formApi?.registerField(
    longitudeField,
    "number",
    longitude,
    false,
    null,
    formStep
  );

  $: unsubscribeLatitude = formFieldLatitude?.subscribe((value) => {
    latFieldState = value?.fieldState;
    latFieldApi = value?.fieldApi;
  });

  $: unsubscribeLongitude = formFieldLongitude?.subscribe((value) => {
    longFieldState = value?.fieldState;
    longFieldApi = value?.fieldApi;
  });

  onDestroy(() => {
    unsubscribeLatitude();
    unsubscribeLongitude();
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
      {#if !disabled}
        <button
          on:click={getLocation}
          class="spectrum-Button spectrum-Button--fill spectrum-Button--sizeM spectrum-Button--primary"
          >Get Location</button
        >
      {/if}
      <p>{latitude}</p>
      <p>{longitude}</p>

      {#if isLoading}
        <span
          class="spectrum-Loader spectrum-Loader--sizeS"
          role="status"
          aria-label="Loading"
        />
      {/if}
      {#if !latitudeField || !longitudeField}
        <div class="error">Please select a field</div>
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
    /* display: flex; */
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
