<template>
  <div class="container">
    <div class="main">
      <div class="qrd-logo" />
      <div>
        <input type="text" class="qr-text" v-model="settings.qrText" placeholder="Type an URL or some text"/>
      </div>
      <div class="color-settings">
        <input type="color" v-model="settings.foreColor" title="Foreground"/>
        <input type="color" v-model="settings.backColor" title="Background" />
        <div class="setting">
          <input type="checkbox" :checked="settings.backAlpha === 0" id="transparency" @click="() => {
            settings.backAlpha = settings.backAlpha === 0 ? 255 : 0
          }"/> <label for="transparency">Transparent</label>
        </div>
      </div>
      <div class="advanced-toggle">
        <input type="checkbox" v-model="settings.showAdvancedOptions" id="advanced-options">
        <label for="advanced-options">Advanced</label>
      </div>
    </div>
    <div class="advanced-options" v-if="settings.showAdvancedOptions">
      <div class="setting" title="Set the foreground alpha.">
        Foreground Alpha
        <input type="number" min="0" max="255" v-model="settings.foreAlpha" placeholder="0-255"/>
      </div>
      <div class="setting" title="Set the background alpha.">
        Background Alpha
        <input type="number" min="0" max="255" v-model="settings.backAlpha" placeholder="0-255"/>
      </div>
      <div class="setting" title="Set the scale.">
        Scale
        <input type="number" min="0" v-model="settings.scale" placeholder="0"/>
      </div>
      <div class="setting" title="Set the scale.">
        Margin
        <input type="number" min="0" v-model="settings.margin" placeholder="0"/>
      </div>
      <div class="setting" title="Set the minimum width (px).">
        Min Width
        <input type="number" min="0" v-model="settings.minWidth" placeholder="0px"/>
      </div>
      <div class="setting" title="Forces the QR version (should keep this off, it is selected automatically).">
        <input type="checkbox" v-model="settings.forcedVersionEnabled" id="force-version"/>
        <label for="force-version">Force Version</label>
        <input type="number" min="1" max="40" v-model="settings.forcedVersion" placeholder="1-40"/>
      </div>
      <div class="setting" title="Set the mask pattern.">
        Mask Pattern
        <input type="number" min="0" max="7" v-model="settings.maskPattern" placeholder="0-7"/>
      </div>
      <div class="setting" title="Error correction level (Default: Medium).">
        Error correction level
        <select v-model="settings.errorCorrectionLevel">
          <option value="low">Low</option>
          <option value="medium">Medium</option>
          <option value="quartile">Quartile</option>
          <option value="high">High</option>
        </select>
      </div>
    </div>
    <div class="save-options">
      <button class="btn-save" @click="toPNG" :disabled="lastError">PNG</button>
      <button class="btn-save" @click="toSVG" :disabled="lastError">SVG</button>
    </div>
    <div class="preview-container">
      <div v-if="lastError" class="last-error">{{ lastError }}</div>
      <div :key="settings" v-else class="preview-info">
        {{ renderedSize }}x{{ renderedSize }}
      </div>
      <canvas ref="qrPreview" v-show="!lastError"></canvas>
    </div>
    <footer>
      <div>
        A simple wrapper around <a href="https://www.npmjs.com/package/qrcode" target="_blank">node-qrcode</a>.
        <span class="why-link" @click="expandWhy = !expandWhy">Why?</span>
      </div>
      <div>
        <span class="qrd-inline">QRD</span> &copy; fisk, 2024. <a href="https://ko-fi.com/fisktech" target="_blank">Support me on Ko-fi!</a>
      </div>
      <div v-if="expandWhy" class="why">
        <p>
          QR codes are super useful for many reasons. They are very simple things. This tool did not take a long time to
          write at all (especially considering I did very little of the work).
        </p>
        <p>
          If you type "QR code generator" into Google, you'll find a dozen sponsored results for sites requiring
          sign-ups, etc. Most of these want a lot of money for little in return. The internet is so saturated with these websites that
          finding one that isn't garbage is near impossible.
        </p>
        <p>I'm just a little person, and I just want to make a QR code sometimes. If you also like making QR codes, I
          gotchu.
        </p>
      </div>
    </footer>
  </div>
</template>

<script setup>
import * as QRCode from 'qrcode'
import { onMounted, reactive, ref, watch} from "vue";

const qrPreview = ref(null)

const expandWhy = ref(false)

const lastError = ref(null)

const settings = reactive(getSettingsFromStorage())

const renderedSize = ref(0)

watch(settings, updateQR)

function getSettingsFromStorage() {
  const stored = localStorage.getItem("settings")

  if (stored != null) {
    return JSON.parse(stored)
  }

  return getDefaultSettings()
}

function getDefaultSettings() {
  return {
    foreColor: '#000000',
    backColor: '#FFFFFF',
    foreAlpha: 255,
    backAlpha: 255,
    maskPattern: 0,
    margin: 4,
    scale: 4,
    minWidth: 512,
    errorCorrectionLevel: "medium",
    forcedVersionEnabled: false,
    forcedVersion: 1,
    qrText: "https://google.co.uk",
    showAdvancedOptions: false,
  }
}

function putSettingsToStorage() {
  localStorage.setItem("settings", JSON.stringify(settings))
}

onMounted(() => {
  updateQR()
})

function buildOptions() {
  const options = {}

  if (settings.forcedVersionEnabled) {
    options.version = settings.forcedVersion
  }

  options.maskPattern = settings.maskPattern
  options.errorCorrectionLevel = settings.errorCorrectionLevel
  options.scale = settings.scale
  options.width = settings.minWidth
  options.margin = settings.margin

  const foreAlphaHex = Math.max(0, Math.min(255, settings.foreAlpha)).toString(16).padStart(2, '0')
  const backAlphaHex = Math.max(0, Math.min(255, settings.backAlpha)).toString(16).padStart(2, '0')

  options.color = {
    dark: `${settings.foreColor}${foreAlphaHex}`,
    light: `${settings.backColor}${backAlphaHex}`
  }

  return options
}

function updateQR() {
  putSettingsToStorage()

  QRCode.toCanvas(qrPreview.value, settings.qrText, buildOptions(), (error) => {
    lastError.value = error?.message
    renderedSize.value = qrPreview.value?.width ?? 0
  })
}

function download(filename, data) {
  const anchor = document.createElement('a');
  anchor.setAttribute('download', filename);
  anchor.setAttribute('href', data);
  anchor.click();
}

function toPNG() {
  const opts = buildOptions()
  opts.type = 'image/png'

  QRCode.toDataURL(settings.qrText, opts,  (e, data) => {
    console.log(data)
    download("qr.png", data)
  })
}

function toSVG() {
  const opts = buildOptions()

  QRCode.toString(settings.qrText, opts,  (e, data) => {
    download("qr.svg", `data:image/svg+xml;base64,${btoa(data)}`)
  })
}

</script>

<style lang="scss">
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  margin-top: 60px;

  user-select: none;
}

.why {
  width: 20em;
}

.container {
  display: flex;
  align-items: center;
  flex-direction: column;
}

%border {
  border: {
    color: #ccc;
    width: 1px;
    style: solid;
    radius: 0.5em;
  }
}


.preview-container {
  @extend %border;

  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;

  margin-bottom: 1em;
  overflow: hidden;

  max-width: 90vw;
  max-height: 90vw;

  > canvas {
    max-width: 100%;
  }
}

.preview-info {
  font-size: 0.7em;
}

.btn-save {
  @extend %border;
  background-color: #779b8a;
  border-color : #a1d2bb;
  color: #fff;
  font-weight: bold;
  padding: 0.5em 0.7em;
  line-height: 1;
  cursor: pointer;
  font-size: 1.05em;
}

body {
  background-color: #f1ebea;
}

footer {
  font-size: 0.8em;
}

input[type=color] {
  padding: 0;
  height: 50px;
  width: 48px;
  border: none;
}

input[type=text], input[type=number] {
  @extend %border;
}

.qr-text {
  @extend %border;

  font-size: 1.1em;
  padding: 0.3em;
  color: #666;
  transition: all 0.3s;

  &:hover {
    color: #000;
  }

  &:focus {
    color: #000;
  }

  text-align: center;
  width: 30em;
  max-width: 90vw;
}

.last-error {
  padding: 0.5em;
  max-width: 400px;
}

.color-settings {
  display: flex;
  align-items: center;
  justify-content: center;
  column-gap: 0.3em;
}

.advanced-options {
  @extend %border;
  padding: 0.5em;
  background-color: #fff;
  display: flex;
  flex-direction: column;
  align-items: start;
  font-size: 0.8em;
  margin-bottom: 0.5em;
}

.setting {
  display: flex;
  align-items: center;
  column-gap: 0.3em;
}

.advanced-toggle {
  display: inline-flex;
  align-items: center;
  column-gap: 0.3em;
}

.save-options {
  display: flex;
  flex-direction: row;
  column-gap: 0.5em;
  margin-bottom: 0.5em;
}

input[type=checkbox] {
  margin: 0;
}

.why-link {
  cursor: pointer;
  color: #517966;
}

a {
  color: #517966;
  text-decoration: none;
}

.qrd-inline {
  color: #517966;
  font-weight: bold;
}

.qrd-logo {
  background: {
    image: url("../public/favicon.png");
    size: contain;
  };

  width: 96px;
  height: 96px;
  margin-bottom: 0.5em;
}

.main {
  display: flex;
  flex-direction: column;
  align-items: center;
}
</style>
