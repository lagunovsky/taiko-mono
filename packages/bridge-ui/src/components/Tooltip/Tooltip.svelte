<script lang="ts">
  import { onDestroy, onMount } from 'svelte';

  import { Icon } from '$components/Icon';
  import { classNames } from '$libs/util/classNames';
  import { positionElementByTarget } from '$libs/util/positionElementByTarget';

  export let position: Position = 'top';
  export let tooltipOpen = false;

  let tooltipId = `tooltip-${crypto.randomUUID()}`;
  let tooltipClass = `block dialog-tooltip`;
  let classes = classNames('flex z-10 ', $$props.class || 'relative');

  const GAP = 10; // distance between trigger element and tooltip
  let triggerElem: HTMLButtonElement;
  let dialogElem: HTMLDialogElement;

  function closeTooltip(ms = 0) {
    setTimeout(() => {
      tooltipOpen = false;
    }, ms);
  }

  function openTooltip() {
    tooltipOpen = true;
  }

  onMount(() => {
    if (position === 'top') {
      tooltipClass = `block dialog-tooltip dialog-tooltip-top`;
    }
    positionElementByTarget(dialogElem, triggerElem, position, GAP);
  });

  onDestroy(() => {
    closeTooltip();
  });
</script>

<div class={classes} role="presentation" on:mouseleave={() => closeTooltip(200)}>
  <button
    aria-haspopup="dialog"
    aria-controls={tooltipId}
    aria-expanded={tooltipOpen}
    on:click|stopPropagation={openTooltip}
    on:focus|stopPropagation={openTooltip}
    on:mouseenter={openTooltip}
    bind:this={triggerElem}>
    <Icon type="question-circle" />
  </button>

  <dialog id={tooltipId} class={tooltipClass} class:block-hidden={!tooltipOpen} bind:this={dialogElem}>
    <slot />
  </dialog>
</div>
