<script>
  import { getContext, onDestroy, onMount } from "svelte";

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

  const { styleable } = getContext("sdk");
  const component = getContext("component");
  const formContext = getContext("form");
  const formStepContext = getContext("form-step");
  const fieldGroupContext = getContext("field-group");

  let error = "";
  let isLoading = false;
  let latFieldState;
  let latFieldApi;
  let longFieldState;
  let longFieldApi;
  let unsubscribeLatitude;
  let unsubscribeLongitude;
  let automodeTimeout;
  let isDestroyed = false;

  const formApi = formContext?.formApi;
  const labelPos = fieldGroupContext?.labelPosition || "above";
  $: formStep = formStepContext ? $formStepContext || 1 : 1;
  $: labelClass =
    labelPos === "above" ? "" : `spectrum-FieldLabel--${labelPos}`;
  $: labelFor =
    (latFieldState?.error && latFieldState?.fieldId) ||
    (longFieldState?.error && longFieldState?.fieldId) ||
    latFieldState?.fieldId ||
    longFieldState?.fieldId;

  const validationKey = (validation) => {
    if (!validation) {
      return "";
    }

    try {
      return JSON.stringify(validation);
    } catch (stringifyError) {
      return "";
    }
  };

  let latitudeRegistrationKey;
  let longitudeRegistrationKey;

  $: if (formApi && latitudeField) {
    const nextKey = `${latitudeField}|${formStep}|${
      disabled ? 1 : 0
    }|${validationKey(latitudeValidation)}`;
    if (nextKey !== latitudeRegistrationKey) {
      latitudeRegistrationKey = nextKey;
      unsubscribeLatitude?.();

      const formFieldLatitude = formApi.registerField(
        latitudeField,
        "number",
        null,
        disabled,
        false,
        latitudeValidation,
        formStep
      );

      unsubscribeLatitude = formFieldLatitude?.subscribe((value) => {
        latFieldState = value?.fieldState;
        latFieldApi = value?.fieldApi;
      });
    }
  } else if (latitudeRegistrationKey) {
    latitudeRegistrationKey = null;
    latFieldApi?.deregister();
    unsubscribeLatitude?.();
    latFieldState = null;
    latFieldApi = null;
  }

  $: if (formApi && longitudeField) {
    const nextKey = `${longitudeField}|${formStep}|${
      disabled ? 1 : 0
    }|${validationKey(longitudeValidation)}`;
    if (nextKey !== longitudeRegistrationKey) {
      longitudeRegistrationKey = nextKey;
      unsubscribeLongitude?.();

      const formFieldLongitude = formApi.registerField(
        longitudeField,
        "number",
        null,
        disabled,
        false,
        longitudeValidation,
        formStep
      );

      unsubscribeLongitude = formFieldLongitude?.subscribe((value) => {
        longFieldState = value?.fieldState;
        longFieldApi = value?.fieldApi;
      });
    }
  } else if (longitudeRegistrationKey) {
    longitudeRegistrationKey = null;
    longFieldApi?.deregister();
    unsubscribeLongitude?.();
    longFieldState = null;
    longFieldApi = null;
  }

  function getLocation() {
    if (disabled || isLoading) {
      return;
    }

    error = "";

    if (!latFieldApi || !longFieldApi) {
      return;
    }

    if (typeof navigator === "undefined" || !navigator.geolocation) {
      error = "Geolocation is not supported by your browser";
      return;
    }

    isLoading = true;
    navigator.geolocation.getCurrentPosition(
      (pos) => {
        if (isDestroyed) {
          return;
        }

        const { latitude, longitude } = pos.coords;
        latFieldApi?.setValue(latitude);
        longFieldApi?.setValue(longitude);
        isLoading = false;
      },
      (err) => {
        if (isDestroyed) {
          return;
        }

        error = `Error(${err.code}): ${err.message}`;
        isLoading = false;
      }
    );
  }

  onMount(() => {
    if (automode && !disabled) {
      if (automodeDelay) {
        automodeTimeout = setTimeout(getLocation, automodeDelay * 1000);
      } else {
        getLocation();
      }
    }
  });

  onDestroy(() => {
    isDestroyed = true;
    if (automodeTimeout) {
      clearTimeout(automodeTimeout);
      automodeTimeout = null;
    }
    latFieldApi?.deregister();
    longFieldApi?.deregister();
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
      for={labelFor}
      class={`spectrum-FieldLabel spectrum-FieldLabel--sizeM spectrum-Form-itemLabel ${labelClass}`}
    >
      {label || " "}
    </label>
    <div class="spectrum-Form-itemField">
      <div class="container">
        {#if !disabled && showButton}
          <div class="actions">
            {#if !isLoading}
              <button
                type="button"
                on:click={getLocation}
                class="spectrum-Button spectrum-Button--fill spectrum-Button--sizeM spectrum-Button--primary"
                >Get location</button
              >
            {:else}
              <div class="spinner">
                <div
                  class="spectrum-ProgressCircle spectrum-ProgressCircle--indeterminate spectrum-ProgressCircle--small"
                >
                  <div class="spectrum-ProgressCircle-track"></div>
                  <div class="spectrum-ProgressCircle-fills">
                    <div class="spectrum-ProgressCircle-fillMask1">
                      <div class="spectrum-ProgressCircle-fillSubMask1">
                        <div class="spectrum-ProgressCircle-fill"></div>
                      </div>
                    </div>
                    <div class="spectrum-ProgressCircle-fillMask2">
                      <div class="spectrum-ProgressCircle-fillSubMask2">
                        <div class="spectrum-ProgressCircle-fill"></div>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            {/if}
          </div>
        {/if}
        {#if showOutput}
          <div class="results">
            <div class="spectrum-FieldLabel">
              {latFieldState?.value ?? ""}
            </div>
            <div class="spectrum-FieldLabel">
              {longFieldState?.value ?? ""}
            </div>
          </div>
        {/if}
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
