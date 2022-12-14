<script lang="ts">
  import Icon from "$components/Icon.svelte";
  import { createEventDispatcher, tick } from "svelte";
  import { tweened } from "svelte/motion";
  import { sineInOut } from "svelte/easing";
  import { fade } from "svelte/transition";

  export let headerContent: String;
  export let bodyContent: String;

  const dispatch = createEventDispatcher();
  const iconSize = tweened(100, {
    duration: 100,
    easing: sineInOut,
  });

  enum Position {
    Above,
    Below,
    NotDragging,
    Within,
  }

  let self: HTMLElement;
  let isDrag: boolean = false;
  let dragStart: { x: number; y: number } = { x: 0, y: 0 };
  let mousePos: Position = Position.NotDragging;
  let iconName: string | undefined = undefined;

  $: if (!isDrag) {
    iconName = "menu";
  } else if (mousePos === Position.Above) {
    iconName = "arrow-up";
  } else if (mousePos === Position.Below) {
    iconName = "arrow-down";
  } else if (mousePos === Position.Within) {
    iconName = "minus";
  } else {
    throw new Error("Unreachable Code");
  }

  function handleClose(event: Event) {
    dispatch("close", event);
  }

  function handleDragStart(event: PointerEvent) {
    isDrag = true;
    dragStart = { x: event.clientX, y: event.clientY };
    mousePos = getPostion(event);
    self.setPointerCapture(event.pointerId);
    dispatch("dragStart", { x: event.x, y: event.y })
  }

  function handleDragMove(event: MouseEvent) {
    if (!isDrag) return
    mousePos = getPostion(event);
    const dragEvent: DragMove = {
      start: {
        x: dragStart.x,
        y: dragStart.y,
      },
      current: {
        x: event.clientX,
        y: event.clientY,
      },
    };
    dispatch("dragMove", dragEvent)
  }

  async function handleDragEnd(event: PointerEvent) {
    isDrag = false;
    const dragEvent: DragFinal = {
      start: {
        x: dragStart.x,
        y: dragStart.y,
      },
      end: {
        x: event.clientX,
        y: event.clientY,
      },
    };
    dispatch("dragFinal", dragEvent);
    self.releasePointerCapture(event.pointerId);
    dragStart = { x: 0, y: 0 };
  }

  function handleDragCancel(_event: Event) {
    dispatch("dragCancel");
    dragStart = { x: 0, y: 0 };
  }

  function getPostion(event: MouseEvent) {
    if (!isDrag) return Position.NotDragging;
    const top = self.getClientRects()[0].top;
    const bottom = self.getClientRects()[0].bottom;
    // 0 is top of page
    if (event.clientY < top) return Position.Above;
    else if (event.clientY >= bottom) return Position.Below;
    else return Position.Within;
  }
</script>

<template lang="pug">
    //- Prevent default mousedown propagation so that there isn't unsightly text
    //- highlighting as the user drags
    .card.card-compact.bg-base-100.w-full.ease-in-out.duration-200.transition-transform.touch-none.select-none(
      "class"!="{ isDrag ? 'drop-shadow-lg -translate-x-4 -translate-y-1' : 'drop-shadow' }"
      "bind:this={self}"
      "on:pointerdown"!="{ handleDragStart }"
      "on:pointerup"!="{ handleDragEnd }"
      "on:pointermove"!="{ handleDragMove }"
      "on:click"!="{ handleDragCancel }"
    )
      .card-body.flex-row.items-center
        //- This must have 'pointer-events' set to 'none'! Otherwise dragging on the menu icon causes problems on mobile (Firefox)
        .place-self-center.text-stone-700.ml-1.text-lg.pointer-events-none
          +key('iconName')
            #drag-icon("in:fade"!="{ { duration: 100, easing: sineInOut } }")
              Icon(name="{ iconName }")
        .divider.divider-horizontal.ml-1(class="before:bg-stone-300 after:bg-stone-300")
        //- Stop propagation to let user select text
        .grow
          h5.card-title { headerContent }
          p { bodyContent }
        //- Stop propagation to avoid being able to drag with the exit button
        .btn.btn-circle.shadow-sm.btn-sm.bg-stone-100.border-stone-300.text-stone-600.justify-self-end.duration-75.text-xl(
          class="hover:scale-125 hover:bg-stone-400 hover:border-stone-400 hover:text-stone-50 hover:shadow"
          "on:pointerdown|stopPropagation"
          "on:click"!="{ handleClose }"
        )
          Icon(name="x")
</template>

<style lang="scss">
  #drag-icon {
    position: relative;

    &::before {
      content: "";
      position: absolute;
      inset: 0 0 0 0;
      height: 100%;
      width: 100%;
      z-index: 1;
    }
  }
</style>