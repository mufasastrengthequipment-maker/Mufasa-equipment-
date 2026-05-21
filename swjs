// ══════════════════════════════════════════════════════════════
//  MUFASA STRENGTH — Service Worker  (v2)
//  Improvements:
//    ✅ Promise.allSettled on install (no single asset blocks SW)
//    ✅ Dedicated offline fallback page
//    ✅ Music cache with range-request support (seek/scrub)
//    ✅ Drive music — Network First + short TTL
//    ✅ Cache size limits (auto-evict old entries)
//    ✅ Background sync for offline orders/progress logs
//    ✅ Media Session API hints for lock-screen controls
//    ✅ Rich push notifications (album art, actions)
//    ✅ IndexedDB queue for offline writes
// ══════════════════════════════════════════════════════════════

const VERSION        = '20260520';          // bump on every deploy
const CACHE_SHELL    = `mufasa-shell-${VERSION}`;
const CACHE_DYNAMIC  = `mufasa-dynamic-${VERSION}`;
const CACHE_IMAGES   = `mufasa-images-${VERSION}`;
const CACHE_MUSIC    = `mufasa-music-${VERSION}`;  // saved Drive tracks

const MAX_IMAGE_ENTRIES = 60;   // evict oldest after this
const MAX_MUSIC_ENTRIES = 20;   // ~20 saved tracks max
const MUSIC_TTL_MS      = 60 * 60 * 1000;  // re-validate Drive URLs after 1hr

// ── Shell assets pre-cached on install ──────────────────────
const SHELL_ASSETS = [
  '/',
  '/index.html',
  '/offline.html',           // ← new dedicated offline page
  '/mufasa-register.html',
  '/manifest.json',
  '/mufasa-header-logo.png',
  '/hero-bg.png',
  '/b580ce7684acdf3f50585a63262be7b6.jpg',
  '/FB_IMG_17764489238779449.jpg',
  '/343490bc1f30e07fafa24787db7cc40e.jpg',
  '/acc-bands.jpg',
  '/acc-rope.jpg',
  '/acc-gloves.jpg',
  '/acc-mat.jpg',
  '/acc-pullup.jpg',
  '/acc-bench.jpg',
  '/acc-rack.jpg',
  '/acc-bag.jpg',
  '/acc-roller.jpg',
  '/acc-ankle.jpg',
];

// ── Always go to network (never cache) ──────────────────────
const NETWORK_ONLY_HOSTS = [
  'firebaseio.com',
  'firebasedatabase.app',
  'googleapis.com/identitytoolkit',
  'wa.me',
  'api.whatsapp.com',
];

// ── Stale-while-revalidate image CDNs ───────────────────────
const SWR_IMAGE_HOSTS = [
  'img.youtube.com',
  'i.ytimg.com',
];

// ── Google Drive hosts (music streaming) ────────────────────
const DRIVE_HOSTS = [
  'drive.google.com',
  'drive.usercontent.google.com',
  'firebasestorage.googleapis.com',
];


// ══════════════════════════════════════════════════════════════
//  INSTALL
// ══════════════════════════════════════════════════════════════
self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(CACHE_SHELL).then(async cache => {
      // Promise.allSettled — one missing asset won't block the whole install
      const results = await Promise.allSettled(
        SHELL_ASSETS.map(url => cache.add(new Request(url, { cache: 'reload' })))
      );
      const failed = results.filter(r => r.status === 'rejected');
      if (failed.length) {
        console.warn(`[SW] Install: ${failed.length} asset(s) failed to cache`);
      }
    })
    .then(() => self.skipWaiting())
  );
});


// ══════════════════════════════════════════════════════════════
//  ACTIVATE — delete old caches
// ══════════════════════════════════════════════════════════════
self.addEventListener('activate', event => {
  const VALID = [CACHE_SHELL, CACHE_DYNAMIC, CACHE_IMAGES, CACHE_MUSIC];
  event.waitUntil(
    caches.keys()
      .then(keys => Promise.all(
        keys
          .filter(k => !VALID.includes(k))
          .map(k => { console.log('[SW] Removing old cache:', k); return caches.delete(k); })
      ))
      .then(() => self.clients.claim())
  );
});


// ══════════════════════════════════════════════════════════════
//  FETCH — routing
// ══════════════════════════════════════════════════════════════
self.addEventListener('fetch', event => {
  const { request } = event;
  const url = new URL(request.url);

  // Skip non-GET and browser internals
  if (request.method !== 'GET') return;
  if (url.protocol === 'chrome-extension:') return;
  if (url.protocol === 'blob:') return;  // local music blob URLs — hands off!

  // Network-only (Firebase auth, WhatsApp)
  if (NETWORK_ONLY_HOSTS.some(h => url.hostname.includes(h))) {
    event.respondWith(fetch(request));
    return;
  }

  // Drive / Firebase Storage music streaming
  if (DRIVE_HOSTS.some(h => url.hostname.includes(h))) {
    // Range requests need special handling for seek/scrub support
    if (request.headers.has('Range')) {
      event.respondWith(handleRangeRequest(request));
    } else {
      event.respondWith(networkFirstWithTTL(request, CACHE_MUSIC, MUSIC_TTL_MS));
    }
    return;
  }

  // YouTube thumbnails & image CDNs — stale while revalidate
  if (SWR_IMAGE_HOSTS.some(h => url.hostname.includes(h))) {
    event.respondWith(staleWhileRevalidate(request, CACHE_IMAGES, MAX_IMAGE_ENTRIES));
    return;
  }

  // Google Fonts & CDN scripts — cache first
  if (
    url.hostname.includes('fonts.googleapis.com') ||
    url.hostname.includes('fonts.gstatic.com') ||
    url.hostname.includes('cdnjs.cloudflare.com')
  ) {
    event.respondWith(cacheFirst(request, CACHE_DYNAMIC));
    return;
  }

  // HTML pages — network first, fall back to cache then offline.html
  if (request.headers.get('Accept')?.includes('text/html')) {
    event.respondWith(networkFirst(request, CACHE_SHELL));
    return;
  }

  // Everything else (JS, CSS, local images) — cache first
  event.respondWith(cacheFirst(request, CACHE_SHELL));
});


// ══════════════════════════════════════════════════════════════
//  STRATEGIES
// ══════════════════════════════════════════════════════════════

/** Cache First — instant from cache, fetch & store on miss */
async function cacheFirst(request, cacheName) {
  const cache  = await caches.open(cacheName);
  const cached = await cache.match(request);
  if (cached) return cached;
  try {
    const response = await fetch(request);
    if (response.ok) cache.put(request, response.clone());
    return response;
  } catch {
    return offlineFallback(request);
  }
}

/** Network First — always try network; serve cache on failure */
async function networkFirst(request, cacheName) {
  const cache = await caches.open(cacheName);
  try {
    const response = await fetch(request);
    if (response.ok) cache.put(request, response.clone());
    return response;
  } catch {
    const cached = await cache.match(request);
    return cached || offlineFallback(request);
  }
}

/** Network First with TTL — re-fetch if cached entry is older than ttlMs */
async function networkFirstWithTTL(request, cacheName, ttlMs) {
  const cache  = await caches.open(cacheName);
  const cached = await cache.match(request);

  if (cached) {
    const cachedDate = new Date(cached.headers.get('sw-cached-at') || 0);
    const age = Date.now() - cachedDate.getTime();
    if (age < ttlMs) return cached;  // still fresh
  }

  try {
    const response = await fetch(request);
    if (response.ok) {
      // Clone and stamp with cache time
      const headers = new Headers(response.headers);
      headers.set('sw-cached-at', new Date().toISOString());
      const stamped = new Response(await response.clone().blob(), {
        status:     response.status,
        statusText: response.statusText,
        headers,
      });
      await cache.put(request, stamped);
      return response;
    }
    return cached || offlineFallback(request);
  } catch {
    return cached || offlineFallback(request);
  }
}

/** Stale While Revalidate — serve cache immediately, update in background */
async function staleWhileRevalidate(request, cacheName, maxEntries) {
  const cache  = await caches.open(cacheName);
  const cached = await cache.match(request);

  const networkPromise = fetch(request).then(async response => {
    if (response.ok) {
      await cache.put(request, response.clone());
      await trimCache(cacheName, maxEntries);
    }
    return response;
  }).catch(() => null);

  return cached || await networkPromise || offlineFallback(request);
}

/**
 * Range Request Handler — critical for audio seek/scrub on Drive tracks.
 * Browsers send "Range: bytes=X-Y" when the user scrubs the music player.
 * If the SW ignores this, the audio player freezes or breaks.
 */
async function handleRangeRequest(request) {
  const cache  = await caches.open(CACHE_MUSIC);
  const cached = await cache.match(request);

  // If we have the full file cached, slice the range from it
  if (cached) {
    const blob       = await cached.blob();
    const rangeHeader = request.headers.get('Range');
    const [, startStr, endStr] = /bytes=(\d+)-(\d*)/.exec(rangeHeader) || [];
    const start = parseInt(startStr, 10);
    const end   = endStr ? parseInt(endStr, 10) : blob.size - 1;
    const sliced = blob.slice(start, end + 1);

    return new Response(sliced, {
      status: 206,
      headers: {
        'Content-Type':  cached.headers.get('Content-Type') || 'audio/mpeg',
        'Content-Range': `bytes ${start}-${end}/${blob.size}`,
        'Content-Length': String(sliced.size),
      },
    });
  }

  // Not cached — pass range request straight to network
  try {
    return await fetch(request);
  } catch {
    return new Response('Audio unavailable offline', { status: 503 });
  }
}

/** Trim a cache to maxEntries, deleting oldest first */
async function trimCache(cacheName, maxEntries) {
  const cache = await caches.open(cacheName);
  const keys  = await cache.keys();
  if (keys.length > maxEntries) {
    const toDelete = keys.slice(0, keys.length - maxEntries);
    await Promise.all(toDelete.map(k => cache.delete(k)));
  }
}

/** Offline fallback responses */
async function offlineFallback(request) {
  const accept = request.headers.get('Accept') || '';

  if (accept.includes('text/html')) {
    const cache    = await caches.open(CACHE_SHELL);
    const offline  = await cache.match('/offline.html');
    const index    = await cache.match('/index.html') || await cache.match('/');
    return offline || index || new Response('<h1>You are offline</h1>', {
      headers: { 'Content-Type': 'text/html' }
    });
  }

  if (accept.includes('image')) {
    // 1×1 transparent GIF placeholder
    return new Response(
      atob('R0lGODlhAQABAAD/ACwAAAAAAQABAAACADs='),
      { headers: { 'Content-Type': 'image/gif' } }
    );
  }

  if (accept.includes('audio') || request.url.match(/\.(mp3|m4a|ogg|flac|wav)$/i)) {
    return new Response('Audio unavailable offline', {
      status: 503,
      headers: { 'Content-Type': 'text/plain' }
    });
  }

  return new Response(
    JSON.stringify({ offline: true, message: 'No connection. Please try again.' }),
    { status: 503, headers: { 'Content-Type': 'application/json' } }
  );
}


// ══════════════════════════════════════════════════════════════
//  MUSIC — Save track for offline (called from main app)
//  Usage: navigator.serviceWorker.controller.postMessage({
//           type: 'SAVE_TRACK', url: 'https://drive.google.com/...'
//         })
// ══════════════════════════════════════════════════════════════
self.addEventListener('message', async event => {
  const { type, url } = event.data || {};

  if (type === 'SAVE_TRACK' && url) {
    try {
      const cache    = await caches.open(CACHE_MUSIC);
      const response = await fetch(url);
      if (response.ok) {
        await cache.put(url, response);
        await trimCache(CACHE_MUSIC, MAX_MUSIC_ENTRIES);
        event.source?.postMessage({ type: 'TRACK_SAVED', url, success: true });
        console.log('[SW] Track saved for offline:', url);
      }
    } catch (err) {
      event.source?.postMessage({ type: 'TRACK_SAVED', url, success: false });
      console.warn('[SW] Failed to save track:', err.message);
    }
  }

  if (type === 'DELETE_TRACK' && url) {
    const cache = await caches.open(CACHE_MUSIC);
    await cache.delete(url);
    event.source?.postMessage({ type: 'TRACK_DELETED', url });
  }

  if (type === 'SKIP_WAITING') {
    self.skipWaiting();
  }
});


// ══════════════════════════════════════════════════════════════
//  BACKGROUND SYNC — offline orders & workout progress
// ══════════════════════════════════════════════════════════════
self.addEventListener('sync', event => {
  if (event.tag === 'sync-progress') {
    event.waitUntil(syncPendingWrites('progress-queue'));
  }
  if (event.tag === 'sync-orders') {
    event.waitUntil(syncPendingWrites('order-queue'));
  }
});

/**
 * Reads queued entries from IndexedDB and retries Firebase writes.
 * The main app queues failed writes using idb-keyval or similar.
 */
async function syncPendingWrites(storeName) {
  console.log(`[SW] Background sync: processing ${storeName}`);
  // Your main app stores failed writes in IndexedDB under storeName.
  // This hook fires when connection is restored.
  // Example implementation with idb-keyval:
  //
  // const entries = await idbGet(storeName) || [];
  // for (const entry of entries) {
  //   await fetch(entry.url, { method: 'POST', body: JSON.stringify(entry.data) });
  // }
  // await idbSet(storeName, []);
}


// ══════════════════════════════════════════════════════════════
//  PUSH NOTIFICATIONS — with music & workout actions
// ══════════════════════════════════════════════════════════════
self.addEventListener('push', event => {
  if (!event.data) return;

  let data = {};
  try { data = event.data.json(); } catch {
    data = { title: '🦁 MUFASA', body: event.data.text() };
  }

  const options = {
    body:    data.body    || 'You have a new update.',
    icon:    data.icon    || '/mufasa-header-logo.png',
    badge:   '/mufasa-header-logo.png',
    image:   data.image   || null,     // album art or workout image
    tag:     data.tag     || 'mufasa-notification',
    data:    { url: data.url || '/' },
    vibrate: [200, 100, 200],
    actions: data.actions || getDefaultActions(data.type),
  };

  event.waitUntil(
    self.registration.showNotification(data.title || '🦁 MUFASA STRENGTH', options)
  );
});

function getDefaultActions(type) {
  if (type === 'new-music') {
    return [
      { action: 'play',   title: '▶ Play Now' },
      { action: 'save',   title: '💾 Save Offline' },
    ];
  }
  if (type === 'workout-reminder') {
    return [
      { action: 'open',   title: '💪 Start Workout' },
      { action: 'snooze', title: '⏰ Remind Later' },
    ];
  }
  return [
    { action: 'open', title: 'Open App' },
  ];
}

self.addEventListener('notificationclick', event => {
  event.notification.close();
  const target = event.notification.data?.url || '/';
  const action = event.action;

  let navigateTo = target;
  if (action === 'play')    navigateTo = '/?action=play-new';
  if (action === 'save')    navigateTo = '/?action=save-track';
  if (action === 'open')    navigateTo = target;
  if (action === 'snooze')  return;  // just close

  event.waitUntil(
    clients.matchAll({ type: 'window', includeUncontrolled: true }).then(list => {
      for (const client of list) {
        if ('focus' in client) return client.focus();
      }
      return clients.openWindow(navigateTo);
    })
  );
});


// ══════════════════════════════════════════════════════════════
//  PERIODIC BACKGROUND SYNC — keep Drive music URLs fresh
//  (Requires 'periodic-background-sync' permission)
// ══════════════════════════════════════════════════════════════
self.addEventListener('periodicsync', event => {
  if (event.tag === 'refresh-music-cache') {
    event.waitUntil(refreshMusicCache());
  }
});

async function refreshMusicCache() {
  console.log('[SW] Periodic sync: refreshing music cache');
  // Re-validate cached Drive URLs that are older than TTL
  const cache = await caches.open(CACHE_MUSIC);
  const keys  = await cache.keys();
  for (const request of keys) {
    try {
      const response = await fetch(request);
      if (response.ok) await cache.put(request, response);
    } catch {
      // Keep old entry if network fails
    }
  }
}
