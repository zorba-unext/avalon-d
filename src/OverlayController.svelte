<script>
  import { onMount } from 'svelte'
  import { sendCommand } from './obs.js'

  const overlaySystemPrefix = 'https://tatooine-e1db8.web.app/'
  const overlaySystemControllerPrefix = '/controller'
  const overlaySystemPreviewPrefix = '/previewSync/'
  const overlaySystemCreatePrefix = 'create'
  let overlayId
  let overlayPreviewUrl
  let overlayControllerUrl
  let previewIdArray = [1,2,3,4,5,6,7,8,9,10,11,12]
  let currentPreviewId = 1
  export let isIdSettingMenuActive

  onMount(async () => {
    const data = await sendCommand('GetInputSettings', {
      inputName: 'Overlay',
    });
    let overlayUrl = data.inputSettings.url;
    overlayId = extractIdFromURL(overlayUrl);

    if (overlayId) {
      overlayControllerUrl = `${overlaySystemPrefix}${overlayId}${overlaySystemControllerPrefix}`
      overlayPreviewUrl = `${overlaySystemPrefix}${overlayId}${overlaySystemPreviewPrefix}${currentPreviewId}`
      isIdSettingMenuActive = false
    } else {
      isIdSettingMenuActive = true
    }
  });

  function extractIdFromURL(url) {
    if (!url.startsWith(overlaySystemPrefix)) {
      console.error(
        'URL does not start with the expected prefix:',
        overlaySystemPrefix,
      );
      return null;
    }
    return url.substring(overlaySystemPrefix.length);
  }

  async function setBrowserInputSetting() {
    try {
      await sendCommand('SetInputSettings', {
        inputName: 'Overlay',
        inputSettings: {
          url: `${overlaySystemPrefix}${overlayId}`,
        },
        overlay: true,
      });
      overlayControllerUrl = `${overlaySystemPrefix}${overlayId}${overlaySystemControllerPrefix}`
      overlayPreviewUrl = `${overlaySystemPrefix}${overlayId}${overlaySystemPreviewPrefix}${currentPreviewId}`
      toggleIdSettingMenu();
    } catch (error) {
      overlayControllerUrl = '';
      alert('エラー: スオーバーレイ設定の更新は失敗しました。');
    }
  }

  function toggleIdSettingMenu() {
    isIdSettingMenuActive = !isIdSettingMenuActive;
  }

  function setCurrentPreviewId(previewId){
    currentPreviewId = previewId;
    overlayPreviewUrl = `${overlaySystemPrefix}${overlayId}${overlaySystemPreviewPrefix}${currentPreviewId}`
    console.log(overlayPreviewUrl)
  }
</script>

{#if isIdSettingMenuActive}
<div class="columns pb-0">
  <div class="column is-three-fifths">
    <p class="mb-2">テロップ ID</p>
    <input
      class="input is-info"
      type="text"
      placeholder="テロップ ID"
      bind:value={overlayId}
    />
    <div class="buttons mt-4 is-right">
      <button class="button is-danger" on:click={setBrowserInputSetting}>適用</button>
    </div>
  </div>
  <div class="column">
    <iframe
    title="Overlay ID Create"
    src="{overlaySystemPrefix}{overlaySystemCreatePrefix}"
    width="500"
    height="345"
    style="overflow: auto; border: none;"
    frameborder="0"
    ></iframe>
  </div>
</div>
{/if}
{#if overlayControllerUrl}
  <hr class={isIdSettingMenuActive ? '' : 'is-hidden'} />
  <div class="box">
  <div class="columns pb-0">
    <div class="column is-three-fifths">
        <p class="has-text-centered has-text-dark">制御中テロップID</p>
        <p class="has-text-centered has-text-dark">{overlayId}</p>
    </div>
    <div class="column">
      <div class="buttons is-centered">
        <button class="button is-danger"
        on:click={toggleIdSettingMenu}>テロップID再設置・新規発行</button>
      </div>
    </div>
  </div>
  </div>
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
{/if}