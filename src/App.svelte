<script>
  /* eslint-env browser */
  const OBS_WEBSOCKET_LATEST_VERSION = '5.0.1' // https://api.github.com/repos/Palakis/obs-websocket/releases/latest

  // Imports
  import { onMount } from 'svelte'
  import {
    mdiFullscreen,
    mdiFullscreenExit,
    mdiConnection
  } from '@mdi/js'
  import Icon from 'mdi-svelte'
  import { compareVersions } from 'compare-versions'

  import './style.scss'
  import { obs, sendCommand } from './obs.js'
  import ProgramPreview from './ProgramPreview.svelte'
  import OverlaySidebar from './OverlaySidebar.svelte'
  import OverlayPreviews from './OverlayPreviews.svelte'

  onMount(async () => {
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.register('/service-worker.js')
    }

    // Request screen wakelock
    if ('wakeLock' in navigator) {
      try {
        await navigator.wakeLock.request('screen')
        // Re-request when coming back
        document.addEventListener('visibilitychange', async () => {
          if (document.visibilityState === 'visible') {
            await navigator.wakeLock.request('screen')
          }
        })
      } catch (e) {}
    }

    // Toggle the navigation hamburger menu on mobile
    const navbar = document.querySelector('.navbar-burger')
    navbar.addEventListener('click', () => {
      navbar.classList.toggle('is-active')
      document
        .getElementById(navbar.dataset.target)
        .classList.toggle('is-active')
    })

    // Listen for fullscreen changes
    document.addEventListener('fullscreenchange', () => {
      isFullScreen = document.fullscreenElement
    })

    document.addEventListener('webkitfullscreenchange', () => {
      isFullScreen = document.webkitFullscreenElement
    })

    document.addEventListener('msfullscreenchange', () => {
      isFullScreen = document.msFullscreenElement
    })

    if (document.location.hash !== '') {
      // Read address from hash
      address = document.location.hash.slice(1)

      // This allows you to add a password in the URL like this:
      // http://obs-web.niek.tv/#ws://localhost:4455#password
      if (address.includes('#')) {
        [address, password, displayName] = address.split('#')
      }
      await connect()
    }

    // Export the sendCommand() function to the window object
    window.sendCommand = sendCommand
  })

  // State
  let connected
  let heartbeatInterval
  let isFullScreen
  let isIconMode = window.localStorage.getItem('isIconMode') || false
  let address
  let password
  let errorMessage = ''
  let imageFormat = 'jpg'
  let displayName = ''
  let connectionPreset = ''
  
  // Overlay System Related
  const overlaySystemUrl = 'https://tatooine-e1db8.web.app/'
  const overlaySystemControllerPrefix = '/controller'
  const overlaySystemPreviewPrefix = '/presetSync/'
  let overlayId

  $: [displayName, address, password] = connectionPreset.split(' ')

  $: isIconMode
    ? window.localStorage.setItem('isIconMode', 'true')
    : window.localStorage.removeItem('isIconMode')

  function toggleFullScreen () {
    if (isFullScreen) {
      if (document.exitFullscreen) {
        document.exitFullscreen()
      } else if (document.webkitExitFullscreen) {
        document.webkitExitFullscreen()
      } else if (document.msExitFullscreen) {
        document.msExitFullscreen()
      }
    } else {
      if (document.documentElement.requestFullscreen) {
        document.documentElement.requestFullscreen()
      } else if (document.documentElement.webkitRequestFullscreen) {
        document.documentElement.webkitRequestFullscreen()
      } else if (document.documentElement.msRequestFullscreen) {
        document.documentElement.msRequestFullscreen()
      }
    }
  }

  async function connect () {
    address = address || 'ws://localhost:4455'
    if (address.indexOf('://') === -1) {
      const secure = location.protocol === 'https:' || address.endsWith(':443')
      address = secure ? 'wss://' : 'ws://' + address
    }
    console.log('Connecting to:', address, '- using password:', password)

    await disconnect()
    try {
      const { obsWebSocketVersion, negotiatedRpcVersion } = await obs.connect(
        address,
        password,
        // {
        //   eventSubscriptions: eventSubscription.InputVolumeMeters,
        //   rpcVersion: 1
        // }
      )
      console.log(
        `Connected to obs-websocket version ${obsWebSocketVersion} (using RPC ${negotiatedRpcVersion})`
      )
    } catch (e) {
      console.log(e)
      errorMessage = e.message
    }
  }

  async function disconnect () {
    await obs.disconnect()
    clearInterval(heartbeatInterval)
    connected = false
    errorMessage = '切断された'
  }

  function extractIdFromURL(url) {
    return url.substring(overlaySystemUrl.length);
  }
  
  // OBS events
  obs.on('ConnectionClosed', () => {
    connected = false
    window.history.pushState(
      '',
      document.title,
      window.location.pathname + window.location.search
    ) // Remove the hash
    console.log('Connection closed')
  })

  obs.on('Identified', async () => {
    console.log('Connected')
    connected = true
    document.location.hash = address + "#" + password + "#" + displayName // For easy bookmarking
    let data = await sendCommand('GetVersion')
    const version = data.obsWebSocketVersion || ''
    console.log('OBS-websocket version:', version)
    if (compareVersions(version, OBS_WEBSOCKET_LATEST_VERSION) < 0) {
      alert(
        'You are running an outdated OBS-websocket (version ' +
          version +
          '), please upgrade to the latest version for full compatibility.'
      )
    }
    if (
      data.supportedImageFormats.includes('webp') &&
      document
        .createElement('canvas')
        .toDataURL('image/webp')
        .indexOf('data:image/webp') === 0
    ) {
      imageFormat = 'webp'
    }

    data = await sendCommand('GetInputSettings', {
      inputName: 'Overlay',
    });
    overlayId = extractIdFromURL(data.inputSettings.url);
  })

  obs.on('ConnectionError', async () => {
    errorMessage = 'Please enter your password:'
    document.getElementById('password').focus()
    if (!password) {
      connected = false
    } else {
      await connect()
    }
  })

  // Refresh when overlay ID is changed.
  obs.on('InputSettingsChanged', async (data) => {
    console.log('InputSettingsChanged', data)
    if(data.inputName === 'Overlay') {
      alert("テロップIDを更新されました！")
      location.reload();
    }
  })

</script>

<svelte:head>
  <title>Avalon Director リモートコントロール</title>
</svelte:head>

<nav class="navbar is-dark mb-2" aria-label="main navigation">
  <div class="navbar-brand">

    <!-- svelte-ignore a11y-missing-attribute -->
    <button
      class="navbar-burger burger"
      aria-label="menu"
      aria-expanded="false"
      data-target="navmenu"
    >
      <span aria-hidden="true" />
      <span aria-hidden="true" />
      <span aria-hidden="true" />
    </button>
  </div>

  <div id="navmenu" class="navbar-menu">
    <div class="navbar-start">
      {#if connected}
        <div class="navbar-item">
          <div class="buttons">
            <!-- svelte-ignore a11y-missing-attribute -->
            <button
              class:is-light={!isFullScreen}
              class="button is-link mr-3 mb-0"
              on:click={toggleFullScreen}
              title="Toggle Fullscreen"
              >
              <span class="icon">
                <Icon path={isFullScreen ? mdiFullscreenExit : mdiFullscreen} />
              </span>
            </button>

            <h2 class="title is-2 has-text-centered has-text-white">{displayName}</h2>
          </div>
        </div>
      {:else}
        <img class="logo" src="/u-next-logo.png" alt="U-NEXT logo" >
      {/if}
    </div>

    <div class="navbar-end">
      <div class="navbar-item">
        <h3 class="title is-3 has-text-centered has-text-white mb-0 mr-3">Avalon Director モード</h3>
        <div class="buttons">
        {#if connected}
        <button
              class="button is-danger is-light}"
              on:click={disconnect}
              title="Disconnect"
            >
              <span class="icon"><Icon path={mdiConnection} /></span>
            </button>
        {/if}
        </div>
      </div>
    </div>
  </div>
</nav>

<section class="section has-background-black-ter">
  <div class="container is-fullhd">
    {#if connected}
    <div class="columns">
      <div class="column is-two-fifths">
        <p class="has-text-centered has-text-white">制御中テロップID</p>
        <p class="title is-3 has-text-centered has-text-white">{overlayId}</p>
        
        {#if overlayId}
        <OverlaySidebar
          overlaySystemUrl = {overlaySystemUrl}
          overlaySystemControllerPrefix = {overlaySystemControllerPrefix}
          overlayId = {overlayId} />
        {/if}
      </div>

      <div class="column">
        <div class="columns">
          <div class="column is-four-fifths">
            <ProgramPreview {imageFormat} />
          </div>

          <div class="column">
            MACRO
          </div>
        </div>

        {#if overlayId}
        <OverlayPreviews
          overlaySystemUrl = {overlaySystemUrl}
          overlaySystemPreviewPrefix = {overlaySystemPreviewPrefix}
          overlayId = {overlayId} />
        {/if}
      </div>
    </div>
    {:else}
      <p class="title is-3 mt-3 has-text-centered has-text-white">Avalon に接続する</p>
      <form on:submit|preventDefault={connect}>
        <p class="has-text-white">プリセット</p>
        <div class="field is-grouped">
          <div class="select">
            <select bind:value={connectionPreset}>
              <option selected>501-PRIMAR ws://Avalon-501:4455Y  </option>
              <option>501-BACKUP ws://Avalon-502:4455  </option>
              <option>502-PRIMARY ws://Avalon-503:4455  </option>
              <option>502-BACKUP ws://Avalon-504:4455  </option>
              <option>DEV ws://10.231.102.227:4455  </option>
              <option>LOCAL ws://localhost:4455  </option>
            </select>
          </div>
        </div>
        <p class="has-text-white">接続情報：</p>
        <div class="field is-grouped">
          <p class="control is-expanded">
            <input
              id="host"
              bind:value={address}
              class="input mb-3"
              type="text"
              autocomplete=""
              placeholder="ws://IPアドレス:4455"
            />
            <input
              id="password"
              bind:value={password}
              class="input mb-3"
              type="password"
              autocomplete="current-password"
              placeholder="パスワード (認証を無効にしている場合は空のままにしてください)"
            />
            <input
              id="displayName"
              bind:value={displayName}
              class="input mb-3"
              type="text"
              autocomplete=""
              placeholder="表示名"
            />
          </p>
          <p>
            <button class="button is-success {address ? '':'is-locked'}">接続</button>
          </p>
        </div>
      </form>
    {/if}
  </div>
</section>
