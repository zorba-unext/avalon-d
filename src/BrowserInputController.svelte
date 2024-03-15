<script>
  import { onMount } from 'svelte';
  import { sendCommand } from './obs.js';
  import { mdiSubtitles } from '@mdi/js';
  import Icon from 'mdi-svelte';

  const overlaySystemPrefix = 'https://tatooine-e1db8.web.app/';
  const overlaySystemControllerPrefix = '/controller';
  const overlaySystemCreatePrefix = '/create';
  let showMenu = false;
  let overlayId = '';
  let currentTab = 'input';
  let overlayControllerUrl = '';
  $: toggleButtonPx = showMenu ? '570px' : '0';

  onMount(async () => {
    const data = await sendCommand('GetInputSettings', {
      inputName: 'Overlay',
    });
    let overlayUrl = data.inputSettings.url;
    overlayId = extractIdFromURL(overlayUrl);
  });

  function toggleShowMenu() {
    showMenu = !showMenu;
  }

  function setCurrentTab(tab) {
    currentTab = tab;
    console.log(currentTab);
  }

  function extractIdFromURL(url) {
    if (!url.startsWith(overlaySystemPrefix)) {
      console.error(
        'URL does not start with the expected prefix:',
        overlaySystemPrefix,
      );
      return 'INPUT ERROR';
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
      overlayControllerUrl = `${overlaySystemPrefix}${overlayId}${overlaySystemControllerPrefix}`;
      alert('Browser settings have been successfully updated.');
    } catch (error) {
      overlayControllerUrl = '';
      alert('ERROR: Update streaming server setting failed.');
    }
    showMenu = false;
  }
</script>

<div class="menu-container {showMenu ? 'is-active' : ''}">
  <button
    class="button is-info is-medium toggle-button"
    style="right: {toggleButtonPx};"
    on:click={toggleShowMenu}
  >
    <span class="icon"><Icon path={mdiSubtitles} /></span>
  </button>

  <div class="slide-menu">
    <div class="dropdown-content">
      <div class="tabs is-medium is-boxed is-centered">
        <ul>
          <li class={currentTab === 'input' ? 'is-active' : ''}>
            <a on:click={() => setCurrentTab('input')}>ID入力</a>
          </li>
          <li class={currentTab === 'create' ? 'is-active' : ''}>
            <a on:click={() => setCurrentTab('create')}>新規発行</a>
          </li>
        </ul>
      </div>
      <div class="dropdown-item">
        {#if currentTab === 'input'}
          <p class="mb-2">Overlay ID</p>
          <input
            class="input is-info"
            type="text"
            placeholder="Overlay ID"
            bind:value={overlayId}
          />
          <div class="buttons mt-4 is-right">
            <a class="button is-danger" on:click={setBrowserInputSetting}>適用</a>
          </div>
        {:else}
          <iframe
            title="Overlay ID Create"
            src="{overlaySystemPrefix}/create"
            width="500"
            height="330"
            style="overflow: auto; border: none;"
            frameborder="0"
          ></iframe>
        {/if}
      </div>
      {#if overlayControllerUrl}
        <hr class="dropdown-divider" />
        <div class="dropdown-item">
          <iframe
            title="Overlay Control"
            src={overlayControllerUrl}
            width="500"
            height="660"
            frameborder="0"></iframe>
          <p class="title is-7 has-text-centered">{overlayControllerUrl}</p>
        </div>
      {/if}
    </div>
  </div>
</div>

<style>
  .menu-container {
    position: relative;
  }

  .toggle-button {
    position: fixed;
    right: 0; /* This will be overridden by inline styles */
    top: 20%;
    z-index: 100;
    transform: translateY(-50%);
    transition: right 0.5s; /* Smooth transition for moving the button */
  }

  .slide-menu {
    height: 100%;
    width: 0;
    position: fixed;
    z-index: 100;
    top: 0;
    right: 0;
    overflow-x: hidden;
    transition: 0.5s;
    padding-top: 3rem;
  }

  .menu-container.is-active .slide-menu {
    width: 580px;
  }

  .dropdown-content {
    margin: 25px;
  }

  .dropdown-item:not(:last-child) {
    margin-bottom: 1rem;
  }
</style>
