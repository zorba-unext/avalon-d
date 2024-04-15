<script>
  import { onMount } from 'svelte'
  import IdSetupMenu from './IdSetupMenu.svelte';

  const overlaySystemPrefix = 'https://tatooine-e1db8.web.app/'
  const overlaySystemControllerPrefix = '/controller'
  const overlaySystemPreviewPrefix = '/previewSync/'
  export let overlayId
  let overlayPreviewUrl
  let overlayControllerUrl
  let previewIdArray = [1,2,3,4,5,6,7,8,9,10,11,12]
  let currentPreviewId = 1
  let isOverlayIdActive

  onMount(async () => {
    if (overlayId) {
      overlayControllerUrl = `${overlaySystemPrefix}${overlayId}${overlaySystemControllerPrefix}`
      overlayPreviewUrl = `${overlaySystemPrefix}${overlayId}${overlaySystemPreviewPrefix}${currentPreviewId}`
      isOverlayIdActive = true
    } else {
      isOverlayIdActive = false
    }
  });

  function setCurrentPreviewId(previewId){
    currentPreviewId = previewId;
    overlayPreviewUrl = `${overlaySystemPrefix}${overlayId}${overlaySystemPreviewPrefix}${currentPreviewId}`
    console.log(overlayPreviewUrl)
  }
</script>

{#if isOverlayIdActive}
  <div class="columns pb-0">
    <div class="column is-three-fifths">
      <div class="buttons is-left">
        {#each previewIdArray as previewId}
          <button class="button is-info"
          on:click={() => setCurrentPreviewId(previewId)}>プレビュー {previewId}</button>
          {/each}
      </div>

      <iframe
        title="Overlay Preview"
        src={overlayPreviewUrl}
        width="1920"
        height="1080"
        style="transform: scale({780 / 1920}, {400 / 1080}); transform-origin: top left; border: 1rem solid;"
      ></iframe>
    </div>

    <div class="column">
      <iframe
        title="Overlay Control"
        src={overlayControllerUrl}
        width="500"
        height="1400"></iframe>
    </div>
  </div>
{:else}
  <IdSetupMenu overlayId = {overlayId}/>
{/if}