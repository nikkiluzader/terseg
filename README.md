# TerSeg

A browser-based satellite image segmentation tool that reduces complex imagery into flat, uniform-color regions using k-means clustering. Built with Vue 3, Tailwind CSS 4, and Web Workers for off-thread processing.

![Vue 3](https://img.shields.io/badge/Vue-3.5-42b883)
![Vite](https://img.shields.io/badge/Vite-6-646cff)
![Tailwind](https://img.shields.io/badge/Tailwind-4-38bdf8)

## Overview

TerSeg takes a satellite or aerial image and produces a simplified "poster" version with a controlled number of colors and crisp, blocky region boundaries. The output is ideal for:

- **Game engine base maps** — PBR base color textures where land cover is represented by uniform regions
- **Land cover classification** — quick visual segmentation of agricultural fields, urban areas, forests, water bodies
- **Cartographic simplification** — reducing photographic imagery to clean, stylised map tiles

## How It Works

The segmentation runs in two phases inside a Web Worker so the UI stays fully responsive.

### Phase 1 — Posterize

1. **Downscale** the image to a configurable max processing size (default 512px) to keep computation fast
2. **Box blur** (separable, O(n) per pass) reduces noise — multiple passes approximate a Gaussian blur
3. **K-means clustering** with **k-means++ initialisation** assigns every pixel to one of exactly *k* color centroids, converging until < 0.1% of pixels change assignment

### Phase 2 — Edge Sharpen

4. **Mode filter** (majority vote in a local window) replaces each pixel's cluster assignment with the most common cluster in its neighbourhood. Multiple passes progressively straighten wavy, gradient-following contour edges into blocky, crisp boundaries

### Post-processing

5. **Flood-fill region labelling** identifies connected same-color components
6. **Small-region merging** absorbs tiny regions into their largest neighbour
7. **Nearest-neighbor upscale** back to the original resolution preserves the crisp block edges

## Features

- **Realtime feedback** — sliders auto-trigger reprocessing with a 350ms debounce; previous in-flight workers are aborted immediately via `AbortSignal`
- **Auto-process on upload** — segmentation runs with default settings as soon as an image is loaded
- **Non-blocking UI** — processing happens in a Web Worker with a thin progress bar; the canvas stays interactive
- **Compare mode** — hold the "Compare" button to see the original image overlaid on the result
- **High-res export** — exports at original resolution using nearest-neighbor upscale
- **Large image support** — canvas dimensions capped at 4096px for browser stability; processing downscales independently

## Parameters

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| **Number of Colors (k)** | 10 | 2–32 | Exact number of output colors via k-means |
| **Blur Radius** | 6 | 0–20 | Box blur kernel radius for noise reduction |
| **Blur Passes** | 3 | 1–6 | Blur iterations (3 ≈ Gaussian) |
| **Edge Sharpening Radius** | 3 | 1–8 | Mode filter window radius |
| **Edge Sharpening Passes** | 3 | 1–8 | Mode filter iterations (more = blockier) |
| **Min Region Size** | 100 | 1–500 | Regions smaller than this are merged |
| **Max Processing Size** | 512 | 256–1024 | Image downscaled to this for processing |

## Getting Started

### Prerequisites

- Node.js 18+
- npm 9+

### Install & Run

```bash
# Clone the repository
git clone <repo-url> terseg
cd terseg

# Install dependencies
npm install

# Start dev server
npm run dev
```

Open `http://localhost:5173` in your browser.

### Build for Production

```bash
npm run build
npm run preview   # preview the production build locally
```

Output is written to `dist/`.

## Project Structure

```
terseg/
├── index.html                  # Entry HTML
├── main.js                     # Vue app bootstrap
├── App.vue                     # Main layout — viewport, toolbar, canvas, pan/zoom
├── style.css                   # Global styles + Tailwind import
├── vite.config.js              # Vite + Vue + Tailwind plugins
├── package.json
├── components/
│   ├── Header.vue              # App header with branding
│   └── ControlPanel.vue        # Sidebar with parameter sliders
└── services/
    └── segmentation.js         # Segmentation engine (inline Web Worker + sync fallback)
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | Vue 3 (Composition API, `<script setup>`) |
| Build | Vite 6 |
| Styling | Tailwind CSS 4 |
| Icons | Lucide Vue Next |
| Processing | Web Workers (inline blob URL) |
| Canvas | HTML5 Canvas + OffscreenCanvas |

## Browser Support

Requires a modern browser with support for:

- Web Workers
- OffscreenCanvas
- ES2020+ (`**` operator, optional chaining, nullish coalescing)
- Pointer Events API

Tested in Chrome, Firefox, and Edge.

## License

MIT
