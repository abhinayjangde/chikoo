<script lang="ts">
  import { check, type Update } from "@tauri-apps/plugin-updater";
  import { relaunch } from "@tauri-apps/plugin-process";
  import { hindiKeyboardMap } from "$lib/hindi-keyboard";

  let menuOpen = false;
  let keyboardMapOpen = false;
  let aboutOpen = false;
  let updateDialogOpen = false;
  let updateState: "idle" | "checking" | "available" | "downloading" | "error" = "idle";
  let update: Update | null = null;
  let updateMessage = "";
  let downloadProgress = 0;
  let downloadedBytes = 0;
  let downloadSize = 0;

  async function checkForUpdates() {
    menuOpen = false;
    updateDialogOpen = true;
    updateState = "checking";
    update = null;
    updateMessage = "Checking for updates…";

    try {
      update = await check();
      if (update) {
        updateState = "available";
        updateMessage = `Version ${update.version} is available.`;
      } else {
        updateState = "idle";
        updateMessage = "You’re up to date.";
      }
    } catch (error) {
      updateState = "error";
      updateMessage = `Could not check for updates. ${String(error)}`;
    }
  }

  async function downloadAndInstallUpdate() {
    if (!update) return;

    updateState = "downloading";
    downloadProgress = 0;
    downloadedBytes = 0;
    downloadSize = 0;
    updateMessage = "Downloading update…";

    try {
      await update.downloadAndInstall((event) => {
        if (event.event === "Started") {
          downloadSize = event.data.contentLength ?? 0;
        } else if (event.event === "Progress") {
          downloadedBytes += event.data.chunkLength;
          downloadProgress = downloadSize
            ? Math.min(100, Math.round((downloadedBytes / downloadSize) * 100))
            : downloadProgress;
        } else if (event.event === "Finished") {
          downloadProgress = 100;
        }
      });
      await relaunch();
    } catch (error) {
      updateState = "error";
      updateMessage = `The update could not be installed. ${String(error)}`;
    }
  }

  function showKeyboardMap() {
    menuOpen = false;
    keyboardMapOpen = true;
  }

  function showAbout() {
    menuOpen = false;
    aboutOpen = true;
  }

  function closeMenuOnOutsideClick(event: MouseEvent) {
    if (!(event.target instanceof Element) || !event.target.closest(".help-menu")) {
      menuOpen = false;
    }
  }
</script>

<svelte:window onclick={closeMenuOnOutsideClick} />

<div class="help-menu">
  <button
    class:open={menuOpen}
    class="help-button"
    type="button"
    aria-haspopup="menu"
    aria-expanded={menuOpen}
    onclick={() => (menuOpen = !menuOpen)}
  >
    Help
  </button>

  {#if menuOpen}
    <div class="menu-items" role="menu">
      <button type="button" role="menuitem" onclick={showKeyboardMap}>Show keyboard map</button>
      <button type="button" role="menuitem" onclick={checkForUpdates}>
        Check for updates...
      </button>
      <button type="button" role="menuitem" onclick={showAbout}>About</button>
    </div>
  {/if}
</div>

{#if updateDialogOpen}
  <div class="dialog-backdrop" role="presentation" onclick={() => (updateDialogOpen = false)}>
    <dialog
      open
      class="dialog update-dialog"
      aria-labelledby="update-title"
      onclick={(event) => event.stopPropagation()}
    >
      <div class="dialog-header">
        <h2 id="update-title">Software update</h2>
        <button class="close-button" type="button" aria-label="Close" onclick={() => (updateDialogOpen = false)}>
          ×
        </button>
      </div>
      <p aria-live="polite">{updateMessage}</p>
      {#if update?.body && updateState === "available"}
        <p class="update-notes">{update.body}</p>
      {/if}
      {#if updateState === "downloading"}
        <progress max="100" value={downloadProgress} aria-label="Update download progress"></progress>
        <p class="progress-label">{downloadProgress}%</p>
      {/if}
      <div class="update-actions">
        {#if updateState === "available"}
          <button class="dialog-action" type="button" onclick={downloadAndInstallUpdate}>Download and install</button>
        {/if}
        {#if updateState !== "checking" && updateState !== "downloading"}
          <button class="secondary-action" type="button" onclick={() => (updateDialogOpen = false)}>Close</button>
        {/if}
      </div>
    </dialog>
  </div>
{/if}

{#if keyboardMapOpen}
  <div class="dialog-backdrop" role="presentation" onclick={() => (keyboardMapOpen = false)}>
    <dialog
      open
      class="dialog keyboard-dialog"
      aria-labelledby="keyboard-map-title"
      onclick={(event) => event.stopPropagation()}
    >
      <div class="dialog-header">
        <h2 id="keyboard-map-title">Hindi keyboard map</h2>
        <button class="close-button" type="button" aria-label="Close" onclick={() => (keyboardMapOpen = false)}>
          ×
        </button>
      </div>
      <p>Use the normal key for regular input and Shift + key for the shifted character.</p>
      <div class="keyboard-map">
        {#each hindiKeyboardMap as key}
          <div class="key-row">
            <strong>{key.code.replace(/^(Key|Digit)/, "")}</strong>
            <span>{key.normal}</span>
            <span>{key.shifted}</span>
          </div>
        {/each}
      </div>
    </dialog>
  </div>
{/if}

{#if aboutOpen}
  <div class="dialog-backdrop" role="presentation" onclick={() => (aboutOpen = false)}>
    <dialog
      open
      class="dialog about-dialog"
      aria-labelledby="about-title"
      onclick={(event) => event.stopPropagation()}
    >
      <div class="dialog-header">
        <h2 id="about-title">About Chikoo</h2>
        <button class="close-button" type="button" aria-label="Close" onclick={() => (aboutOpen = false)}>
          ×
        </button>
      </div>
      <p class="about-name">Chikoo Hindi Typing</p>
      <p>Version 0.1.1</p>
      <p>Developed by Abhinay Jangde.</p>
      <button class="dialog-action" type="button" onclick={() => (aboutOpen = false)}>Close</button>
    </dialog>
  </div>
{/if}

<style>
  .help-menu {
    position: relative;
  }

  .help-button,
  .menu-items button,
  .close-button,
  .dialog-action {
    border: 0;
    cursor: pointer;
  }

  .help-button {
    padding: 7px 13px;
    color: #0a0a0a;
    background: transparent;
    border-radius: 3px;
  }

  .help-button:hover,
  .help-button.open {
    color: white;
    background: #6076cd;
  }

  .menu-items {
    position: absolute;
    top: calc(100% + 5px);
    left: 0;
    z-index: 20;
    display: flex;
    min-width: 190px;
    max-width: calc(100vw - 16px);
    flex-direction: column;
    padding: 5px;
    background: white;
    border: 1px solid #bcbcbc;
    box-shadow: 0 5px 14px rgb(0 0 0 / 22%);
  }

  .menu-items button {
    padding: 9px 12px;
    color: #161616;
    text-align: left;
    background: white;
  }

  .menu-items button:hover {
    color: white;
    background: #6076cd;
  }

  .dialog-backdrop {
    position: fixed;
    inset: 0;
    z-index: 30;
    display: grid;
    place-items: center;
    padding: 24px;
    background: rgb(0 0 0 / 45%);
  }

  .dialog {
    width: min(620px, 100%);
    max-height: 85vh;
    padding: 22px;
    overflow: auto;
    color: #161616;
    background: #fff;
    box-shadow: 0 10px 30px rgb(0 0 0 / 30%);
  }

  .dialog-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 20px;
  }

  h2 {
    margin: 0;
    font-size: 1.3rem;
  }

  .close-button {
    width: 30px;
    height: 30px;
    color: #333;
    background: #e7e7e7;
    font-size: 1.4rem;
    line-height: 1;
  }

  .keyboard-map {
    display: grid;
    grid-template-columns: repeat(4, minmax(0, 1fr));
    gap: 7px;
    margin-top: 18px;
  }

  .key-row {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 5px;
    padding: 7px;
    background: #edf0fb;
    text-align: center;
  }

  .key-row strong {
    grid-column: 1 / -1;
    font-size: 0.78rem;
  }

  .key-row span {
    padding: 4px;
    background: white;
    font-family: "Nirmala UI", "Noto Sans Devanagari", sans-serif;
    font-size: 1.1rem;
  }

  .about-name {
    margin-top: 24px;
    font-size: 1.1rem;
    font-weight: 700;
  }

  .update-dialog {
    width: min(460px, 100%);
  }

  .update-notes {
    max-height: 180px;
    padding: 10px;
    overflow: auto;
    white-space: pre-wrap;
    background: #f3f4f7;
  }

  .update-dialog progress {
    width: 100%;
    accent-color: #6076cd;
  }

  .progress-label {
    margin-top: 4px;
    text-align: right;
  }

  .update-actions {
    display: flex;
    justify-content: flex-end;
    gap: 8px;
    margin-top: 20px;
  }

  .secondary-action {
    padding: 8px 16px;
    color: #222;
    background: #e7e7e7;
    border: 0;
    cursor: pointer;
  }

  .dialog-action {
    padding: 8px 16px;
    color: white;
    background: #6076cd;
  }

  @media (max-width: 900px) {
    .keyboard-map {
      grid-template-columns: repeat(3, minmax(0, 1fr));
    }
  }
</style>
