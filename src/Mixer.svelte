<script>
  import { onMount, onDestroy } from 'svelte';
  import { obs, sendCommand } from './obs.js';

  onMount(async () => {
    const data = await sendCommand('GetProfileList')
    console.log(data)

    sendCommand('GetInputList').then((data) => {
      // console.log('Mixer GetInputList', data);
      for (let i = 0; i < data.inputs.length; i++) {
        if (data.inputs[i].inputName.endsWith('@ah')) {
          // hide if input has @ah suffix
          continue;
        }
        sendCommand('GetInputVolume', {
          inputName: data.inputs[i].inputName,
        }).then((vol) => {
          // console.log('Mixer GetInputVolume', vol);
          if ( "inputVolumeDb" in vol) {;
            inputs[data.inputs[i].inputName] = {
              ...data.inputs[i],
              ...vol,
            };
          }
        });
      }
    });
  });

  onDestroy(() => {});

  let inputs = {};

  const audioButtons = ['+ 6', '+ 3', '= 0', '- 3', '- 6']

  obs.on('StudioModeStateChanged', async (data) => {
    console.log('Mixer StudioModeStateChanged', data.studioModeEnabled);
  });

  obs.on('CurrentPreviewSceneChanged', async (data) => {
    console.log('Mixer CurrentPreviewSceneChanged', data.sceneName);
  });

  obs.on('CurrentProgramSceneChanged', async (data) => {
    console.log('Mixer CurrentProgramSceneChanged', data.sceneName);
  });

  obs.on('InputVolumeChanged', async (data) => {
    // console.log('Mixer InputVolumeChanged', data)
    if (inputs[data.inputName]) {
      inputs[data.inputName] = { ...inputs[data.inputName], ...data };
    }
  });

  // obs.on('InputVolumeMeters', async (data) => {
  //   // console.log('Mixer InputVolumeChanged', data.inputs)
  //   inputMeter = data.inputs
  //   if (inputMeter && inputMeter.length > 0) {
  //     tempval = Math.round(inputMeter[0].inputLevelsMul[0][2] * 100)
  //   }
  // })

  // obs.on('SceneNameChanged', async (data) => {
  //   if (data.oldSceneName === programScene) programScene = data.sceneName
  //   if (data.oldSceneName === previewScene) previewScene = data.sceneName
  // })

  function updateVolume(e) {
    // console.log('updateVolume', e.target.name, e.target.value)
    sendCommand('SetInputVolume', {
      inputName: e.target.name,
      inputVolumeDb: parseFloat(e.target.value),
    });
  }

  function updateVolumeDelta(inputName, db) {
    const newVolume = db === 0 ? 0 : inputs[inputName].inputVolumeDb + db;
    sendCommand('SetInputVolume', {
      inputName: inputName,
      inputVolumeDb: parseFloat(newVolume),
    });
  }
</script>


<div class="mixer">
  <div class="audio-level-buttons px-6 my-5">
    {#each audioButtons as audioButton}
    <button class="button is-warning has-text-dark"
      on:click={async () => {
        await sendCommand('CallVendorRequest', {
          'vendorName': 'AdvancedSceneSwitcher',
          "requestType": "AdvancedSceneSwitcherMessage",
          "requestData": 
          {"message": `switch-is`}
        })

        await sendCommand('CallVendorRequest', {
          'vendorName': 'AdvancedSceneSwitcher',
          "requestType": "AdvancedSceneSwitcherMessage",
          "requestData": 
          {"message": `is-audio ${audioButton}`}
        })
      }}
      >IS {audioButton}</button>
    {/each}
  </div>

  <div class="audio-level-buttons px-6">
    {#each audioButtons as audioButton}
    <button class="button is-info has-text-white"
      on:click={async () => {
        await sendCommand('CallVendorRequest', {
          'vendorName': 'AdvancedSceneSwitcher',
          "requestType": "AdvancedSceneSwitcherMessage",
          "requestData": 
          {"message": `switch-pg`}
        })

        await sendCommand('CallVendorRequest', {
          'vendorName': 'AdvancedSceneSwitcher',
          "requestType": "AdvancedSceneSwitcherMessage",
          "requestData": 
            {"message": `pg-audio ${audioButton}`}
        })
      }}
      >PG {audioButton}</button>
    {/each}
  </div>
  
  <ol class="mt-5">
    {#if inputs && Object.keys(inputs).length > 0}
      {#each Object.keys(inputs).sort() as iname}
        <li class="box is-marginless has-background-dark">
          <div class="is-relative">
            <span class="tag is-dark is-small mixer-label"
              >{inputs[iname].inputName}</span
            >
            <input
              orient="vertical"
              class="slider mixer-slider"
              step="0.1"
              min="-60"
              max="12"
              value={inputs[iname].inputVolumeDb}
              type="range"
              on:input={updateVolume}
              name={inputs[iname].inputName}
            />
            <div class="buttons are-small mixer-buttons">
              <button
                class="button is-white is-outlined"
                on:click={() => updateVolumeDelta(inputs[iname].inputName, 1)}
                >+1</button
              >
              <button
                class="button is-white is-outlined"
                on:click={() => updateVolumeDelta(inputs[iname].inputName, 0)}
                >=0</button
              >
              <button
                class="button is-white is-outlined"
                on:click={() => updateVolumeDelta(inputs[iname].inputName, -1)}
                >-1</button
              >
            </div>
          </div>
          <span
            class="tag is-info is-small is-marginless is-centered has-background-dark mixer-value"
            >{typeof inputs[iname].inputVolumeDb === 'number'
              ? inputs[iname].inputVolumeDb.toFixed(1)
              : inputs[iname].inputVolumeDb} dB
          </span>
        </li>
      {/each}
    {/if}
  </ol>
</div>

<style>
.audio-level-buttons {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
}

.button {
  width: 6rem;
  justify-content: center;
  align-items: center;
  text-align: center;
}


  ol {
    list-style: None;
    display: flex;
    flex-wrap: wrap;
    justify-content: space-between;
    gap: 0.5rem;
    row-gap: 0.5rem;
  }
  .mixer-slider {
    width: 4rem;
    height: 10rem;
  }
  /* patch bulma-slider bug on chrome */
  input[type='range'].slider[orient='vertical']::-webkit-slider-thumb {
    position: relative;
    left: 0.25rem;
  }
  .mixer-value {
    width: 4rem;
  }
  .mixer-buttons {
    position: absolute;
    top: 0;
    right: 0rem;
    width: 1rem;
  }
  .mixer-label {
    position: absolute;
    transform-origin: 0 0;
    left: 0.5rem;
    transform: rotate(90deg);
  }
</style>
