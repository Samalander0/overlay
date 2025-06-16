<script>
  import { onMount } from 'svelte';
  import '$lib/main.scss';
  import { IconDownload, IconFileUpload, IconSettings } from "@tabler/icons-svelte";

  let video_element,
      canvas_element,
      dialog_element;
  let photo_data;

  let devices = [];
  let selected_device_id = null;

  async function startCamera(device_id) {
    // Stop existing stream if any
    if (video_element?.srcObject) {
      (video_element.srcObject)
        .getTracks()
        .forEach((track) => track.stop());
    }

    try {
      const constraints = {
        video: device_id
          ? { device_id: { exact: device_id } }
          : { device_id: { ideal: 'environment' } },
        audio: false
      };

      const stream = await navigator.mediaDevices.getUserMedia(constraints);
      video_element.srcObject = stream;
      await video_element.play();
    } catch (err) {
      console.error('Error accessing camera:', err);
    }
  }

  async function getDevices() {
    const allDevices = await navigator.mediaDevices.enumerateDevices();
    devices = allDevices.filter((d) => d.kind === 'videoinput');
  }

  async function takePhoto() {
    const context = canvas_element.getContext('2d');
    if (!context) return;
    context.drawImage(video_element, 0, 0, canvas_element.width, canvas_element.height);
    photo_data = canvas_element.toDataURL('image/png');
  };

  async function savePhoto() {
    if (!photo_data) return;
    const a = document.createElement('a');
    a.href = photo_data;
    a.download = 'photo.png';
    a.click();
  };

  let selected_file,
      image_url;

  function handleFileChange(event) {
    const input = event.target;
    if (input.files && input.files.length > 0) {
      selected_file = input.files[0];
      image_url = URL.createObjectURL(selected_file);
    }
  }

  // Auto-start camera when component mounts
  onMount(() => {
    startCamera();
    setTimeout(getDevices, 1000);
  });

  $: if (selected_device_id) {
    startCamera(selected_device_id);
  }
</script>

<main class="app">
  <div class="top-bar">
    <h1>Overlay</h1>
    <button class="settings-button" on:click={() => dialog_element.showModal()}>
      <IconSettings size="32" stroke="1.75"/>
    </button>
  </div>

  <dialog bind:this={dialog_element} class="settings-popup" closed>
    <h2>Settings</h2>
    <div class="camera-selector">
      <h3 on:click={getDevices}>Camera Select</h3>
      <div class="camera-list">
        {#each devices as device}
          <button on:click={() => (selected_device_id = device.deviceId)}>
            {device.label || 'Camera ' + device.deviceId}
          </button>
        {/each}
      </div>
    </div>
    <button autofocus on:click={() => dialog_element.close()} class="close">Close</button>
  </dialog>

  <div class="video-wrapper" style="--overlay-opacity: 0.1">
    <video bind:this={video_element} autoplay playsinline class="main-video"></video>
    {#if image_url}
      <img src={image_url} alt="Selected" class="overlay-image" />
    {/if}
  </div>

  <div class="app-bottom-bar">
    <button class="last-photo" on:click={savePhoto} disabled={!photo_data}>
      {#if photo_data}
        <img src={photo_data} alt="Latest"/>
      {/if}
      <IconDownload size="24" class={photo_data ? "visible" : "invisible"}/>
    </button>
    <button on:click={takePhoto} class="camera-button" aria-label="Take Photo"></button>
    <div>
      <input
        type="file"
        accept="image/*"
        id="file-upload"
        class="hidden"
        on:change={handleFileChange}
      />

      <label for="file-upload" class="upload-photo">
        <IconFileUpload size="24"/>
      </label>
    </div>
  </div>

  <canvas bind:this={canvas_element} width="640" height="480" class="hidden"></canvas>
</main>
