<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>My App</title>
<meta id="theme" name="theme-color" content="#2f5bff">
<meta id="appleTitle" name="apple-mobile-web-app-title" content="My App">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="mobile-web-app-capable" content="yes">
<link id="mf" rel="manifest">
<link id="ico" rel="icon" type="image/png">
<link id="apple" rel="apple-touch-icon">
<style>
:root{--bg:#eef1f5;--panel:#fff;--ink:#1a2230;--mute:#5d6878;--line:#d5dbe4;--accent:#2f5bff;color-scheme:light dark}
@media (prefers-color-scheme:dark){:root{--bg:#10141b;--panel:#171d27;--ink:#e8ecf3;--mute:#98a3b5;--line:#2a3342;--accent:#7b97ff}}
*{box-sizing:border-box}
html,body{margin:0}
body{background:var(--bg);color:var(--ink);font:16px/1.5 "Avenir Next","Segoe UI",system-ui,sans-serif;
  padding:max(24px,env(safe-area-inset-top)) 20px max(24px,env(safe-area-inset-bottom));min-height:100vh}
main{max-width:860px;margin:0 auto}
h1{font-size:clamp(1.9rem,5vw,2.8rem);line-height:1.1;margin:0 0 .4rem;letter-spacing:-.02em}
.lead{color:var(--mute);margin:0 0 2rem;max-width:56ch}
.grid{display:grid;grid-template-columns:1fr 280px;gap:20px;align-items:start}
@media (max-width:720px){.grid{grid-template-columns:1fr}}
.panel{background:var(--panel);border:1px solid var(--line);border-radius:14px;padding:22px}
label{display:block;font-weight:600;margin:0 0 6px}
.field{margin-bottom:20px}
input[type=text]{width:100%;padding:10px 12px;border:1px solid var(--line);border-radius:8px;background:transparent;color:inherit;font:inherit}
input[type=color]{width:56px;height:40px;padding:2px;border:1px solid var(--line);border-radius:8px;background:transparent}
.drop{display:block;border:2px dashed var(--line);border-radius:10px;padding:22px;text-align:center;color:var(--mute);cursor:pointer;font-weight:400}
.drop:hover,.drop:focus-within{border-color:var(--accent);color:var(--ink)}
.drop input{position:absolute;opacity:0;width:1px;height:1px}
.hint{color:var(--mute);font-size:.9rem;margin:6px 0 0}
.row{display:flex;gap:10px;flex-wrap:wrap;align-items:center}
button{font:inherit;font-weight:600;padding:10px 18px;border-radius:8px;border:1px solid var(--accent);cursor:pointer}
.primary{background:var(--accent);color:#fff}
.ghost{background:transparent;color:var(--accent)}
button:focus-visible,input:focus-visible,.drop:focus-within{outline:3px solid var(--accent);outline-offset:2px}
#msg{min-height:1.5em;margin:12px 0 0;color:var(--mute)}
.warn{border-left:4px solid var(--accent);padding:8px 12px;margin:0 0 16px;background:rgba(47,91,255,.08);border-radius:0 8px 8px 0}
.preview{text-align:center}
.tile{width:160px;height:160px;margin:4px auto 14px;border-radius:36px;display:grid;place-items:center;
  background:#2f5bff;box-shadow:0 8px 24px rgba(0,0,0,.18)}
.tile img{width:76%;height:76%;object-fit:contain}
.preview p{margin:0;font-weight:600}
.preview small{color:var(--mute)}
</style>
</head>
<body>
<main>
  <h1>Choose your app logo</h1>
  <p class="lead">Upload a logo, save it, then install this page as an app. The installed app uses the icon shown on the right.</p>
  <p class="warn" id="fileWarn" hidden>Installing needs this page served over http(s), not opened as a file. In this folder run <code>python -m http.server 8000</code>, then open <code>http://localhost:8000</code>.</p>

  <div class="grid">
    <form id="f" class="panel" autocomplete="off">
      <div class="field">
        <label for="logo">Logo image</label>
        <label class="drop" for="logo" id="dropText"><span id="dropLabel">Choose a PNG, JPG or WebP (square works best)</span>
          <input type="file" id="logo" accept="image/png,image/jpeg,image/webp">
        </label>
      </div>
      <div class="field">
        <label for="name">App name</label>
        <input type="text" id="name" maxlength="45">
      </div>
      <div class="field">
        <label for="color">Icon background</label>
        <div class="row"><input type="color" id="color">
          <span class="hint" style="margin:0">Fills the icon on iPhone and in round or squircle Android icons.</span></div>
      </div>
      <div class="row">
        <button class="primary" type="submit">Save logo</button>
        <button class="ghost" id="installBtn" type="button" hidden>Install app</button>
      </div>
      <p id="msg" role="status"></p>
      <p class="hint">No Install button? In Chrome or Edge use the install icon in the address bar. On iPhone tap Share, then Add to Home Screen.</p>
    </form>

    <aside class="panel preview" aria-label="Icon preview">
      <div class="tile" id="tile"><img id="prev" alt="Current app icon"></div>
      <p id="prevName"></p>
      <small>Home screen preview</small>
    </aside>
  </div>
</main>

<script>
const $ = id => document.getElementById(id);
const KEY = 'logoApp.v1';
const state = { name: 'My App', color: '#2f5bff', logo: null }; // logo = data URL, or null for the default
try { Object.assign(state, JSON.parse(localStorage.getItem(KEY) || '{}')); } catch (e) {}
let pending = null; // newly chosen logo, not saved yet
let manifestUrl = null;

const loadImg = src => new Promise((ok, fail) => {
  const i = new Image(); i.onload = () => ok(i); i.onerror = fail; i.src = src;
});

// Draw the logo centred inside a square, with padding and an optional solid background.
function compose(img, size, pad, bg) {
  const c = document.createElement('canvas'); c.width = c.height = size;
  const x = c.getContext('2d');
  if (bg) { x.fillStyle = bg; x.fillRect(0, 0, size, size); }
  const inner = size * (1 - 2 * pad);
  const k = Math.min(inner / img.width, inner / img.height);
  const w = img.width * k, h = img.height * k;
  x.imageSmoothingQuality = 'high';
  x.drawImage(img, (size - w) / 2, (size - h) / 2, w, h);
  return c.toDataURL('image/png');
}

function defaultLogo(name, color) {
  const c = document.createElement('canvas'); c.width = c.height = 512;
  const x = c.getContext('2d');
  x.fillStyle = '#fff'; x.beginPath(); x.arc(256, 256, 208, 0, Math.PI * 2); x.fill();
  x.fillStyle = color; x.font = '700 280px "Avenir Next","Segoe UI",system-ui,sans-serif';
  x.textAlign = 'center'; x.textBaseline = 'middle';
  x.fillText((name[0] || 'A').toUpperCase(), 256, 272);
  return c.toDataURL('image/png');
}

// Build the icons and a manifest from the saved settings and point the page's <link>s at them.
async function applyIdentity() {
  const src = state.logo || defaultLogo(state.name, state.color);
  const img = await loadImg(src);
  const icon192 = compose(img, 192, 0.06);
  const icon512 = compose(img, 512, 0.06);
  const maskable = compose(img, 512, 0.20, state.color); // OS crops these, so keep the logo in the safe zone
  const apple = compose(img, 180, 0.12, state.color);    // iOS fills transparency with black

  const start = location.origin + location.pathname;
  const manifest = {
    id: start, name: state.name, short_name: state.name.slice(0, 12),
    start_url: start, scope: new URL('./', start).href, display: 'standalone',
    background_color: state.color, theme_color: state.color,
    icons: [
      { src: icon192, sizes: '192x192', type: 'image/png', purpose: 'any' },
      { src: icon512, sizes: '512x512', type: 'image/png', purpose: 'any' },
      { src: maskable, sizes: '512x512', type: 'image/png', purpose: 'maskable' }
    ]
  };
  if (manifestUrl) URL.revokeObjectURL(manifestUrl);
  manifestUrl = URL.createObjectURL(new Blob([JSON.stringify(manifest)], { type: 'application/manifest+json' }));

  $('mf').href = manifestUrl;
  $('ico').href = icon192;
  $('apple').href = apple;
  $('theme').content = state.color;
  $('appleTitle').content = state.name;
  document.title = state.name;
  $('prev').src = src;
}

// ----- UI -----
$('name').value = state.name;
$('color').value = state.color;
$('tile').style.background = state.color;
$('prevName').textContent = state.name;
if (location.protocol === 'file:') $('fileWarn').hidden = false;
applyIdentity().catch(() => { $('msg').textContent = 'Could not build the icons. Try a different image.'; });

$('logo').addEventListener('change', async e => {
  const file = e.target.files[0];
  if (!file) return;
  const url = URL.createObjectURL(file);
  try {
    const img = await loadImg(url);
    const k = Math.min(1, 512 / Math.max(img.width, img.height)); // keep what we store small
    const c = document.createElement('canvas');
    c.width = Math.round(img.width * k); c.height = Math.round(img.height * k);
    c.getContext('2d').drawImage(img, 0, 0, c.width, c.height);
    pending = c.toDataURL('image/png');
    $('prev').src = pending;
    $('dropLabel').textContent = file.name;
    $('msg').textContent = '';
  } catch (err) {
    $('msg').textContent = "That file isn't a readable image. Use PNG, JPG or WebP.";
  } finally { URL.revokeObjectURL(url); }
});
$('color').addEventListener('input', e => { $('tile').style.background = e.target.value; });
$('name').addEventListener('input', e => { $('prevName').textContent = e.target.value || 'My App'; });

$('f').addEventListener('submit', e => {
  e.preventDefault();
  state.name = $('name').value.trim().slice(0, 45) || 'My App';
  state.color = $('color').value;
  if (pending) state.logo = pending;
  try {
    localStorage.setItem(KEY, JSON.stringify(state));
    location.reload(); // reload so the browser re-reads the new manifest before you install
  } catch (err) {
    $('msg').textContent = 'Could not save in this browser (storage is full or blocked).';
  }
});

let deferred;
window.addEventListener('beforeinstallprompt', e => { e.preventDefault(); deferred = e; $('installBtn').hidden = false; });
$('installBtn').addEventListener('click', async () => {
  deferred.prompt(); await deferred.userChoice; deferred = null; $('installBtn').hidden = true;
});
</script>
</body>
</html>
