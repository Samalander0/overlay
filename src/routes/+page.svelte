<script>
  import { onMount } from 'svelte';
  import '$lib/main.scss';
  import { IconDownload, IconFileUpload, IconSettings, IconRepeat } from "@tabler/icons-svelte";
    import { flip } from 'svelte/animate';

  // Elements
  let video_element,
      canvas_element,
      settings_dialog_element,
      welcome_dialog_element;

  let photo_data; 
  let selfie_mode = false;
  let onboarding_step = 0;

  let overlay_opacity = 0.15;

  let devices = [];
  let selected_device_id = null;

  async function startCamera(device_id) { // Note: used to be for starting the camera but is now just camera setup. TODO: rename function
    // Stop existing stream if any
    if (video_element?.srcObject) {
      (video_element.srcObject)
        .getTracks()
        .forEach((track) => track.stop());
    }

    if (!device_id && selected_device_id) {
      device_id = selected_device_id;
    }

    try {
      const constraints = {
        video: device_id
          ? { deviceId: { exact: device_id } }
          : { deviceId: { ideal: 'environment' } },
        audio: false
      };

      const stream = await navigator.mediaDevices.getUserMedia(constraints);
      video_element.srcObject = stream;

      await video_element.play();
    } catch (err) {
      console.error('Error accessing camera:', err);
      welcome_dialog_element.showModal();
    }

    await getDevices();
  }

  async function getDevices() {
    const allDevices = await navigator.mediaDevices.enumerateDevices();
    devices = allDevices.filter((d) => d.kind === 'videoinput');
    console.log('Available devices:', devices);
  }

  async function takePhoto() {
    const context = canvas_element.getContext('2d');
    if (!context) return;

    canvas_element.width = video_element.videoWidth;
    canvas_element.height = video_element.videoHeight;

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

  // Automattically start camera when component mounts
  onMount(() => {
    startCamera();
    setTimeout(getDevices, 1000);
  });

  async function start() {
    await startCamera();

    if (devices.filter((d) => d.label == "Back Camera").length > 0) { // Note: in iOS, the back camera is called "Back Camera" - same for front camera. Need to test in Android to see how cameras are labeled
      selected_device_id = devices.filter((d) => d.label == "Back Camera")[0].deviceId;
    } else if (devices.length > 0) {
      selected_device_id = devices[0].deviceId;
    }

    onboarding_step = 1;
  }

  function setDevice(label) {
    selected_device_id = devices.filter((d) => d.label == label)[0].deviceId;
  }

  function flipCamera() {
    if (selfie_mode) {
      setDevice("Back Camera")
    } else {
      setDevice("Front Camera")
    }

    selfie_mode = !selfie_mode;
  }

  $: if (selected_device_id) {
    startCamera(selected_device_id); 
  }
</script>

<main class="app">
  <div class="top-bar">
    <h1>Overlay</h1>
    <button class="settings-button" on:click={() => {settings_dialog_element.showModal(); getDevices()}}>
      <IconSettings size="32" stroke="1.75"/>
    </button>
  </div>

  <dialog bind:this={welcome_dialog_element} class="welcome-popup">
    <h2>Welcome!</h2>
    <p>This app allows you to add an overlay to your camera. If you haven't already, make sure to allow camera access.</p>
    <button on:click={() => {start()}} class="close">Start -></button>
  </dialog>

  <dialog bind:this={settings_dialog_element} class="settings-popup">
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
    <button on:click={() => settings_dialog_element.close()} class="close">Close</button>
  </dialog>

  <div class="video-wrapper" style={`--overlay-opacity: ${overlay_opacity}`}>
    <video bind:this={video_element} autoplay playsinline class="main-video"></video>
    {#if image_url}
      <img src={image_url} alt="Selected" class="overlay-image" />
    {/if}

    <div class="camera-buttons">
      {#if devices.filter((d) => d.label == "Back Camera").length > 0}
        <div class="zoom-controls">
          {#if devices.filter((d) => d.label == "Back Ultra Wide Camera").length > 0}
            <button on:click={() => {setDevice("Back Ultra Wide Camera")}}>0.5</button>
          {/if}
          <button on:click={() => {setDevice("Back Camera")}}>1</button>
          {#if devices.filter((d) => d.label == "Back Telephoto Camera").length > 0}
            <button on:click={() => {setDevice("Back Telephoto Camera")}}>3</button>
          {/if}
        </div>
      {/if}
      {#if devices.filter((d) => d.label == "Front Camera").length > 0 && devices.filter((d) => d.label == "Back Camera").length > 0}
        <button class="flip-camera" on:click={flipCamera}>
          <IconRepeat size="24"/>
        </button>
      {/if}
    </div>
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

  <canvas bind:this={canvas_element} width="480" height="640" class="hidden"></canvas>
</main>
