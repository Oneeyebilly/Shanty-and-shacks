# Shanty-and-shacks
Real-estate listing 
/* script.js
   - Two modes:
     1) proxy mode (recommended): set API_MODE = "proxy" and set API_BASE to your backend proxy URL (e.g. https://api.yoursite.com)
     2) direct mode (dev/testing only): set API_MODE = "direct" and fill RAPIDAPI_KEY and API_HOST below (not recommended for production)
   - Paste your PayPal.me base link (without the amount) into PAYPAL_ME_BASE.
     Example: https://paypal.me/WilliamEaton5446
     The script will append /50 to make $50 automatically.
*/

const CONFIG = {
  API_MODE: "proxy", // "proxy" or "direct"
  API_BASE: "",      // e.g. "https://api.yourdomain.com" (used when API_MODE === "proxy")
  // Direct RapidAPI settings (dev only — exposes your key; not recommended for production)
  RAPIDAPI_KEY: "",           // paste your RapidAPI key here ONLY for quick testing (not for public sites)
  RAPIDAPI_HOST: "zillow56.p.rapidapi.com", // example host (Zillow via RapidAPI)
  // PayPal: paste your PayPal.me base link here (no trailing slash)
  // Example: "https://paypal.me/WilliamEaton22928"
  PAYPAL_ME_BASE: "",

  // application fee (USD)
  APPLICATION_FEE: 50.00
};

// ---------- Helper: UI ----------
const elements = {
  locationInput: document.getElementById('locationInput'),
  listingType: document.getElementById('listingType'),
  minBeds: document.getElementById('minBeds'),
  searchBtn: document.getElementById('searchBtn'),
  clearBtn: document.getElementById('clearBtn'),
  results: document.getElementById('results'),
  status: document.getElementById('status'),
  year: document.getElementById('year')
};
elements.year.textContent = new Date().getFullYear();

elements.searchBtn.addEventListener('click', doSearch);
elements.clearBtn.addEventListener('click', () => {
  elements.locationInput.value = '';
  elements.listingType.value = '';
  elements.minBeds.value = '';
  elements.results.innerHTML = '';
  elements.status.textContent = '';
});

// ---------- Main search ----------
async function doSearch(){
  const q = (elements.locationInput.value || '').trim();
  if(!q){ elements.status.textContent = 'Enter a city, state, or ZIP to search.'; return; }
  elements.status.textContent = 'Searching…';
  elements.results.innerHTML = '';

  try {
    let data;
    if (CONFIG.API_MODE === 'proxy') {
      if (!CONFIG.API_BASE) { elements.status.textContent = 'Set CONFIG.API_BASE to your backend URL.'; return; }
      const url = `${CONFIG.API_BASE.replace(/\/$/,'')}/api/search?q=${encodeURIComponent(q)}&type=${encodeURIComponent(elements.listingType.value)}&limit=60`;
      const r = await fetch(url);
      if (!r.ok) {
        const err = await r.json().catch(()=>({error:r.statusText}));
        elements.status.textContent = 'API search failed: ' + (err.error || r.statusText);
        return;
      }
      const json = await r.json();
      data = json.data || [];
      elements.status.textContent = `Found ${data.length} (via ${json.source || 'proxy'})`;
    } else {
      // direct RapidAPI call (dev only) — NOT recommended for production
      if (!CONFIG.RAPIDAPI_KEY) { elements.status.textContent = 'Set CONFIG.RAPIDAPI_KEY for direct mode.'; return; }
      const host = CONFIG.RAPIDAPI_HOST;
      const directUrl = `https://${host}/search?location=${encodeURIComponent(q)}&limit=60`;
      const r = await fetch(directUrl, {
        method: 'GET',
        headers: {
          'X-RapidAPI-Key': CONFIG.RAPIDAPI_KEY,
          'X-RapidAPI-Host': host,
          'Accept': 'application/json'
        }
      });
      if (!r.ok) {
        const err = await r.json().catch(()=>({error:r.statusText}));
        elements.status.textContent = 'Provider search failed: ' + (err.error || r.statusText);
        return;
      }
      const json = await r.json();
      // try to normalize provider data
      data = normalizeProviderData(json);
      elements.status.textContent = `Found ${data.length} (direct)`;
    }

    // Filter by beds if selected
    const minBeds = parseInt(elements.minBeds.value) || 0;
    if (minBeds) data = data.filter(it => (it.beds || 0) >= minBeds);

    renderResults(data);
  } catch (err) {
    console.error(err);
    elements.status.textContent = 'Search error: ' + (err.message || String(err));
  }
}

// ---------- Render ----------
function renderResults(items){
  if (!items || items.length === 0) {
    elements.results.innerHTML = `<p class="muted">No listings found for that location.</p>`;
    return;
  }
  elements.results.innerHTML = '';
  items.forEach(item => {
    const img = (item.images && item.images[0]) || 'https://via.placeholder.com/800x450?text=No+Image';
    const priceLabel = item.price ? Number(item.price).toLocaleString(undefined,{style:'currency',currency:'USD',maximumFractionDigits:0}) : '';
    const card = document.createElement('article');
    card.className = 'card';
    card.innerHTML = `
      <div class="thumb" style="background-image:url('${escapeHtml(img)}')"></div>
      <div class="body">
        <div>
          <div style="display:flex;justify-content:space-between;align-items:center">
            <div>
              <div style="font-weight:700">${escapeHtml(item.title || item.address || 'Listing')}</div>
              <div class="muted">${escapeHtml(item.address || '')}</div>
            </div>
            <div class="price">${priceLabel}</div>
          </div>
          <div class="meta">${item.beds ? item.beds + ' bd • ' : ''}${item.baths ? item.baths + ' ba' : ''}</div>
        </div>
        <div class="actions">
          <button class="btn primary applyNow" data-id="${escapeHtml(item.id || '')}">Apply Now — $${Number(CONFIG.APPLICATION_FEE).toFixed(2)}</button>
          <a class="btn ghost" target="_blank" rel="noopener" href="${buildPayPalLink(CONFIG.APPLICATION_FEE, item)}">Pay $${Number(CONFIG.APPLICATION_FEE).toFixed(2)}</a>
        </div>
      </div>
    `;
    elements.results.appendChild(card);
  });

  // attach apply button handlers
  document.querySelectorAll('.applyNow').forEach(btn=>{
    btn.addEventListener('click', (e)=>{
      const id = e.currentTarget.dataset.id || 'unknown';
      // For tracking you may include id/title in the PayPal note or custom fields (server-side is safer)
      const note = `Application ${id}`;
      // open PayPal
      const url = buildPayPalLink(CONFIG.APPLICATION_FEE, { title: note });
      window.open(url, '_blank', 'noopener');
    });
  });
}

// ---------- PayPal link builder ----------
/*
  buildPayPalLink(amount, item):
  - If CONFIG.PAYPAL_ME_BASE is set (e.g. https://paypal.me/YourName), we append /amount
    Example result: https://paypal.me/YourName/50
  - If not set, we build a standard PayPal buy-now URL that uses no business email (works if PayPal is configured)
    For production you should use your paypal business email or server-side checkout for tracking.
*/
function buildPayPalLink(amount, item = {}) {
  const amt = Number(amount || CONFIG.APPLICATION_FEE).toFixed(2);
  const title = item.title || item.address || 'Application Fee';
  if (CONFIG.PAYPAL_ME_BASE) {
    // Ensure no trailing slash
    const base = CONFIG.PAYPAL_ME_BASE.replace(/\/$/, '');
    // Many clients accept integer amounts for paypal.me — we'll pass an integer if possible
    const amtUrl = (Number(amt) % 1 === 0) ? String(parseInt(amt, 10)) : amt;
    // Optional note — paypal.me doesn't reliably accept note param for all clients; we prioritize amount.
    return `${base}/${amtUrl}`;
  } else {
    // Build PayPal buy-now link (note: you should include your business email in production)
    // We leave business blank so PayPal may prompt or rely on your PayPal cookies
    const params = new URLSearchParams({
      cmd: "_xclick",
      business: "", // put your PayPal business email here if you prefer: e.g., "you@yourdomain.com"
      item_name: title,
      amount: amt,
      currency_code: "USD",
      no_note: 1
    });
    return `https://www.paypal.com/cgi-bin/webscr?${params.toString()}`;
  }
}

// ---------- Provider data normalization (best-effort) ----------
function normalizeProviderData(payload) {
  if (!payload) return [];
  // If payload is already normalized array
  if (Array.isArray(payload)) return payload.map(mapRaw);
  // common RapidAPI zillow56 shape: try payload.results or payload.data
  if (Array.isArray(payload.results)) return payload.results.map(mapRaw);
  if (Array.isArray(payload.listings)) return payload.listings.map(mapRaw);
  if (Array.isArray(payload.data)) return payload.data.map(mapRaw);
  // find first array inside
  const candidate = Object.values(payload).find(v => Array.isArray(v) && v.length);
  if (candidate) return candidate.map(mapRaw);
  return [];
}

function mapRaw(x = {}) {
  // Map common fields from various providers to our expected schema
  return {
    id: x.id || x.zpid || x.listing_id || x.property_id || '',
    title: x.title || x.headline || (x.address && x.address.line) || '',
    address: (x.address && (x.address.line || x.address.formatted)) || x.displayAddress || x.location || '',
    price: x.price || x.list_price || x.rent || 0,
    beds: x.bedrooms || x.beds || (x.property && x.property.bedrooms) || '',
    baths: x.bathrooms || x.baths || (x.property && x.property.bathrooms) || '',
    images: x.images || x.photos || x.media?.photos || (x.image ? [x.image] : []),
  };
}

// ---------- Utility helpers ----------
function escapeHtml(s=''){ return String(s).replace(/[&<>"']/g, c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c])); }:root{
  --accent:#2463ff; --muted:#6b7280; --bg:#fff;
  font-family:system-ui, -apple-system, "Segoe UI", Roboto, Arial;
  color:#0f172a;
}
*{box-sizing:border-box}
body{margin:0;background:var(--bg);-webkit-font-smoothing:antialiased}
.container{max-width:1100px;margin:0 auto;padding:18px}
.skip-link{position:absolute;left:1rem;top:-40px;background:#111827;color:white;padding:.5rem .75rem;border-radius:6px}
.skip-link:focus{top:1rem}
.site-header{border-bottom:1px solid #eef4fb;padding:12px 0}
.muted{color:var(--muted);font-size:.95rem}
.controls{display:flex;gap:10px;flex-wrap:wrap;align-items:center;margin:16px 0}
.controls input, .controls select{padding:8px;border-radius:8px;border:1px solid #eef4fb}
button{padding:8px 12px;border-radius:8px;border:0;background:var(--accent);color:white;cursor:pointer}
button.ghost{background:transparent;border:1px solid #eef4fb;color:var(--muted)}
.results{display:grid;grid-template-columns:repeat(auto-fit,minmax(300px,1fr));gap:12px;margin-top:12px}
.card{border:1px solid #eef4fb;border-radius:10px;overflow:hidden;background:white;display:flex;flex-direction:column}
.thumb{height:180px;background-size:cover;background-position:center}
.body{padding:12px;display:flex;flex-direction:column;gap:8px;flex:1}
.price{font-weight:700;color:var(--accent)}
.meta{color:var(--muted);font-size:.95rem}
.actions{display:flex;gap:8px;margin-top:auto}
.btn{padding:8px 10px;border-radius:8px;text-decoration:none;border:0;font-weight:600}
.btn.primary{background:var(--accent);color:white}
.btn.ghost{background:transparent;border:1px solid #e6eef8}
@media (max-width:640px){
  .controls{flex-direction:column;align-items:stretch}
  .controls label{width:100%}
}<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Nationwide Listings — YourAgency</title>
  <meta name="description" content="Search nationwide real estate listings and apply with a $50 application fee." />
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <a class="skip-link" href="#main">Skip to content</a>
  <header class="site-header">
    <div class="container">
      <h1>YourAgency — Nationwide Listings</h1>
      <p class="muted">Search any city or ZIP. Every listing includes an "Apply Now — $50" PayPal button.</p>
    </div>
  </header>

  <main id="main" class="container">
    <section class="controls">
      <label>
        Location (city, state or ZIP):
        <input id="locationInput" placeholder="e.g. Seattle, WA or 98101" />
      </label>

      <label>
        Show:
        <select id="listingType">
          <option value="">All</option>
          <option value="rent">Rentals</option>
          <option value="sale">For Sale</option>
        </select>
      </label>

      <label>
        Min beds:
        <select id="minBeds">
          <option value="">Any</option>
          <option value="1">1+</option>
          <option value="2">2+</option>
          <option value="3">3+</option>
        </select>
      </label>

      <button id="searchBtn" class="btn primary">Search</button>
      <button id="clearBtn" class="btn ghost">Clear</button>
      <span id="status" class="muted" aria-live="polite"></span>
    </section>

    <section id="results" class="results" aria-live="polite"></section>
  </main>

  <footer class="site-footer container">
    <small>© <span id="year"></span> YourAgency — Read the README for legal & deployment notes.</small>
  </footer>

  <script src="script.js"></script>
</body>
</html>
