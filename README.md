# awesome-lucky-wheel

<img width="1919" height="1024" alt="image" src="https://github.com/user-attachments/assets/83d424e6-b6c5-4e7e-837c-96863f221f11" />

A practical and flexible lucky wheel web app built with Vue 3 + Vite. It supports weighted draws, auto-fill, touch gestures, sound effects, history tracking, and bilingual UI.

Live demo: https://luketsengtw.github.io/awesome-lucky-wheel/

## Features

1. You can customize the **Drawer**, allowing only the Drawer to participate in the draw among multiple options.
2. You can customize a message, which will appear together in the winner announcement window.
3. You can use **Auto Fill** to automatically generate prize options by setting a prefix, suffix, and a numeric range.
4. You can use **Weighted Mode** to customize the probability of each prize during the draw.
5. You can use the **Remove Winner** feature to automatically remove the winner after they are drawn.
6. You can view the winner history, which includes the winner’s name and timestamp.
7. Accidentally clicked refresh? This website uses **localStorage** to temporarily store your information and prevent data loss when the page is refreshed.
8. The website features chill background music (BGM) along with pleasant sound effects.

## Tech Stack

- Vue 3 (Composition API, `<script setup>`)
- Vite
- canvas-confetti

## Getting Started

### Prerequisites

- Node.js 18+ (recommended)
- npm 9+

### Install

```bash
npm install
```

### Run Dev Server

```bash
npm run dev
```

### Build

```bash
npm run build
```

### Preview Build

```bash
npm run preview
```

## Usage Notes

- Options are entered one per line.
- In Weighted Mode, use `name:weight`. Example:
  - `Grand Prize:1`
  - `Small Prize:5`
  - `Thanks:10`
- Swipe up on the wheel or press `Space` to spin.
- When “Remove Winner” is enabled, the winner is removed after you close the result popup.

## Audio Assets

Place these files in [public/audio/](public/audio/):

- `bgm-idle.mp3`
- `sfx-click.mp3`
- `sfx-winner.mp3`

## Data Persistence

The app stores the following in `localStorage`:

- options list
- weighted mode on/off
- remove winner on/off
- history records
- sound/music toggle state
- language selection

Clearing site data resets these to defaults.

## Deploy to GitHub Pages

1) Set the base path in [vite.config.js](vite.config.js):

```js
export default defineConfig({
  plugins: [vue()],
  base: '/awesome-lucky-wheel/',
});
```

2) Install deploy tool:

```bash
npm i -D gh-pages
```

3) Add scripts in [package.json](package.json):

```json
{
  "scripts": {
    "build": "vite build",
    "deploy": "gh-pages -d dist"
  }
}
```

4) Deploy:

```bash
npm run build
npm run deploy
```

5) In GitHub repo settings, set Pages source to `gh-pages` / root.
