<template>
  <div :class="['mermaid-wrapper', props.class]">
    <!-- Thumbnail -->
    <div
      class="mermaid-thumb"
      role="button"
      tabindex="0"
      :aria-label="'Open diagram preview'"
      @click="openLightbox"
      @keydown.enter="openLightbox"
      @keydown.space.prevent="openLightbox"
      title="Click to preview"
    >
      <div ref="thumbnailRef" v-html="svg" class="mermaid-thumb-svg"></div>
      <div class="mermaid-thumb-overlay">
        <svg class="mermaid-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <circle cx="11" cy="11" r="8"/><path d="M21 21l-4.35-4.35"/><path d="M11 8v6"/><path d="M8 11h6"/>
        </svg>
      </div>
    </div>

    <!-- Lightbox -->
    <Teleport to="body">
      <div v-if="lightboxOpen" class="mermaid-lightbox" @click.self="closeLightbox">
        <div class="mermaid-lightbox-backdrop"></div>
        <div class="mermaid-lightbox-panel" role="dialog" aria-modal="true">
          <!-- Toolbar -->
          <div class="mermaid-toolbar">
            <span class="mermaid-toolbar-title">Diagram Preview</span>
            <div class="mermaid-toolbar-actions">
              <span class="mermaid-zoom-level">{{ Math.round((scale / fitScale) * 100) }}%</span>
              <button class="mermaid-btn" @click="zoomOut" title="Zoom out" aria-label="Zoom out">
                <svg class="mermaid-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="11" cy="11" r="8"/><path d="M21 21l-4.35-4.35"/><path d="M8 11h6"/></svg>
              </button>
              <button class="mermaid-btn" @click="zoomIn" title="Zoom in" aria-label="Zoom in">
                <svg class="mermaid-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="11" cy="11" r="8"/><path d="M21 21l-4.35-4.35"/><path d="M11 8v6"/><path d="M8 11h6"/></svg>
              </button>
              <button class="mermaid-btn" @click="resetView" title="Fit to viewport" aria-label="Fit to viewport">
                <svg class="mermaid-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="1 4 1 10 7 10"/><path d="M3.51 15a9 9 0 1 0 2.13-9.36L1 10"/></svg>
              </button>
              <span class="mermaid-toolbar-sep"></span>
              <button class="mermaid-btn" @click="downloadSvg" title="Download SVG" aria-label="Download SVG">
                <svg class="mermaid-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="7 10 12 15 17 10"/><line x1="12" y1="15" x2="12" y2="3"/></svg>
              </button>
              <span class="mermaid-toolbar-sep"></span>
              <button class="mermaid-btn mermaid-btn-close" @click="closeLightbox" title="Close" aria-label="Close">
                <svg class="mermaid-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="18" y1="6" x2="6" y2="18"/><line x1="6" y1="6" x2="18" y2="18"/></svg>
              </button>
            </div>
          </div>
          <!-- Viewport -->
          <div
            ref="viewportRef"
            class="mermaid-viewport"
            :class="{ 'mermaid-viewport--panning': isPanning }"
            @pointerdown="onPointerDown"
            @pointermove="onPointerMove"
            @pointerup="onPointerUp"
            @pointercancel="onPointerUp"
            @wheel.prevent="onWheel"
          >
            <div v-if="previewError" class="mermaid-preview-error" role="status">
              Unable to render diagram preview.
            </div>
            <div
              class="mermaid-viewport-content"
              :style="{
                transform: `translate(${panX}px, ${panY}px)`,
              }"
              v-html="svg"
            ></div>
          </div>
          <!-- Hint -->
          <div class="mermaid-lightbox-hint">Scroll to zoom &middot; Drag to pan &middot; Esc to close</div>
        </div>
      </div>
    </Teleport>
  </div>
</template>

<script setup>
import { nextTick, onMounted, onUnmounted, ref, toRaw } from "vue";
import { render, init } from "./mermaid";

//get mermaid settings
import { useData } from "vitepress";

const pluginSettings = ref({
  securityLevel: "loose",
  startOnLoad: false,
  externalDiagrams: [],
});
const { page } = useData();
const { frontmatter } = toRaw(page.value);
const mermaidPageTheme = frontmatter.mermaidTheme || "";

const props = defineProps({
  graph: {
    type: String,
    required: true,
  },
  id: {
    type: String,
    required: true,
  },
  class: {
    type: String,
    required: false,
    default: "mermaid",
  },
});

const svg = ref(null);
const lightboxOpen = ref(false);
const scale = ref(1);
const fitScale = ref(1);
const previewError = ref(false);
const panX = ref(0);
const panY = ref(0);
const isPanning = ref(false);
const viewportRef = ref(null);
const thumbnailRef = ref(null);

// Intrinsic (unscaled) diagram pixel size, captured from the viewBox
let naturalW = 0;
let naturalH = 0;

let mut = null;
let dragStart = null;

// ---- Lightbox controls ----

const openLightbox = () => {
  lightboxOpen.value = true;
  scale.value = 1;
  previewError.value = false;
  panX.value = 0;
  panY.value = 0;
  // Wait for Vue to flush Teleport + v-html, then for the SVG element to appear,
  // then for one more frame so the layout has settled.
  nextTick(() => {
    let attempts = 0;
    const tryFit = () => {
      const svgEl = getLightboxSvg();
      if (svgEl && svgEl.viewBox?.baseVal?.width) {
        fitToViewport();
      } else if (attempts < 30) {
        attempts++;
        requestAnimationFrame(tryFit);
      } else {
        previewError.value = true;
      }
    };
    requestAnimationFrame(tryFit);
  });
  document.addEventListener("keydown", onKeydown);
  document.body.style.overflow = "hidden";
};

const closeLightbox = () => {
  lightboxOpen.value = false;
  document.removeEventListener("keydown", onKeydown);
  document.body.style.overflow = "";
};

// Get the rendered SVG element in the lightbox viewport
const getLightboxSvg = () => {
  if (!viewportRef.value) return null;
  return viewportRef.value.querySelector(".mermaid-viewport-content svg");
};

// Mermaid renders flowcharts as `<svg width="100%" viewBox="0 0 W H"`,
// which collapses to zero width inside the inline-block `.mermaid-viewport-content`.
// Replace the 100% width with the intrinsic pixel size from the viewBox so the
// SVG actually has a layout size inside the lightbox.
const fixSvgSizing = (svgEl) => {
  if (!svgEl) return false;
  const vb = svgEl.viewBox?.baseVal;
  if (!vb || !vb.width || !vb.height) return false;
  // Strip any max-width that mermaid put on the element
  if (svgEl.style.maxWidth) svgEl.style.maxWidth = "none";
  // Replace `width="100%"` / percent-height with fixed pixel size
  const wAttr = svgEl.getAttribute("width");
  const hAttr = svgEl.getAttribute("height");
  if (wAttr && wAttr.endsWith("%")) svgEl.setAttribute("width", String(vb.width));
  if (hAttr && hAttr.endsWith("%")) svgEl.setAttribute("height", String(vb.height));
  // If width is missing entirely, set it so the SVG has a size
  if (!svgEl.getAttribute("width")) svgEl.setAttribute("width", String(vb.width));
  if (!svgEl.getAttribute("height")) svgEl.setAttribute("height", String(vb.height));
  return true;
};

// Natural diagram size, preferring viewBox (matches mermaid intrinsic graph size)
// then width/height attributes, then getBBox as fallback.
const getNaturalSize = (svgEl) => {
  if (!svgEl) return { w: 0, h: 0 };
  let w = 0;
  let h = 0;
  const vb = svgEl.viewBox?.baseVal;
  if (vb && vb.width && vb.height) {
    w = vb.width;
    h = vb.height;
  }
  if (!w || !h) {
    try {
      const bb = svgEl.getBBox();
      if (bb.width && bb.height) {
        w = bb.width;
        h = bb.height;
      }
    } catch {
      /* getBBox can fail when element is hidden */
    }
  }
  if (!w || !h) {
    w = svgEl.width?.baseVal?.value || svgEl.clientWidth || 0;
    h = svgEl.height?.baseVal?.value || svgEl.clientHeight || 0;
  }
  return { w, h };
};

// Fit the diagram into the viewport then center it.
const fitToViewport = () => {
  const svgEl = getLightboxSvg();
  if (!svgEl) return;
  fixSvgSizing(svgEl);
  const vp = viewportRef.value.getBoundingClientRect();
  const { w, h } = getNaturalSize(svgEl);
  if (!w || !h) {
    // Fallback: stack at center with scale 1
    fitScale.value = 1;
    scale.value = 1;
    panX.value = vp.width / 2;
    panY.value = 24;
    return;
  }
  naturalW = w;
  naturalH = h;
  // Leave a bit of padding around the diagram
  const availW = Math.max(vp.width - 48, 32);
  const availH = Math.max(vp.height - 48, 32);
  const fit = Math.min(availW / w, availH / h);
  // Cap initial zoom in so the diagram doesn't appear huge if it's tiny
  fitScale.value = Math.min(fit, 3);
  scale.value = fitScale.value;
  applyZoomToSvg();
  panX.value = (vp.width - w * scale.value) / 2;
  panY.value = (vp.height - h * scale.value) / 2;
};

const centerDiagram = () => {
  const svgEl = getLightboxSvg();
  if (!svgEl) return;
  const vp = viewportRef.value.getBoundingClientRect();
  const { w, h } = getNaturalSize(svgEl);
  if (w && h) {
    naturalW = w;
    naturalH = h;
  }
  panX.value = (vp.width - (naturalW || w) * scale.value) / 2;
  panY.value = (vp.height - (naturalH || h) * scale.value) / 2;
};

// Apply the current scale to the SVG element by setting its width/height attributes
// directly. Unlike CSS transforms, resizing via width/height attributes causes the
// browser to re-layout the SVG at the new resolution, so text and lines stay crisp
// at any zoom level (no rasterized upscale blur).
const applyZoomToSvg = () => {
  const svgEl = getLightboxSvg();
  if (!svgEl || !naturalW || !naturalH) return;
  svgEl.setAttribute("width", String(naturalW * scale.value));
  svgEl.setAttribute("height", String(naturalH * scale.value));
};

// Centered zooming: keep the content point under `centerX/centerY` stationary.
const setScale = (newScale, centerX, centerY) => {
  const clamped = Math.min(Math.max(newScale, 0.05), 20);
  if (clamped === scale.value) return;

  const vp = viewportRef.value?.getBoundingClientRect();
  const cx = centerX !== undefined ? centerX : (vp ? vp.width / 2 : 0);
  const cy = centerY !== undefined ? centerY : (vp ? vp.height / 2 : 0);

  // The content point that was under (cx, cy) before zoom: (cx - pan) / scale
  // After zoom, we want it to remain at the same viewport position:
  // pan' = (cx,cy) - contentPoint * newScale
  const contentX = (cx - panX.value) / scale.value;
  const contentY = (cy - panY.value) / scale.value;
  panX.value = cx - contentX * clamped;
  panY.value = cy - contentY * clamped;
  scale.value = clamped;
  applyZoomToSvg();
};

const zoomIn = () => {
  setScale(scale.value * 1.25);
};

const zoomOut = () => {
  setScale(scale.value / 1.25);
};

const resetView = () => {
  fitToViewport();
};

// Wheel zoom: gentler step for trackpads (small, frequent deltaY events) and
// a slightly larger step for physical mouse wheels (large, discrete deltaY).
// The zoom centers on the cursor so it feels natural.
const onWheel = (e) => {
  const vp = viewportRef.value?.getBoundingClientRect();
  if (!vp) return;
  // Normalize deltaY across browsers and deltaMode (line vs pixel scrolling).
  let delta = e.deltaY;
  if (e.deltaMode === 1) delta *= 16; // lines -> pixels
  else if (e.deltaMode === 2) delta *= vp.height; // pages -> pixels
  // Continuous input (trackpad / pinch): scale smoothly with |deltaY|.
  // Discrete input (mouse wheel, |delta| >= ~50): one notch is a fixed step.
  let factor;
  if (Math.abs(delta) >= 40) {
    factor = delta > 0 ? 1 / 1.1 : 1.1;
  } else {
    // Exponential zoom for smooth trackpad feel; multiplier per pixel scrolled.
    factor = Math.exp(-delta * 0.004);
  }
  setScale(scale.value * factor, e.clientX - vp.left, e.clientY - vp.top);
};

// ---- Pan (drag) ----

const onPointerDown = (e) => {
  if (e.button !== 0) return;
  isPanning.value = true;
  dragStart = { x: e.clientX - panX.value, y: e.clientY - panY.value };
  e.currentTarget.setPointerCapture(e.pointerId);
};

const onPointerMove = (e) => {
  if (!isPanning.value || !dragStart) return;
  panX.value = e.clientX - dragStart.x;
  panY.value = e.clientY - dragStart.y;
};

const onPointerUp = () => {
  isPanning.value = false;
  dragStart = null;
};

// ---- Download ----

const downloadSvg = () => {
  if (!svg.value) return;
  // Strip the hidden salt span from the SVG content before download
  const raw = svg.value.replace(/<span[^>]*style="display:\s*none[^"]*"[^>]*>.*?<\/span>\s*$/s, "");
  const blob = new Blob([raw], { type: "image/svg+xml;charset=utf-8" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = `${props.id || "diagram"}.svg`;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
};

// ---- Keyboard ----

const onKeydown = (e) => {
  if (e.key === "Escape") closeLightbox();
  if (e.key === "+" || e.key === "=") zoomIn();
  if (e.key === "-") zoomOut();
  if (e.key === "0") resetView();
};

// ---- Render ----

onMounted(async () => {
  await init(pluginSettings.value.externalDiagrams);
  let settings = await import("virtual:mermaid-config");
  if (settings?.default) pluginSettings.value = settings.default;

  mut = new MutationObserver(async () => await renderChart());
  mut.observe(document.documentElement, { attributes: true });
  await renderChart();

  //refresh images on first render
  const hasImages =
    /<img([\w\W]+?)>/.exec(decodeURIComponent(props.graph))?.length > 0;
  if (hasImages)
    setTimeout(() => {
      let imgElements = document.getElementsByTagName("img");
      let imgs = Array.from(imgElements);
      if (imgs.length) {
        Promise.all(
          imgs
            .filter((img) => !img.complete)
            .map(
              (img) =>
                new Promise((resolve) => {
                  img.onload = img.onerror = resolve;
                })
            )
        ).then(async () => {
          await renderChart();
        });
      }
    }, 100);
});

onUnmounted(() => {
  mut?.disconnect();
  document.removeEventListener("keydown", onKeydown);
  document.body.style.overflow = "";
});

const renderChart = async () => {
  const hasDarkClass = document.documentElement.classList.contains("dark");
  let mermaidConfig = {
    ...pluginSettings.value,
  };

  if (mermaidPageTheme) mermaidConfig.theme = mermaidPageTheme;
  if (hasDarkClass) mermaidConfig.theme = "dark";

  let svgCode = await render(
    props.id,
    decodeURIComponent(props.graph),
    mermaidConfig
  );
  // This is a hack to force v-html to re-render, otherwise the diagram disappears
  // when **switching themes** or **reloading the page**.
  // The cause is that the diagram is deleted during rendering (out of Vue's knowledge).
  // Because svgCode does NOT change, v-html does not re-render.
  // This is not required for all diagrams, but it is required for c4c, mindmap and zenuml.
  const salt = Math.random().toString(36).substring(7);
  svg.value = svgCode + ' <span style="display: none">' + salt + '</span>';
  // Fix the thumbnail SVG sizing so viewBox is used instead of collapsing to 300px
    nextTick(() => {
      const thumbSvg = thumbnailRef.value?.querySelector("svg");
    if (thumbSvg) fixSvgSizing(thumbSvg);
  });
};
</script>

<style>
/* ---- Wrapper ---- */
.mermaid-wrapper {
  position: relative;
  margin: 1rem 0;
}

/* ---- Thumbnail ---- */
.mermaid-thumb {
  position: relative;
  display: block;
  border-radius: 8px;
  border: 1px solid var(--vp-c-divider, #e2e2e3);
  background: var(--vp-c-bg-soft, #f6f6f7);
  overflow: hidden;
  cursor: zoom-in;
  transition: box-shadow 0.2s ease;
}
.mermaid-thumb:hover {
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
}
.mermaid-thumb-svg {
  padding: 1rem;
  text-align: center;
  max-width: 100%;
  overflow: hidden;
}
.mermaid-thumb-svg svg {
  display: inline-block;
  max-width: 100% !important;
  height: auto;
  /* Keep the SVG vector perfect */
  image-rendering: auto;
}
.mermaid-thumb-overlay {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(0, 0, 0, 0);
  opacity: 0;
  transition: opacity 0.2s ease, background 0.2s ease;
  pointer-events: none;
}
.mermaid-thumb:hover .mermaid-thumb-overlay {
  opacity: 1;
  background: rgba(0, 0, 0, 0.06);
}
.mermaid-thumb-overlay .mermaid-icon {
  width: 40px;
  height: 40px;
  color: var(--vp-c-text-1, #333);
}

/* ---- Lightbox ---- */
.mermaid-lightbox {
  position: fixed;
  inset: 0;
  z-index: 9999;
  display: flex;
  align-items: center;
  justify-content: center;
}
.mermaid-lightbox-backdrop {
  position: absolute;
  inset: 0;
  background: rgba(0, 0, 0, 0.6);
  backdrop-filter: blur(2px);
}
.mermaid-lightbox-panel {
  position: relative;
  display: flex;
  flex-direction: column;
  width: 92vw;
  height: 88vh;
  background: var(--vp-c-bg, #fff);
  border-radius: 12px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  overflow: hidden;
}

/* ---- Toolbar ---- */
.mermaid-toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0.5rem 1rem;
  border-bottom: 1px solid var(--vp-c-divider, #e2e2e3);
  background: var(--vp-c-bg-soft, #f6f6f7);
  flex-shrink: 0;
}
.mermaid-toolbar-title {
  font-size: 0.85rem;
  font-weight: 600;
  color: var(--vp-c-text-2, #666);
  user-select: none;
}
.mermaid-toolbar-actions {
  display: flex;
  align-items: center;
  gap: 2px;
}
.mermaid-zoom-level {
  font-size: 0.78rem;
  font-variant-numeric: tabular-nums;
  color: var(--vp-c-text-2, #666);
  min-width: 3.2em;
  text-align: center;
  user-select: none;
  margin-right: 4px;
}
.mermaid-toolbar-sep {
  width: 1px;
  height: 20px;
  background: var(--vp-c-divider, #e2e2e3);
  margin: 0 4px;
}

/* ---- Toolbar Buttons ---- */
.mermaid-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 32px;
  height: 32px;
  border: none;
  background: transparent;
  border-radius: 6px;
  cursor: pointer;
  color: var(--vp-c-text-1, #333);
  transition: background 0.15s ease, color 0.15s ease;
  padding: 0;
}
.mermaid-btn:hover {
  background: var(--vp-c-bg-mute, #e8e8e8);
}
.mermaid-btn:active {
  background: var(--vp-c-bg-mute, #ddd);
}
.mermaid-btn-close:hover {
  color: #e53e3e;
}
.mermaid-icon {
  width: 18px;
  height: 18px;
}

/* ---- Viewport ---- */
.mermaid-viewport {
  flex: 1;
  overflow: hidden;
  cursor: grab;
  user-select: none;
  touch-action: none;
  position: relative;
}
.mermaid-viewport--panning {
  cursor: grabbing;
}
.mermaid-preview-error {
  position: absolute;
  inset: 0;
  display: grid;
  place-items: center;
  color: var(--vp-c-text-2, #666);
}
.mermaid-viewport-content {
  display: inline-block;
  /* Cancel any inherited rules that might constrain the SVG. */
  max-width: none !important;
  width: auto !important;
  height: auto !important;
}
.mermaid-viewport-content svg {
  display: block;
  /* Override global `svg { max-width: 100% }` so the diagram can be rendered
     at its intrinsic pixel size inside the lightbox. Width/height are managed
     by JS (fixSvgSizing + applyZoomToSvg) so the vector output stays crisp at
     any zoom level. */
  max-width: none !important;
  max-height: none !important;
}

/* ---- Hint ---- */
.mermaid-lightbox-hint {
  text-align: center;
  padding: 0.4rem;
  font-size: 0.75rem;
  color: var(--vp-c-text-3, #999);
  border-top: 1px solid var(--vp-c-divider, #e2e2e3);
  user-select: none;
}

/* ---- Dark mode adjustments ---- */
.dark .mermaid-thumb {
  border-color: var(--vp-c-divider, #3a3a3a);
  background: var(--vp-c-bg-soft, #252529);
}
.dark .mermaid-lightbox-panel {
  background: var(--vp-c-bg, #1b1b1f);
}
.dark .mermaid-toolbar {
  background: var(--vp-c-bg-soft, #252529);
  border-bottom-color: var(--vp-c-divider, #3a3a3a);
}
.dark .mermaid-lightbox-hint {
  border-top-color: var(--vp-c-divider, #3a3a3a);
}
</style>
