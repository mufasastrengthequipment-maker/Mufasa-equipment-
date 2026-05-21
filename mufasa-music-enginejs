// ══════════════════════════════════════════════════════════════
//  MUFASA STRENGTH — Music Engine  v2
//  Drop this script into MufasaMusic.html
//
//  Covers:
//    ✅ IndexedDB — playlist persistence across sessions
//    ✅ Last played position — resume exactly where you left off
//    ✅ Media Session API — lock screen controls (play/pause/skip)
//    ✅ "Save for offline" — tells SW to cache Drive tracks
//    ✅ Offline indicator — shows which tracks are saved
//    ✅ Shuffle & repeat modes
//    ✅ Waveform progress animation
// ══════════════════════════════════════════════════════════════

'use strict';

// ── IndexedDB Setup ──────────────────────────────────────────
const DB_NAME    = 'MufasaMusicDB';
const DB_VERSION = 2;
let   idb        = null;

function openIDB() {
  return new Promise((resolve, reject) => {
    if (idb) { resolve(idb); return; }
    const req = indexedDB.open(DB_NAME, DB_VERSION);

    req.onupgradeneeded = e => {
      const db = e.target.result;

      // Store current playlist state
      if (!db.objectStoreNames.contains('state')) {
        db.createObjectStore('state');
      }

      // Store which tracks are saved offline
      if (!db.objectStoreNames.contains('offline')) {
        db.createObjectStore('offline', { keyPath: 'src' });
      }

      // Store locally uploaded files (user's own music)
      if (!db.objectStoreNames.contains('localTracks')) {
        const ls = db.createObjectStore('localTracks', { keyPath: 'id', autoIncrement: true });
        ls.createIndex('name', 'name');
      }
    };

    req.onsuccess = e => { idb = e.target.result; resolve(idb); };
    req.onerror   = e => reject(e.target.error);
  });
}

async function idbGet(store, key) {
  const db  = await openIDB();
  const tx  = db.transaction(store, 'readonly');
  return new Promise((res, rej) => {
    const req = tx.objectStore(store).get(key);
    req.onsuccess = () => res(req.result);
    req.onerror   = () => rej(req.error);
  });
}

async function idbSet(store, key, value) {
  const db = await openIDB();
  const tx = db.transaction(store, 'readwrite');
  return new Promise((res, rej) => {
    const req = tx.objectStore(store).put(value, key);
    req.onsuccess = () => res();
    req.onerror   = () => rej(req.error);
  });
}

async function idbGetAll(store) {
  const db = await openIDB();
  const tx = db.transaction(store, 'readonly');
  return new Promise((res, rej) => {
    const req = tx.objectStore(store).getAll();
    req.onsuccess = () => res(req.result || []);
    req.onerror   = () => rej(req.error);
  });
}

async function idbDelete(store, key) {
  const db = await openIDB();
  const tx = db.transaction(store, 'readwrite');
  return new Promise((res, rej) => {
    const req = tx.objectStore(store).delete(key);
    req.onsuccess = () => res();
    req.onerror   = () => rej(req.error);
  });
}


// ── Player State ─────────────────────────────────────────────
const MusicEngine = {
  audio:        new Audio(),
  playlist:     [],       // [{title, artist, src, dur, img, cat, id, isLocal, localId}]
  currentIndex: -1,
  isPlaying:    false,
  shuffle:      false,
  repeat:       'none',   // 'none' | 'one' | 'all'
  shuffleOrder: [],
  offlineSaved: new Set(),  // set of src URLs saved offline
  localTracks:  [],         // tracks from user's own files

  // ── Init ──────────────────────────────────────────────────
  async init() {
    await this.restoreState();
    await this.loadOfflineStatus();
    await this.loadLocalTracks();
    this.setupAudioListeners();
    this.setupMediaSession();
    this.render();
    console.log('[MusicEngine] Initialised. Tracks:', this.playlist.length);
  },

  // ── State persistence ─────────────────────────────────────
  async saveState() {
    try {
      await idbSet('state', 'playerState', {
        currentIndex: this.currentIndex,
        currentTime:  this.audio.currentTime || 0,
        currentSrc:   this.audio.src || '',
        shuffle:      this.shuffle,
        repeat:       this.repeat,
        volume:       this.audio.volume,
        savedAt:      Date.now(),
      });
    } catch(e) {
      console.warn('[MusicEngine] State save failed:', e);
    }
  },

  async restoreState() {
    try {
      const state = await idbGet('state', 'playerState');
      if (!state) return;

      this.shuffle = state.shuffle || false;
      this.repeat  = state.repeat  || 'none';
      if (state.volume != null) this.audio.volume = state.volume;

      // Restore last track and position
      this._restoredIndex = state.currentIndex ?? -1;
      this._restoredTime  = state.currentTime  || 0;
      this._restoredSrc   = state.currentSrc   || '';

      console.log('[MusicEngine] State restored — last track index:', this._restoredIndex,
                  'at', Math.round(this._restoredTime) + 's');
    } catch(e) {
      console.warn('[MusicEngine] State restore failed:', e);
    }
  },

  applyRestoredState() {
    if (this._restoredIndex >= 0 && this._restoredIndex < this.playlist.length) {
      this.currentIndex = this._restoredIndex;
      const track = this.playlist[this.currentIndex];
      // Only restore if same track (URL match)
      if (track && this._restoredSrc && track.src === this._restoredSrc) {
        this.audio.src = track.src;
        this.audio.currentTime = this._restoredTime || 0;
        this.updateNowPlaying();
        console.log('[MusicEngine] Restored position at', Math.round(this._restoredTime) + 's');
      }
    }
    // Clean up restore flags
    this._restoredIndex = undefined;
    this._restoredTime  = undefined;
    this._restoredSrc   = undefined;
  },

  // ── Load playlist from external source ───────────────────
  loadPlaylist(tracks) {
    this.playlist = tracks.map((t, i) => ({ ...t, _listIndex: i }));
    this.buildShuffleOrder();
    this.applyRestoredState();
    this.render();
  },

  // ── Shuffle order ─────────────────────────────────────────
  buildShuffleOrder() {
    this.shuffleOrder = [...Array(this.playlist.length).keys()];
    if (this.shuffle) {
      for (let i = this.shuffleOrder.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [this.shuffleOrder[i], this.shuffleOrder[j]] = [this.shuffleOrder[j], this.shuffleOrder[i]];
      }
    }
  },

  // ── Playback controls ─────────────────────────────────────
  play(index) {
    if (index != null) this.currentIndex = index;
    if (this.currentIndex < 0) this.currentIndex = 0;
    if (!this.playlist.length) return;

    const track = this.playlist[this.currentIndex];
    if (!track) return;

    if (this.audio.src !== track.src) {
      this.audio.src = track.src;
      this.audio.currentTime = 0;
    }

    this.audio.play()
      .then(() => {
        this.isPlaying = true;
        this.updateNowPlaying();
        this.updateMediaSession(track);
        this.saveState();
      })
      .catch(err => {
        console.warn('[MusicEngine] Play error:', err.message);
        // Autoplay blocked — update UI but mark as paused
        this.updateNowPlaying();
      });
  },

  pause() {
    this.audio.pause();
    this.isPlaying = false;
    this.updateNowPlaying();
    this.saveState();
  },

  togglePlay() {
    if (this.isPlaying) this.pause();
    else this.play();
  },

  next() {
    if (!this.playlist.length) return;
    if (this.repeat === 'one') { this.audio.currentTime = 0; this.play(); return; }

    if (this.shuffle) {
      const curPos = this.shuffleOrder.indexOf(this.currentIndex);
      const nextPos = (curPos + 1) % this.shuffleOrder.length;
      this.play(this.shuffleOrder[nextPos]);
    } else {
      const nextIndex = (this.currentIndex + 1) % this.playlist.length;
      if (nextIndex === 0 && this.repeat === 'none') {
        // End of playlist
        this.pause();
        this.currentIndex = 0;
        this.audio.currentTime = 0;
        this.updateNowPlaying();
        return;
      }
      this.play(nextIndex);
    }
  },

  prev() {
    if (!this.playlist.length) return;
    // If more than 3 seconds in, restart current track
    if (this.audio.currentTime > 3) {
      this.audio.currentTime = 0;
      return;
    }
    if (this.shuffle) {
      const curPos = this.shuffleOrder.indexOf(this.currentIndex);
      const prevPos = (curPos - 1 + this.shuffleOrder.length) % this.shuffleOrder.length;
      this.play(this.shuffleOrder[prevPos]);
    } else {
      const prevIndex = (this.currentIndex - 1 + this.playlist.length) % this.playlist.length;
      this.play(prevIndex);
    }
  },

  toggleShuffle() {
    this.shuffle = !this.shuffle;
    this.buildShuffleOrder();
    this.saveState();
    const btn = document.getElementById('muf-btn-shuffle');
    if (btn) btn.style.color = this.shuffle ? 'var(--g)' : '';
    this.showToast(this.shuffle ? '🔀 Shuffle on' : '🔀 Shuffle off');
  },

  toggleRepeat() {
    const modes = ['none', 'all', 'one'];
    const cur = modes.indexOf(this.repeat);
    this.repeat = modes[(cur + 1) % modes.length];
    this.saveState();
    const btn = document.getElementById('muf-btn-repeat');
    const labels = { none: '', all: '🔁', one: '🔂' };
    if (btn) {
      btn.style.color = this.repeat !== 'none' ? 'var(--g)' : '';
      btn.title = this.repeat === 'none' ? 'Repeat off' : this.repeat === 'all' ? 'Repeat all' : 'Repeat one';
    }
    const icons = { none: 'Repeat off', all: '🔁 Repeat all', one: '🔂 Repeat one' };
    this.showToast(icons[this.repeat]);
  },

  seek(fraction) {
    if (!isNaN(this.audio.duration)) {
      this.audio.currentTime = fraction * this.audio.duration;
    }
  },

  setVolume(v) {
    this.audio.volume = Math.max(0, Math.min(1, v));
    this.saveState();
  },


  // ── Audio event listeners ─────────────────────────────────
  setupAudioListeners() {
    const a = this.audio;

    a.addEventListener('ended', () => {
      this.isPlaying = false;
      this.next();
    });

    a.addEventListener('timeupdate', () => {
      this.updateProgress();
      // Save position every 5 seconds
      if (Math.floor(a.currentTime) % 5 === 0) {
        this.saveState();
      }
    });

    a.addEventListener('play',  () => { this.isPlaying = true;  this.updateNowPlaying(); });
    a.addEventListener('pause', () => { this.isPlaying = false; this.updateNowPlaying(); });

    a.addEventListener('error', e => {
      console.warn('[MusicEngine] Audio error:', e);
      this.showToast('⚠️ Could not play track');
    });

    a.addEventListener('waiting', () => {
      const disc = document.getElementById('muf-disc');
      if (disc) disc.style.animationPlayState = 'paused';
    });

    a.addEventListener('playing', () => {
      const disc = document.getElementById('muf-disc');
      if (disc) disc.style.animationPlayState = 'running';
    });
  },


  // ── Media Session API ─────────────────────────────────────
  // This enables lock screen controls on Android/iOS
  setupMediaSession() {
    if (!('mediaSession' in navigator)) {
      console.log('[MusicEngine] Media Session API not supported');
      return;
    }

    navigator.mediaSession.setActionHandler('play',     () => this.play());
    navigator.mediaSession.setActionHandler('pause',    () => this.pause());
    navigator.mediaSession.setActionHandler('nexttrack',     () => this.next());
    navigator.mediaSession.setActionHandler('previoustrack', () => this.prev());

    // Seekto — enables scrubbing from lock screen / notification
    navigator.mediaSession.setActionHandler('seekto', details => {
      if (details.fastSeek && 'fastSeek' in this.audio) {
        this.audio.fastSeek(details.seekTime);
      } else {
        this.audio.currentTime = details.seekTime;
      }
    });

    navigator.mediaSession.setActionHandler('seekforward', details => {
      const skip = details.seekOffset || 10;
      this.audio.currentTime = Math.min(this.audio.currentTime + skip, this.audio.duration || 0);
    });

    navigator.mediaSession.setActionHandler('seekbackward', details => {
      const skip = details.seekOffset || 10;
      this.audio.currentTime = Math.max(this.audio.currentTime - skip, 0);
    });

    console.log('[MusicEngine] Media Session API ready — lock screen controls enabled');
  },

  updateMediaSession(track) {
    if (!('mediaSession' in navigator)) return;

    navigator.mediaSession.playbackState = this.isPlaying ? 'playing' : 'paused';

    navigator.mediaSession.metadata = new MediaMetadata({
      title:  track.title  || 'Unknown Track',
      artist: track.artist || 'MUFASA STRENGTH',
      album:  'MUFASA Workout Playlist',
      artwork: track.img ? [
        { src: track.img, sizes: '512x512', type: 'image/jpeg' },
      ] : [
        { src: '/mufasa-header-logo.png', sizes: '192x192', type: 'image/png' },
      ],
    });

    // Update position state for scrubbar on lock screen
    if (!isNaN(this.audio.duration) && this.audio.duration > 0) {
      try {
        navigator.mediaSession.setPositionState({
          duration:     this.audio.duration,
          playbackRate: this.audio.playbackRate,
          position:     this.audio.currentTime,
        });
      } catch(e) {
        // setPositionState not supported everywhere
      }
    }
  },


  // ── Offline saving ────────────────────────────────────────
  async loadOfflineStatus() {
    try {
      const saved = await idbGetAll('offline');
      this.offlineSaved = new Set(saved.map(s => s.src));
    } catch(e) {}
  },

  async saveForOffline(track) {
    if (!track || !track.src) return;
    this.showToast('⏳ Saving "' + track.title + '" for offline…');

    // Tell the service worker to cache this track
    if (navigator.serviceWorker?.controller) {
      navigator.serviceWorker.controller.postMessage({
        type: 'SAVE_TRACK',
        url: track.src,
      });

      // Listen for confirmation from SW
      const handler = event => {
        if (event.data?.type === 'TRACK_SAVED' && event.data.url === track.src) {
          navigator.serviceWorker.removeEventListener('message', handler);
          if (event.data.success) {
            this.offlineSaved.add(track.src);
            idbSet('offline', track.src, { src: track.src, title: track.title, savedAt: Date.now() });
            this.showToast('✅ "' + track.title + '" saved offline!');
            this.renderOfflineBadges();
          } else {
            this.showToast('❌ Save failed — try again on WiFi');
          }
        }
      };
      navigator.serviceWorker.addEventListener('message', handler);

      // Timeout fallback
      setTimeout(() => navigator.serviceWorker.removeEventListener('message', handler), 30000);
    } else {
      this.showToast('⚠️ Service worker not active');
    }
  },

  async removeOffline(track) {
    if (!track || !track.src) return;
    if (navigator.serviceWorker?.controller) {
      navigator.serviceWorker.controller.postMessage({
        type: 'DELETE_TRACK',
        url: track.src,
      });
    }
    this.offlineSaved.delete(track.src);
    await idbDelete('offline', track.src);
    this.showToast('🗑️ Removed offline copy');
    this.renderOfflineBadges();
  },

  isOffline(track) {
    return track && this.offlineSaved.has(track.src);
  },


  // ── Local tracks (user's own files) ───────────────────────
  async loadLocalTracks() {
    try {
      this.localTracks = await idbGetAll('localTracks');
    } catch(e) {
      this.localTracks = [];
    }
  },

  async addLocalFile(file) {
    if (!file || !file.type.startsWith('audio/')) {
      this.showToast('⚠️ Only audio files are supported');
      return;
    }

    // Store file as ArrayBuffer in IndexedDB (persists across sessions)
    const arrayBuffer = await file.arrayBuffer();
    const db    = await openIDB();
    const tx    = db.transaction('localTracks', 'readwrite');
    const store = tx.objectStore('localTracks');

    const record = {
      name:      file.name.replace(/\.[^.]+$/, ''),  // strip extension
      fileName:  file.name,
      type:      file.type,
      size:      file.size,
      data:      arrayBuffer,
      addedAt:   Date.now(),
      isLocal:   true,
    };

    await new Promise((res, rej) => {
      const req = store.add(record);
      req.onsuccess = () => { record.id = req.result; res(); };
      req.onerror   = () => rej(req.error);
    });

    this.localTracks.push(record);
    this.showToast('✅ "' + record.name + '" added to library');
    this.renderLocalTracks();
  },

  createBlobURL(localTrack) {
    const blob = new Blob([localTrack.data], { type: localTrack.type });
    return URL.createObjectURL(blob);
  },

  async playLocal(localTrackId) {
    const track = this.localTracks.find(t => t.id === localTrackId);
    if (!track) return;

    // Add to front of playlist if not already there
    const existing = this.playlist.find(t => t.localId === localTrackId);
    if (!existing) {
      const blobUrl = this.createBlobURL(track);
      const playlistTrack = {
        title:   track.name,
        artist:  'My Music',
        src:     blobUrl,
        dur:     '?:??',
        img:     null,
        cat:     'local',
        isLocal: true,
        localId: localTrackId,
      };
      this.playlist.unshift(playlistTrack);
      this.buildShuffleOrder();
      this.render();
    }
    this.play(0);
  },

  async deleteLocalTrack(localTrackId) {
    await idbDelete('localTracks', localTrackId);
    this.localTracks = this.localTracks.filter(t => t.id !== localTrackId);
    // Remove from playlist
    this.playlist = this.playlist.filter(t => t.localId !== localTrackId);
    this.buildShuffleOrder();
    if (this.currentIndex >= this.playlist.length) this.currentIndex = 0;
    this.renderLocalTracks();
    this.showToast('🗑️ Track removed');
  },


  // ── UI update helpers ─────────────────────────────────────
  updateNowPlaying() {
    const track = this.playlist[this.currentIndex];
    if (!track) return;

    const titleEl  = document.getElementById('muf-track-title');
    const artistEl = document.getElementById('muf-track-artist');
    const playBtn  = document.getElementById('muf-btn-play');
    const discEl   = document.getElementById('muf-disc');

    if (titleEl)  titleEl.textContent  = track.title  || 'Unknown';
    if (artistEl) artistEl.textContent = track.artist || 'Artist';

    if (playBtn) {
      playBtn.textContent      = this.isPlaying ? '⏸' : '▶';
      playBtn.setAttribute('aria-label', this.isPlaying ? 'Pause' : 'Play');
    }

    if (discEl) {
      if (this.isPlaying) discEl.classList.add('sp');
      else                discEl.classList.remove('sp');
    }

    // Highlight active item in list
    document.querySelectorAll('.mitem').forEach((el, i) => {
      el.classList.toggle('act', i === this.currentIndex);
    });

    // Offline save button state
    this.renderOfflineBadges();

    // Media session update
    if (this.isPlaying) this.updateMediaSession(track);
  },

  updateProgress() {
    const a = this.audio;
    if (!a.duration || isNaN(a.duration)) return;

    const pct = (a.currentTime / a.duration) * 100;

    // Progress bar
    const bar = document.getElementById('muf-progress-fill');
    if (bar) bar.style.width = pct + '%';

    // Worm segments
    const segs = document.querySelectorAll('.wseg');
    if (segs.length) {
      const total    = segs.length;
      const active   = Math.floor((pct / 100) * total);
      segs.forEach((seg, i) => {
        seg.classList.toggle('done', i < active);
        seg.classList.toggle('act',  i === active);
      });
    }

    // Time display
    const timeEl = document.getElementById('muf-time');
    if (timeEl) {
      timeEl.textContent = formatTime(a.currentTime) + ' / ' + formatTime(a.duration);
    }

    // Media Session position update (throttled — every 5s)
    if (Math.floor(a.currentTime) % 5 === 0 && 'mediaSession' in navigator) {
      try {
        navigator.mediaSession.setPositionState({
          duration:     a.duration,
          playbackRate: a.playbackRate,
          position:     a.currentTime,
        });
      } catch(e) {}
    }
  },

  renderOfflineBadges() {
    document.querySelectorAll('[data-offline-btn]').forEach(btn => {
      const src  = btn.dataset.offlineBtn;
      const saved = this.offlineSaved.has(src);
      btn.textContent  = saved ? '📶 Saved' : '💾 Save offline';
      btn.title        = saved ? 'Remove offline copy' : 'Save for offline';
      btn.style.color  = saved ? 'var(--g)' : '';
    });
  },

  renderLocalTracks() {
    const container = document.getElementById('muf-local-tracks');
    if (!container) return;

    if (!this.localTracks.length) {
      container.innerHTML = `<div style="color:var(--dim);font-size:.78rem;padding:12px;text-align:center">
        No local tracks yet.<br>Tap + to add music from your device.
      </div>`;
      return;
    }

    container.innerHTML = this.localTracks.map(t => `
      <div class="mitem" style="justify-content:space-between">
        <div style="display:flex;align-items:center;gap:10px;flex:1;min-width:0" onclick="MusicEngine.playLocal(${t.id})">
          <div style="font-size:1.1rem;flex-shrink:0">🎵</div>
          <div class="mitem-i">
            <div class="mitem-t">${escHtml(t.name)}</div>
            <div class="mitem-a">📁 My Music · ${formatBytes(t.size)}</div>
          </div>
        </div>
        <button onclick="MusicEngine.deleteLocalTrack(${t.id})" title="Remove"
          style="background:none;border:none;color:var(--dim);cursor:pointer;font-size:.82rem;padding:4px 6px;flex-shrink:0">🗑️</button>
      </div>
    `).join('');
  },

  // Full list render (called on init and playlist load)
  render() {
    this.renderTrackList();
    this.renderLocalTracks();
    this.updateNowPlaying();

    // Set initial control states
    const shuffleBtn = document.getElementById('muf-btn-shuffle');
    const repeatBtn  = document.getElementById('muf-btn-repeat');
    if (shuffleBtn) shuffleBtn.style.color = this.shuffle ? 'var(--g)' : '';
    if (repeatBtn) {
      repeatBtn.style.color = this.repeat !== 'none' ? 'var(--g)' : '';
      const icons = { none: '🔁', all: '🔁', one: '🔂' };
      repeatBtn.textContent = icons[this.repeat] || '🔁';
    }
  },

  renderTrackList() {
    const container = document.getElementById('muf-track-list');
    if (!container) return;

    if (!this.playlist.length) {
      container.innerHTML = `<div style="color:var(--dim);font-size:.82rem;padding:18px;text-align:center">
        No tracks yet. Ask your coach to add music from the Admin panel.
      </div>`;
      return;
    }

    container.innerHTML = this.playlist.map((track, i) => `
      <div class="mitem ${i === this.currentIndex ? 'act' : ''}" 
           onclick="MusicEngine.play(${i})" 
           data-index="${i}">
        <div class="mnum">${i === this.currentIndex && this.isPlaying ? '▶' : i + 1}</div>
        <div class="mitem-i">
          <div class="mitem-t">${escHtml(track.title || 'Unknown')}</div>
          <div class="mitem-a">${escHtml(track.artist || '')}${track.isLocal ? ' · 📁 Local' : ''}</div>
        </div>
        <div style="display:flex;align-items:center;gap:6px;flex-shrink:0">
          ${!track.isLocal ? `
            <button data-offline-btn="${escHtml(track.src)}"
              onclick="event.stopPropagation(); MusicEngine.offlineSaved.has('${escHtml(track.src)}')
                ? MusicEngine.removeOffline(MusicEngine.playlist[${i}])
                : MusicEngine.saveForOffline(MusicEngine.playlist[${i}])"
              style="background:none;border:none;cursor:pointer;font-size:.7rem;color:var(--dim);
                     padding:3px 5px;font-family:'Barlow Condensed',sans-serif;font-weight:700;
                     letter-spacing:.04em;transition:color .2s"
              title="Save for offline">
              ${this.offlineSaved.has(track.src) ? '📶 Saved' : '💾 Save'}
            </button>
          ` : ''}
          <span class="mitem-d">${track.dur || ''}</span>
        </div>
      </div>
    `).join('');
  },


  // ── Seekbar click ──────────────────────────────────────────
  handleSeekClick(e) {
    const bar   = e.currentTarget;
    const rect  = bar.getBoundingClientRect();
    const frac  = Math.max(0, Math.min(1, (e.clientX - rect.left) / rect.width));
    this.seek(frac);
  },


  // ── Toast ─────────────────────────────────────────────────
  showToast(msg, duration = 2600) {
    let toast = document.getElementById('muf-toast');
    if (!toast) {
      // Fallback — try the main app toast
      toast = document.getElementById('toast');
    }
    if (!toast) { console.log('[MusicEngine]', msg); return; }
    toast.textContent = msg;
    toast.classList.add('show');
    setTimeout(() => toast.classList.remove('show'), duration);
  },
};


// ── Utility helpers ───────────────────────────────────────────
function formatTime(secs) {
  if (!secs || isNaN(secs)) return '0:00';
  const m = Math.floor(secs / 60);
  const s = Math.floor(secs % 60).toString().padStart(2, '0');
  return m + ':' + s;
}

function formatBytes(bytes) {
  if (!bytes) return '';
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(0) + ' KB';
  return (bytes / (1024 * 1024)).toFixed(1) + ' MB';
}

function escHtml(str) {
  if (!str) return '';
  return String(str)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;');
}


// ── Wire up seekbar ───────────────────────────────────────────
document.addEventListener('DOMContentLoaded', () => {
  const seekbar = document.getElementById('muf-progress');
  if (seekbar) {
    seekbar.addEventListener('click', e => MusicEngine.handleSeekClick(e));
    seekbar.style.cursor = 'pointer';
  }

  // Local file upload trigger
  const uploadInput = document.getElementById('muf-local-upload');
  if (uploadInput) {
    uploadInput.addEventListener('change', async e => {
      const files = Array.from(e.target.files || []);
      for (const file of files) {
        await MusicEngine.addLocalFile(file);
      }
      uploadInput.value = '';
    });
  }
});


// ── Export globally ───────────────────────────────────────────
window.MusicEngine = MusicEngine;
