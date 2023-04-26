<script>
  import { getContext, onDestroy } from "svelte";

  export let disabled;
  export let field;
  export let label;

  const { styleable } = getContext("sdk");
  const component = getContext("component");
  const formContext = getContext("form");
  const formStepContext = getContext("form-step");
  const fieldGroupContext = getContext("field-group");

  let position = "";
  let fieldState;
  let fieldApi;

  async function getLocation() {
    if (!navigator.geolocation) {
      alert("Geolocation is not supported by your browser");
    } else {
      navigator.geolocation.getCurrentPosition(
        (pos) => {
          position = `Latitude: ${pos.coords.latitude}, Longitude: ${pos.coords.longitude}`;
          fieldApi?.setValue(position);
        },
        (err) => {
          alert(`Error(${err.code}): ${err.message}`);
        }
      );
    }
  }

  const formApi = formContext?.formApi;
  const labelPos = fieldGroupContext?.labelPosition || "above";
  $: formStep = formStepContext ? $formStepContext || 1 : 1;
  $: formField = formApi?.registerField(
    field,
    "text",
    position,
    false,
    null,
    formStep
  );

  $: unsubscribe = formField?.subscribe((value) => {
    fieldState = value?.fieldState;
    fieldApi = value?.fieldApi;
  });

  $: labelClass =
    labelPos === "above" ? "" : `spectrum-FieldLabel--${labelPos}`;

  onDestroy(() => {
    fieldApi?.deregister();
    unsubscribe?.();
  });
</script>

<div class="spectrum-Form-item" use:styleable={$component.styles}>
  {#if !formContext}
    <div class="placeholder">Form components need to be wrapped in a form</div>
  {:else}
    <label
      class:hidden={!label}
      for={fieldState?.fieldId}
      class={`spectrum-FieldLabel spectrum-FieldLabel--sizeM spectrum-Form-itemLabel ${labelClass}`}
    >
      {label || " "}
    </label>
    <div class="spectrum-Form-itemField">
      <input type="text" value={position} readonly />
      <button on:click={getLocation} {disabled}>Get Geolocation</button>
    </div>
    {#if !field}
      <div class="error">Please select a field</div>
    {/if}
    {#if fieldState?.error}
      <div class="error">{fieldState.error}</div>
    {/if}
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
