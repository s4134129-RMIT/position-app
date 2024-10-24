<script>
    export let showModal // boolean

    let dialog // HTMLDialogElement

    $: if (dialog && showModal) { dialog.showModal() }
</script>

<!-- svelte-ignore a11y-click-events-have-key-events -->

<!-- add to dialog to enable click to close
 on:click|self={() => dialog.close()}
 -->
<dialog
    class="modal"
    bind:this={dialog}
    on:close={() => (showModal = false)}
>
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <div
        class="modal-box"
        on:click|stopPropagation>
        <slot name="header" />
        <hr />
        <slot />
        <div class="modal-action">
            <form method="dialog">
                <!-- svelte-ignore a11y-autofocus -->
                <button
                    class="btn btn-success"
                    autofocus
                    on:click={() => dialog.close()}>close modal</button>
            </form>
        </div>
    </div>
</dialog>

<style>
  dialog::backdrop {
    background: rgba(0, 0, 0, 0.3);
  }
  dialog > div {
    padding: 1em;
  }
  dialog[open] {
    animation: zoom 0.3s cubic-bezier(0.34, 1.56, 0.64, 1);
  }
  @keyframes zoom {
    from {
      transform: scale(0.95);
    }
    to {
      transform: scale(1);
    }
  }
  dialog[open]::backdrop {
    animation: fade 0.2s ease-out;
  }
  @keyframes fade {
    from {
    opacity: 0;
    }
    to {
    opacity: 1;
    }
  }
  button {
    display: block;
  }
</style>
