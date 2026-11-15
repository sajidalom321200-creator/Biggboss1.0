# Biggboss1.0
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
  <title>Bigg Boss 3D — Real Multiplayer Social Survival</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
  <script src="https://unpkg.com/mqtt/dist/mqtt.min.js"></script>
  <script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-database-compat.js"></script>
  <style>
    :root {
      --bg: #12091c;
      --gold: #f5c451;
      --crimson: #e11d48;
      --glass: rgba(15, 8, 25, 0.86);
      --border: rgba(245, 196, 81, 0.34);
    }
    * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; user-select: none; }
    input, textarea, select { user-select: text; -webkit-user-select: text; }
    html, body {
      margin: 0; padding: 0; width: 100%; height: 100%;
      overflow: hidden; background: var(--bg); color: #f8f4ee;
      font-family: 'Inter', system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    }
    #canvas-container { position: fixed; inset: 0; z-index: 1; touch-action: none; }
    #speech-layer { position: fixed; inset: 0; z-index: 10; pointer-events: none; overflow: hidden; }
    .speech-bubble {
      position: absolute; transform: translate(-50%, -100%);
      background: rgba(15, 8, 25, 0.94); color: #fff;
      border: 1.5px solid var(--gold); border-radius: 14px;
      padding: 5px 10px; font-size: 11px; line-height: 1.3;
      max-width: 200px; text-align: center;
      box-shadow: 0 8px 24px rgba(0,0,0,0.7);
      animation: popIn 0.2s ease-out;
    }
    .speech-bubble.angry {
      border-color: #ff2e55; background: rgba(70, 8, 22, 0.95);
      box-shadow: 0 0 18px rgba(255, 46, 85, 0.55);
    }
    .speech-bubble::after {
      content: ''; position: absolute; left: 50%; bottom: -7px; transform: translateX(-50%);
      border-width: 7px 6px 0 6px; border-style: solid;
      border-color: var(--gold) transparent transparent transparent;
    }
    .speech-bubble.angry::after { border-color: #ff2e55 transparent transparent transparent; }
    .name-tag {
      position: absolute; transform: translate(-50%, -100%);
      font-size: 10px; font-weight: 700; padding: 2px 8px; border-radius: 999px;
      background: rgba(12, 6, 20, 0.82); border: 1px solid rgba(255,255,255,0.22);
      white-space: nowrap; text-shadow: 0 1px 2px #000;
    }
    @keyframes popIn { from { opacity: 0; transform: translate(-50%, -85%) scale(0.85); } to { opacity: 1; transform: translate(-50%, -100%) scale(1); } }
    @keyframes blinkRec { 0%, 49% { opacity: 1; } 50%, 100% { opacity: 0.2; } }
    .rec-dot { animation: blinkRec 1.1s infinite; }

    .glass-panel {
      background: var(--glass);
      backdrop-filter: blur(14px); -webkit-backdrop-filter: blur(14px);
      border: 1px solid var(--border);
    }
    .cctv-scanlines {
      position: fixed; inset: 0; z-index: 15; pointer-events: none;
      background: repeating-linear-gradient(0deg, rgba(0,0,0,0.13) 0px, rgba(0,0,0,0.13) 1px, transparent 1px, transparent 3px);
    }
    .night-vision { filter: sepia(100%) hue-rotate(85deg) saturate(210%) contrast(1.15); }
    .custom-scroll::-webkit-scrollbar { width: 4px; height: 4px; }
    .custom-scroll::-webkit-scrollbar-thumb { background: rgba(245, 196, 81, 0.38); border-radius: 4px; }
    .no-scrollbar::-webkit-scrollbar { display: none; }
    .joystick-zone {
      width: 94px; height: 94px; border-radius: 999px;
      background: radial-gradient(circle, rgba(245,196,81,0.18) 0%, rgba(19,10,29,0.75) 72%);
      border: 1.5px solid rgba(245,196,81,0.45);
      touch-action: none; position: relative;
    }
    .joystick-knob {
      width: 38px; height: 38px; border-radius: 999px;
      background: linear-gradient(135deg, #f5c451, #e11d48);
      box-shadow: 0 4px 16px rgba(225,29,72,0.55);
      position: absolute; left: 50%; top: 50%;
      transform: translate(-50%, -50%); pointer-events: none;
    }
    #fps-crosshair {
      position: fixed; left: 50%; top: 50%; transform: translate(-50%, -50%);
      width: 6px; height: 6px; border-radius: 999px;
      background: rgba(245, 196, 81, 0.72); border: 1px solid rgba(0,0,0,0.65);
      z-index: 9; pointer-events: none;
    }
  </style>
</head>
<body>
  <!-- 3D WebGL Canvas -->
  <div id="canvas-container"></div>
  <div id="fps-crosshair" class="hidden"></div>
  <div id="speech-layer"></div>

  <!-- Bigg Boss House-Wide Announcement Banner -->
  <div id="bb-announcement-banner" class="hidden fixed top-12 left-1/2 -translate-x-1/2 z-50 w-[86%] max-w-xl pointer-events-none">
    <div class="glass-panel rounded-2xl px-4 py-2.5 border-2 border-amber-400 shadow-[0_0_35px_rgba(245,196,81,0.45)] text-center">
      <div class="text-[10px] uppercase tracking-[0.25em] text-amber-400 font-extrabold mb-0.5">👁️ BIGG BOSS ANNOUNCEMENT</div>
      <div id="bb-announcement-text" class="text-xs sm:text-sm font-bold text-white leading-snug"></div>
    </div>
  </div>

  <!-- Now Playing Morning Song / Kukdoo-Koo Pill -->
  <div id="music-now-playing" class="hidden fixed top-[92px] left-1/2 -translate-x-1/2 z-50 pointer-events-auto glass-panel rounded-full px-3.5 py-1 border border-amber-400/70 flex items-center gap-2 shadow-xl">
    <span id="music-badge-icon" class="text-sm">🎵</span>
    <span id="music-track-title" class="text-[11px] font-extrabold text-amber-200 max-w-[230px] truncate">Morning Anthem</span>
    <button onclick="stopHouseMusic()" class="px-2 py-0.5 rounded-full bg-rose-600/85 text-white text-[9px] font-black">⏹ Stop</button>
  </div>

  <!-- Wardrobe Modal -->
  <div id="wardrobe-modal" class="hidden fixed inset-0 z-50 bg-black/70 backdrop-blur-sm flex items-center justify-center p-3">
    <div class="glass-panel max-w-md w-full rounded-3xl p-4 border-2 border-amber-400/60 space-y-3">
      <div class="flex items-center justify-between">
        <div>
          <h3 class="text-sm font-black text-amber-400">👗 BIGG BOSS DESIGNER WARDROBE</h3>
          <p class="text-[10px] text-gray-300">Choose your new outfit & style — updates live in 3D!</p>
        </div>
        <button onclick="closeWardrobeModal()" class="px-2.5 py-1 rounded-xl bg-white/10 text-xs font-bold text-rose-300">✕ Close</button>
      </div>
      <div id="wardrobe-outfit-grid" class="grid grid-cols-2 sm:grid-cols-4 gap-2"></div>
      <div class="pt-1 border-t border-white/10 flex items-center justify-between gap-2">
        <span class="text-[10px] font-bold text-amber-300">💇 Hair Style:</span>
        <select id="wardrobe-hair-select" onchange="changeWardrobeHair(this.value)" class="flex-1 px-2.5 py-1 rounded-xl bg-black/65 border border-white/15 text-xs text-white">
          <option value="classic">Tailored Fade</option>
          <option value="pompadour">Royal Pompadour</option>
          <option value="long">Long Flowing Waves</option>
          <option value="ponytail">Sleek High Ponytail</option>
          <option value="curly">Textured Curls</option>
          <option value="crown">Street Snapback</option>
        </select>
      </div>
    </div>
  </div>

  <!-- Conference Room Nomination Modal -->
  <div id="conference-modal" class="hidden fixed inset-0 z-50 bg-black/75 backdrop-blur-sm flex items-center justify-center p-3">
    <div class="glass-panel max-w-lg w-full rounded-3xl p-4 border-2 border-cyan-400/60 space-y-2.5 max-h-[88vh] flex flex-col">
      <div class="flex items-center justify-between shrink-0">
        <div>
          <h3 class="text-sm font-black text-cyan-300">⚖️ GLASS CONFERENCE — NOMINATION TABLE</h3>
          <p id="conf-nom-subtitle" class="text-[10px] text-gray-300">Baith kar apna Nomination vote cast karein</p>
        </div>
        <button onclick="closeConferenceModal()" class="px-2.5 py-1 rounded-xl bg-white/10 text-xs font-bold text-rose-300">✕ Close</button>
      </div>
      <div id="conf-seated-strip" class="shrink-0 flex flex-wrap gap-1.5 text-[10px] bg-black/45 rounded-xl p-2 border border-white/10"></div>
      <div class="shrink-0 flex items-center gap-2">
        <span class="text-[10px] font-bold text-amber-300">Reason:</span>
        <select id="conf-nom-reason" class="flex-1 px-2.5 py-1 rounded-xl bg-black/65 border border-white/15 text-xs text-white">
          <option value="Fake gameplay & double face">Fake gameplay &amp; double face</option>
          <option value="Zero task & kitchen contribution">Zero task &amp; kitchen contribution</option>
          <option value="Breaks Bigg Boss house rules">Breaks Bigg Boss house rules</option>
          <option value="Secret groupism & plotting">Secret groupism &amp; plotting</option>
        </select>
      </div>
      <div id="conf-nom-list" class="flex-1 overflow-y-auto custom-scroll space-y-1.5 pr-0.5"></div>
    </div>
  </div>

  <!-- Event / Eviction / Finale Modal -->
  <div id="event-modal" class="hidden fixed inset-0 z-50 bg-black/80 backdrop-blur-md flex items-center justify-center p-4">
    <div class="glass-panel max-w-md w-full rounded-3xl p-5 text-center border-2 border-amber-400">
      <div id="event-emoji" class="text-4xl mb-1.5">🏆</div>
      <h2 id="event-title" class="text-xl font-black text-amber-400 tracking-wide mb-1.5"></h2>
      <p id="event-desc" class="text-xs text-gray-200 mb-4"></p>
      <button onclick="document.getElementById('event-modal').classList.add('hidden')" class="px-6 py-2 rounded-xl bg-gradient-to-r from-amber-400 to-rose-600 text-black font-extrabold text-xs">Continue</button>
    </div>
  </div>

  <!-- AUTH SCREEN / LOBBY PORTAL -->
  <div id="auth-screen" class="fixed inset-0 z-40 flex items-center justify-center p-2.5 bg-black/55 backdrop-blur-sm overflow-y-auto">
    <div class="glass-panel w-full max-w-3xl rounded-3xl p-4 sm:p-5 shadow-2xl border border-amber-400/35 max-h-[95vh] overflow-y-auto custom-scroll">
      <div class="flex items-center justify-between mb-3 gap-2">
        <div class="flex items-center gap-2.5">
          <div id="bb-eye-logo" onclick="handleSecretEyeTap()" class="w-9 h-9 rounded-xl bg-gradient-to-br from-amber-400 to-rose-600 flex items-center justify-center text-xl shadow-lg cursor-pointer">👁️</div>
          <div>
            <h1 class="text-lg sm:text-xl font-black tracking-tight text-amber-400">BIGG BOSS 3D LIVE</h1>
            <p class="text-[10px] text-purple-200/85">100% Real Players Only • Live Date, Sleep & Morning Songs</p>
          </div>
        </div>
        <div class="flex items-center gap-2">
          <span id="lobby-online-pill" class="px-2.5 py-1 rounded-full text-[10px] font-bold bg-emerald-500/20 text-emerald-300 border border-emerald-400/30">🌐 Connecting Live...</span>
        </div>
      </div>

      <form id="auth-form" onsubmit="handleAuthSubmit(event)" class="grid grid-cols-1 md:grid-cols-12 gap-3.5 items-start">
        <!-- Credentials Column -->
        <div class="md:col-span-5 space-y-2">
          <div class="grid grid-cols-3 gap-1 p-1 rounded-xl bg-black/45">
            <button type="button" id="tab-register" onclick="switchAuthTab('register')" class="py-1.5 rounded-lg text-[11px] font-extrabold bg-gradient-to-r from-amber-400 to-rose-600 text-black">Register</button>
            <button type="button" id="tab-login" onclick="switchAuthTab('login')" class="py-1.5 rounded-lg text-[11px] font-bold text-gray-300 hover:text-white">Login</button>
            <button type="button" id="tab-admin" onclick="switchAuthTab('admin')" class="py-1.5 rounded-lg text-[11px] font-bold text-amber-300 hover:text-white">👁️ Admin</button>
          </div>

          <div class="grid grid-cols-2 gap-2">
            <div>
              <label id="auth-user-label" class="block text-[10px] font-bold text-amber-300/90 mb-0.5">Your Name</label>
              <input id="auth-username" type="text" required autocomplete="off" placeholder="Player name..."
                class="w-full px-3 py-1.5 rounded-xl bg-black/55 border border-white/15 text-xs text-white focus:outline-none focus:border-amber-400" />
            </div>
            <div>
              <label id="auth-pass-label" class="block text-[10px] font-bold text-amber-300/90 mb-0.5">Passcode</label>
              <div class="relative flex items-center">
                <input id="auth-password" type="password" required autocomplete="off" placeholder="Passcode..."
                  class="w-full pl-3 pr-7 py-1.5 rounded-xl bg-black/55 border border-white/15 text-xs text-white focus:outline-none focus:border-amber-400" />
                <button type="button" onclick="togglePasswordVis()" title="Show/Hide Passcode" class="absolute right-1.5 text-[11px] text-gray-300 px-1">👁️</button>
              </div>
            </div>
          </div>

          <div>
            <label class="block text-[10px] font-bold text-emerald-300 mb-0.5">🌐 Live Multiplayer Room Code</label>
            <input id="auth-room-code" type="text" value="BB-SEASON-1" maxlength="20"
              class="w-full px-3 py-1.5 rounded-xl bg-black/55 border border-emerald-400/30 text-xs font-bold text-emerald-200 uppercase focus:outline-none focus:border-amber-400" />
          </div>

          <div id="auth-error" class="hidden text-[11px] text-rose-300 font-semibold bg-rose-950/70 border border-rose-500/50 rounded-xl px-3 py-1.5"></div>

          <button type="submit" id="auth-submit-btn" class="w-full py-2.5 rounded-xl bg-gradient-to-r from-amber-400 via-amber-500 to-rose-600 text-black font-black text-xs tracking-wide shadow-lg hover:brightness-110 active:scale-[0.99] transition">
            JOIN REAL MULTIPLAYER HOUSE
          </button>
        </div>

        <!-- Avatar Customizer Column -->
        <div id="avatar-customizer" class="md:col-span-7 rounded-2xl bg-black/35 border border-white/10 p-3 space-y-2.5">
          <div class="flex items-center justify-between">
            <span class="text-xs font-extrabold text-amber-300">Realistic 3D Avatar & Voice</span>
            <span id="voice-preview-tag" class="text-[10px] px-2 py-0.5 rounded-full bg-blue-500/20 text-blue-200 border border-blue-400/30">♂️ Male Voice Assigned</span>
          </div>

          <div>
            <span class="text-[10px] text-gray-300 block mb-1">Gender & Voice Type</span>
            <div class="grid grid-cols-2 gap-2">
              <button type="button" id="gender-male-btn" onclick="selectGender('male')" class="py-1.5 rounded-xl text-xs font-extrabold bg-blue-500/30 border border-blue-400 text-white">♂️ Boy (Male Voice)</button>
              <button type="button" id="gender-female-btn" onclick="selectGender('female')" class="py-1.5 rounded-xl text-xs font-bold bg-black/40 border border-white/15 text-gray-300">♀️ Girl (Female Voice)</button>
            </div>
          </div>

          <div class="grid grid-cols-2 gap-3">
            <div>
              <span class="text-[10px] text-gray-300 block mb-1">Skin Tone</span>
              <div id="skin-swatches" class="flex gap-1.5 flex-wrap"></div>
            </div>
            <div>
              <span class="text-[10px] text-gray-300 block mb-1">Outfit Color</span>
              <div id="outfit-swatches" class="flex gap-1.5 flex-wrap"></div>
            </div>
          </div>

          <div class="grid grid-cols-2 gap-3">
            <div>
              <span class="text-[10px] text-gray-300 block mb-1">Hair Style</span>
              <select id="hair-style" class="w-full px-2.5 py-1.5 rounded-lg bg-black/65 border border-white/15 text-xs text-white">
                <option value="classic">Tailored Fade</option>
                <option value="pompadour">Royal Pompadour</option>
                <option value="long">Long Flowing Waves</option>
                <option value="ponytail">Sleek High Ponytail</option>
                <option value="curly">Textured Curls</option>
                <option value="crown">Street Snapback</option>
              </select>
            </div>
            <div>
              <span class="text-[10px] text-gray-300 block mb-1">Hair Color</span>
              <div id="hair-swatches" class="flex gap-1.5 flex-wrap pt-0.5"></div>
            </div>
          </div>

          <div class="text-[10px] text-purple-200/80 bg-white/5 rounded-xl px-2.5 py-1.5 flex items-center justify-between">
            <span>👥 Real Players Currently in Room:</span>
            <strong id="lobby-real-count" class="text-amber-300">0 / 15 Real Players</strong>
          </div>
        </div>
      </form>
    </div>
  </div>

  <!-- MAIN HUD & CONTROLS INTERFACE -->
  <div id="game-hud" class="hidden fixed inset-0 z-20 pointer-events-none flex flex-col justify-between p-3 select-none">
    <!-- Top HUD Bar -->
    <div class="flex items-center justify-between pointer-events-auto gap-2">
      <div class="glass-panel rounded-2xl px-3 py-1.5 flex items-center gap-2.5 border border-amber-400/40">
        <div class="w-7 h-7 rounded-xl bg-amber-400/20 flex items-center justify-center text-amber-400 text-sm font-black">👁️</div>
        <div>
          <div class="text-[10px] text-amber-400 font-black tracking-wider uppercase" id="hud-room-title">SEASON 1 • LIVE</div>
          <div class="text-[11px] font-extrabold text-white" id="hud-player-name">Player</div>
        </div>
      </div>

      <!-- Time / Date / Day State & Voting Status -->
      <div class="glass-panel rounded-2xl px-3 py-1.5 flex items-center gap-3 border border-amber-400/40">
        <div class="text-center">
          <div class="text-[9px] text-amber-300 font-bold uppercase" id="hud-day-state">☀️ MORNING</div>
          <div class="text-[11px] font-black text-white" id="hud-time-str">08:00 AM</div>
        </div>
        <div class="h-6 w-[1px] bg-white/20"></div>
        <div class="text-center">
          <div class="text-[9px] text-gray-300 font-bold">RATION</div>
          <div class="text-[11px] font-black text-emerald-400" id="hud-ration-val">100%</div>
        </div>
      </div>

      <!-- Quick Action Buttons -->
      <div class="flex items-center gap-1.5">
        <button onclick="openWardrobeModal()" class="glass-panel rounded-xl px-2.5 py-1.5 text-[11px] font-bold text-amber-300 border border-amber-400/40 hover:bg-amber-400/20">👗 Wardrobe</button>
        <button onclick="toggleCctvModal()" class="glass-panel rounded-xl px-2.5 py-1.5 text-[11px] font-bold text-cyan-300 border border-cyan-400/40 hover:bg-cyan-400/20">📹 CCTV</button>
        <button id="admin-hud-btn" onclick="toggleAdminModal()" class="hidden glass-panel rounded-xl px-2.5 py-1.5 text-[11px] font-bold text-rose-300 border border-rose-400/40 hover:bg-rose-400/20">👁️ Admin</button>
      </div>
    </div>

    <!-- Middle Floating Notifications / Target Prompts -->
    <div class="self-center pointer-events-auto">
      <div id="interactive-prompt" class="hidden glass-panel rounded-2xl px-4 py-2 border-2 border-amber-400 text-center shadow-2xl animate-bounce">
        <div id="interactive-prompt-text" class="text-xs font-black text-amber-300">Press E to Enter Conference Room</div>
      </div>
    </div>

    <!-- Bottom HUD Controls -->
    <div class="flex items-end justify-between pointer-events-auto gap-2">
      <!-- Joystick & Movement Controls -->
      <div class="flex items-end gap-3">
        <div id="joystick-wrapper" class="joystick-zone">
          <div id="joystick-knob" class="joystick-knob"></div>
        </div>
        <div class="flex flex-col gap-1.5">
          <button onclick="triggerPlayerAction('dance')" class="glass-panel rounded-xl px-3 py-1.5 text-xs font-bold text-amber-300 border border-amber-400/30">🕺 Dance</button>
          <button onclick="triggerPlayerAction('sit')" class="glass-panel rounded-xl px-3 py-1.5 text-xs font-bold text-cyan-300 border border-cyan-400/30">🪑 Sit</button>
          <button onclick="triggerPlayerAction('fight')" class="glass-panel rounded-xl px-3 py-1.5 text-xs font-bold text-rose-300 border border-rose-400/30">🔥 Fight</button>
        </div>
      </div>

      <!-- Chat & Voice Input Strip -->
      <div class="flex-1 max-w-md glass-panel rounded-2xl p-2 border border-amber-400/40 flex flex-col gap-1.5">
        <div class="flex items-center gap-1.5">
          <input id="chat-input" type="text" placeholder="Type message or argument in house..." onkeydown="if(event.key==='Enter') sendPlayerChat()"
            class="flex-1 px-3 py-1.5 rounded-xl bg-black/60 border border-white/15 text-xs text-white focus:outline-none focus:border-amber-400" />
          <button onclick="sendPlayerChat()" class="px-3 py-1.5 rounded-xl bg-amber-400 text-black font-black text-xs">Send</button>
        </div>
        <div class="flex items-center justify-between text-[10px] text-gray-300 px-1">
          <span>🎤 Voice Speech Bubble:</span>
          <button onclick="toggleVoiceSpeech()" id="mic-toggle-btn" class="px-2.5 py-0.5 rounded-lg bg-emerald-600/80 text-white font-bold">🎙️ Speak (Mic)</button>
        </div>
      </div>

      <!-- Camera & Jump Controls -->
      <div class="flex flex-col gap-2">
        <button onclick="cycleCameraView()" class="glass-panel rounded-2xl px-3 py-2 text-xs font-extrabold text-amber-300 border border-amber-400/40">📹 Cam: <span id="cam-mode-label">3rd Person</span></button>
        <button onclick="playerJump()" class="glass-panel rounded-2xl px-4 py-3 text-xs font-black text-white bg-amber-500/20 border border-amber-400/40 active:scale-95">⬆️ Jump</button>
      </div>
    </div>
  </div>

  <!-- ADMIN CONTROL MODAL -->
  <div id="admin-modal" class="hidden fixed inset-0 z-50 bg-black/80 backdrop-blur-md flex items-center justify-center p-3">
    <div class="glass-panel max-w-xl w-full rounded-3xl p-4 border-2 border-rose-500/60 space-y-3 max-h-[90vh] flex flex-col">
      <div class="flex items-center justify-between shrink-0">
        <div>
          <h3 class="text-sm font-black text-rose-400">👁️ BIGG BOSS ADMIN MASTER VAULT</h3>
          <p class="text-[10px] text-gray-300">Absolute control over House, Music, Nominations &amp; Evictions</p>
        </div>
        <button onclick="toggleAdminModal()" class="px-2.5 py-1 rounded-xl bg-white/10 text-xs font-bold text-rose-300">✕ Close</button>
      </div>
      <div class="flex-1 overflow-y-auto custom-scroll space-y-3 pr-1 text-xs">
        <!-- Section 1: House Audio & Wakeup -->
        <div class="rounded-2xl bg-black/45 p-3 border border-white/10 space-y-2">
          <div class="font-extrabold text-amber-300">🎵 Morning Songs &amp; House Audio</div>
          <div class="grid grid-cols-2 sm:grid-cols-4 gap-1.5">
            <button onclick="adminPlayAnthem('kukdoo')" class="p-2 rounded-xl bg-amber-500/20 text-amber-200 font-bold border border-amber-400/30 text-left">🐓 Rooster Kukdoo-Koo</button>
            <button onclick="adminPlayAnthem('anthem')" class="p-2 rounded-xl bg-purple-500/20 text-purple-200 font-bold border border-purple-400/30 text-left">🎵 Morning Anthem</button>
            <button onclick="adminPlayAnthem('siren')" class="p-2 rounded-xl bg-rose-500/20 text-rose-200 font-bold border border-rose-400/30 text-left">🚨 Alarm Siren</button>
            <button onclick="stopHouseMusic()" class="p-2 rounded-xl bg-white/10 text-white font-bold border border-white/20 text-left">⏹ Stop Music</button>
          </div>
        </div>

        <!-- Section 2: House State & Lights -->
        <div class="rounded-2xl bg-black/45 p-3 border border-white/10 space-y-2">
          <div class="font-extrabold text-amber-300">☀️ House Day / Night &amp; Lights</div>
          <div class="grid grid-cols-3 gap-1.5">
            <button onclick="adminSetTimeState('morning')" class="p-2 rounded-xl bg-amber-500/20 text-amber-200 font-bold border border-amber-400/30 text-center">☀️ Morning Wakeup</button>
            <button onclick="adminSetTimeState('day')" class="p-2 rounded-xl bg-blue-500/20 text-blue-200 font-bold border border-blue-400/30 text-center">☀️ Day Time</button>
            <button onclick="adminSetTimeState('night')" class="p-2 rounded-xl bg-purple-900/40 text-purple-200 font-bold border border-purple-400/30 text-center">🌙 Lights Out</button>
          </div>
        </div>

        <!-- Section 3: House Announcements -->
        <div class="rounded-2xl bg-black/45 p-3 border border-white/10 space-y-2">
          <div class="font-extrabold text-amber-300">📢 Bigg Boss Voice / Announcement</div>
          <div class="flex gap-2">
            <input id="admin-announcement-input" type="text" placeholder="Type Bigg Boss voice announcement..." class="flex-1 px-3 py-1.5 rounded-xl bg-black/60 border border-white/15 text-xs text-white" />
            <button onclick="adminSendAnnouncement()" class="px-3 py-1.5 rounded-xl bg-amber-400 text-black font-black text-xs">Broadcast</button>
          </div>
        </div>

        <!-- Section 4: Live Players List & Eviction -->
        <div class="rounded-2xl bg-black/45 p-3 border border-white/10 space-y-2">
          <div class="font-extrabold text-amber-300">👥 Real Players in House (<span id="admin-player-count">0</span>)</div>
          <div id="admin-players-list" class="space-y-1.5 max-h-40 overflow-y-auto custom-scroll pr-1"></div>
        </div>
      </div>
    </div>
  </div>

  <!-- CCTV MODAL -->
  <div id="cctv-modal" class="hidden fixed inset-0 z-50 bg-black/85 backdrop-blur-md flex items-center justify-center p-3">
    <div class="glass-panel max-w-lg w-full rounded-3xl p-4 border-2 border-cyan-400/60 space-y-3">
      <div class="flex items-center justify-between">
        <div>
          <h3 class="text-sm font-black text-cyan-300">📹 BIGG BOSS CCTV CONTROL ROOM</h3>
          <p class="text-[10px] text-gray-300">Select camera angle to monitor housemates live</p>
        </div>
        <button onclick="toggleCctvModal()" class="px-2.5 py-1 rounded-xl bg-white/10 text-xs font-bold text-rose-300">✕ Close</button>
      </div>
      <div class="grid grid-cols-2 sm:grid-cols-3 gap-2">
        <button onclick="selectCctvCam('living')" class="p-2.5 rounded-xl bg-black/50 border border-cyan-400/40 text-cyan-200 font-bold text-left text-xs">🏠 Living Room</button>
        <button onclick="selectCctvCam('kitchen')" class="p-2.5 rounded-xl bg-black/50 border border-cyan-400/40 text-cyan-200 font-bold text-left text-xs">🍳 Kitchen & Dining</button>
        <button onclick="selectCctvCam('bedroom')" class="p-2.5 rounded-xl bg-black/50 border border-cyan-400/40 text-cyan-200 font-bold text-left text-xs">🛏️ Bedroom</button>
        <button onclick="selectCctvCam('garden')" class="p-2.5 rounded-xl bg-black/50 border border-cyan-400/40 text-cyan-200 font-bold text-left text-xs">🌳 Garden & Pool</button>
        <button onclick="selectCctvCam('conference')" class="p-2.5 rounded-xl bg-black/50 border border-cyan-400/40 text-cyan-200 font-bold text-left text-xs">⚖️ Glass Conference</button>
        <button onclick="selectCctvCam('bathroom')" class="p-2.5 rounded-xl bg-black/50 border border-cyan-400/40 text-cyan-200 font-bold text-left text-xs">🚿 Bathroom & Gym</button>
      </div>
    </div>
  </div>

  <div id="cctv-overlay-scan" class="hidden cctv-scanlines"></div>

  <!-- GAME LOGIC SCRIPT -->
  <script>
    // --- STATE & MULTIPLAYER SETUP ---
    let scene, camera, renderer;
    let myPlayerId = 'p_' + Math.random().toString(36).substring(2, 9);
    let myPlayerData = {
      name: 'Player',
      passcode: '1234',
      isAdmin: false,
      gender: 'male',
      skinColor: 0xffdbac,
      outfitColor: 0xe11d48,
      hairStyle: 'classic',
      hairColor: 0x222222,
      x: 0, z: 0, rot: 0,
      anim: 'idle',
      speech: '',
      speechTime: 0
    };

    let remotePlayers = {};
    let mqttClient = null;
    let firebaseDb = null;
    let syncMode = 'mqtt'; // 'mqtt' or 'firebase'

    // House timing & state
    let houseTimeMinutes = 480; // 08:00 AM in minutes
    let houseDayState = 'morning'; // 'morning', 'day', 'night'
    let rationPercentage = 100;
    let houseMusicAudio = null;

    // Movement & controls
    let moveForward = false, moveBackward = false, moveLeft = false, moveRight = false;
    let cameraMode = 0; // 0: 3rd person, 1: 1st person, 2: CCTV
    let currentCctvCam = 'living';
    let isConnected = false;

    // --- COLOR & ASSET PALETTES ---
    const skinTones = [0xffdbac, 0xf1c27d, 0xe0ac69, 0xc68642, 0x8d5524, 0x3d2314];
    const outfitColors = [0xe11d48, 0x2563eb, 0x16a34a, 0x9333ea, 0xd97706, 0x0891b2, 0xdb2777, 0x111827];
    const hairColors = [0x111827, 0x78350f, 0xd97706, 0x9ca3af, 0xffedd5];

    // --- INITIALIZATION ---
    window.addEventListener('DOMContentLoaded', () => {
      buildColorSwatches();
      initThreeScene();
      initNetworkSync();
      setupJoystick();
      setupKeyboardControls();

      // Tick loop
      setInterval(gameLoop, 1000 / 60);
      setInterval(houseClockTick, 2000); // Advance house time
    });

    function buildColorSwatches() {
      const skinBox = document.getElementById('skin-swatches');
      skinTones.forEach(c => {
        const div = document.createElement('div');
        div.className = `w-6 h-6 rounded-full cursor-pointer border-2 ${myPlayerData.skinColor === c ? 'border-amber-400 scale-110' : 'border-transparent'}`;
        div.style.backgroundColor = '#' + c.toString(16).padStart(6, '0');
        div.onclick = () => { myPlayerData.skinColor = c; buildColorSwatches(); };
        skinBox.appendChild(div);
      });

      const outfitBox = document.getElementById('outfit-swatches');
      outfitColors.forEach(c => {
        const div = document.createElement('div');
        div.className = `w-6 h-6 rounded-full cursor-pointer border-2 ${myPlayerData.outfitColor === c ? 'border-amber-400 scale-110' : 'border-transparent'}`;
        div.style.backgroundColor = '#' + c.toString(16).padStart(6, '0');
        div.onclick = () => { myPlayerData.outfitColor = c; buildColorSwatches(); };
        outfitBox.appendChild(div);
      });

      const hairBox = document.getElementById('hair-swatches');
      hairColors.forEach(c => {
        const div = document.createElement('div');
        div.className = `w-6 h-6 rounded-full cursor-pointer border-2 ${myPlayerData.hairColor === c ? 'border-amber-400 scale-110' : 'border-transparent'}`;
        div.style.backgroundColor = '#' + c.toString(16).padStart(6, '0');
        div.onclick = () => { myPlayerData.hairColor = c; buildColorSwatches(); };
        hairBox.appendChild(div);
      });
    }

    function selectGender(gender) {
      myPlayerData.gender = gender;
      document.getElementById('gender-male-btn').className = gender === 'male' ? 'py-1.5 rounded-xl text-xs font-extrabold bg-blue-500/30 border border-blue-400 text-white' : 'py-1.5 rounded-xl text-xs font-bold bg-black/40 border border-white/15 text-gray-300';
      document.getElementById('gender-female-btn').className = gender === 'female' ? 'py-1.5 rounded-xl text-xs font-extrabold bg-pink-500/30 border border-pink-400 text-white' : 'py-1.5 rounded-xl text-xs font-bold bg-black/40 border border-white/15 text-gray-300';
      document.getElementById('voice-preview-tag').innerText = gender === 'male' ? '♂️ Male Voice Assigned' : '♀️ Female Voice Assigned';
    }

    // --- AUTH TABS & ADMIN TOGGLE ---
    let authMode = 'register';
    function switchAuthTab(mode) {
      authMode = mode;
      document.getElementById('tab-register').className = mode === 'register' ? 'py-1.5 rounded-lg text-[11px] font-extrabold bg-gradient-to-r from-amber-400 to-rose-600 text-black' : 'py-1.5 rounded-lg text-[11px] font-bold text-gray-300 hover:text-white';
      document.getElementById('tab-login').className = mode === 'login' ? 'py-1.5 rounded-lg text-[11px] font-extrabold bg-gradient-to-r from-amber-400 to-rose-600 text-black' : 'py-1.5 rounded-lg text-[11px] font-bold text-gray-300 hover:text-white';
      document.getElementById('tab-admin').className = mode === 'admin' ? 'py-1.5 rounded-lg text-[11px] font-extrabold bg-amber-400 text-black' : 'py-1.5 rounded-lg text-[11px] font-bold text-amber-300 hover:text-white';

      const passLabel = document.getElementById('auth-pass-label');
      const submitBtn = document.getElementById('auth-submit-btn');
      if(mode === 'admin') {
        passLabel.innerText = 'Admin Secret Passcode';
        submitBtn.innerText = 'ENTER ADMIN VAULT';
      } else if(mode === 'login') {
        passLabel.innerText = 'Passcode';
        submitBtn.innerText = 'LOGIN & ENTER HOUSE';
      } else {
        passLabel.innerText = 'Create Passcode';
        submitBtn.innerText = 'JOIN REAL MULTIPLAYER HOUSE';
      }
    }

    let secretEyeTaps = 0;
    function handleSecretEyeTap() {
      secretEyeTaps++;
      if(secretEyeTaps >= 3) {
        switchAuthTab('admin');
        document.getElementById('auth-password').value = 'BOSS999';
        showAnnouncement("👁️ Bigg Boss Secret Admin Portal Unlocked!");
      }
    }

    function togglePasswordVis() {
      const p = document.getElementById('auth-password');
      p.type = p.type === 'password' ? 'text' : 'password';
    }

    function handleAuthSubmit(e) {
      e.preventDefault();
      const name = document.getElementById('auth-username').value.trim();
      const pass = document.getElementById('auth-password').value.trim();
      const room = document.getElementById('auth-room-code').value.trim().toUpperCase() || 'BB-SEASON-1';

      if(!name || !pass) {
        showAuthError('Please enter valid name and passcode.');
        return;
      }

      if(authMode === 'admin' || pass === 'BOSS999' || name.toLowerCase() === 'admin') {
        if(pass !== 'BOSS999' && authMode === 'admin') {
          showAuthError('Incorrect Admin Passcode! (Use BOSS999)');
          return;
        }
        myPlayerData.isAdmin = true;
      }

      myPlayerData.name = name;
      myPlayerData.passcode = pass;
      myRoomCode = room;

      // Hide auth screen, show HUD
      document.getElementById('auth-screen').classList.add('hidden');
      document.getElementById('game-hud').classList.remove('hidden');
      if(myPlayerData.isAdmin) {
        document.getElementById('admin-hud-btn').classList.remove('hidden');
      }

      document.getElementById('hud-player-name.innerText = myPlayerData.name + (myPlayerData.isAdmin ? ' (Admin 👁️)' : '');
      document.getElementById('hud-room-title.innerText = myRoomCode + ' • LIVE';

      showAnnouncement(`👁️ Welcome to Bigg Boss House, ${myPlayerData.name}!`);
      broadcastPresence();
    }

    function showAuthError(msg) {
      const err = document.getElementById('auth-error');
      err.innerText = msg;
      err.classList.remove('hidden');
    }

    // --- THREE.JS 3D HOUSE & ENVIRONMENT ---
    function initThreeScene() {
      const container = document.getElementById('canvas-container');
      scene = new THREE.Scene();
      scene.background = new THREE.Color(0x150b24);

      camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
      renderer = new THREE.WebGLRenderer({ antialias: true, powerPreference: 'high-performance' });
      renderer.setSize(window.innerWidth, window.innerHeight);
      renderer.shadowMap.enabled = true;
      container.appendChild(renderer.domElement);

      // Lights
      const ambientLight = new THREE.AmbientLight(0xffffff, 0.7);
      scene.add(ambientLight);

      const dirLight = new THREE.DirectionalLight(0xfff5e6, 0.9);
      dirLight.position.set(20, 40, 20);
      dirLight.castShadow = true;
      scene.add(dirLight);

      buildBiggBossHouse();

      window.addEventListener('resize', () => {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
      });
    }

    function buildBiggBossHouse() {
      // Floor
      const floorGeo = new THREE.PlaneGeometry(80, 80);
      const floorMat = new THREE.MeshStandardMaterial({ color: 0x1f162e, roughness: 0.8 });
      const floor = new THREE.Mesh(floorGeo, floorMat);
      floor.rotation.x = -Math.PI / 2;
      floor.receiveShadow = true;
      scene.add(floor);

      // Living Room Rug
      const rugGeo = new THREE.PlaneGeometry(16, 12);
      const rugMat = new THREE.MeshStandardMaterial({ color: 0x881337, roughness: 0.9 });
      const rug = new THREE.Mesh(rugGeo, rugMat);
      rug.rotation.x = -Math.PI / 2;
      rug.position.set(0, 0.01, 0);
      scene.add(rug);

      // Sofa Set in Living Room
      createSofa(0, 0.5, -5, 4, 1.5, 0.7, 0x451a03);
      createSofa(-6, 0.5, 0, 1.5, 4, 0.7, 0x451a03);
      createSofa(6, 0.5, 0, 1.5, 4, 0.7, 0x451a03);

      // Glass Conference Table in Center
      createGlassConferenceTable(0, 0.75, 0);

      // Kitchen Counter & Dining
      createKitchenCounter(-18, 1, -12);
      createDiningTable(14, 0.75, -12);

      // Bedrooms
      createBed(-18, 0.4, 14);
      createBed(-12, 0.4, 14);
      createBed(12, 0.4, 14);
      createBed(18, 0.4, 14);

      // Swimming Pool in Garden
      const poolGeo = new THREE.BoxGeometry(14, 0.2, 10);
      const poolMat = new THREE.MeshStandardMaterial({ color: 0x0284c7, roughness: 0.2, transparent: true, opacity: 0.85 });
      const pool = new THREE.Mesh(poolGeo, poolMat);
      pool.position.set(0, 0.1, -24);
      scene.add(pool);

      // House Walls & Eye Logo
      createHouseWalls();
    }

    function createSofa(x, y, z, w, d, h, color) {
      const group = new THREE.Group();
      const baseMat = new THREE.MeshStandardMaterial({ color: color, roughness: 0.7 });
      const seat = new THREE.Mesh(new THREE.BoxGeometry(w, h, d), baseMat);
      seat.position.y = h / 2;
      seat.castShadow = true;
      group.add(seat);

      const back = new THREE.Mesh(new THREE.BoxGeometry(w, h * 1.5, 0.3), baseMat);
      back.position.set(0, h + (h * 0.75), -d/2 + 0.15);
      back.castShadow = true;
      group.add(back);

      group.position.set(x, y - 0.5, z);
      scene.add(group);
    }

    function createGlassConferenceTable(x, y, z) {
      const group = new THREE.Group();
      const topMat = new THREE.MeshStandardMaterial({ color: 0x06b6d4, roughness: 0.1, transparent: true, opacity: 0.6 });
      const top = new THREE.Mesh(new THREE.CylinderGeometry(3.5, 3.5, 0.15, 32), topMat);
      top.position.y = y;
      top.castShadow = true;
      group.add(top);

      // Table legs
      for(let i=0; i<4; i++) {
        const angle = (i / 4) * Math.PI * 2;
        const leg = new THREE.Mesh(new THREE.CylinderGeometry(0.1, 0.1, y), new THREE.MeshStandardMaterial({ color: 0xf5c451 }));
        leg.position.set(Math.cos(angle)*2.5, y/2, Math.sin(angle)*2.5);
        group.add(leg);
      }
      group.position.set(x, 0, z);
      scene.add(group);
    }

    function createKitchenCounter(x, y, z) {
      const geo = new THREE.BoxGeometry(8, 2, 2.5);
      const mat = new THREE.MeshStandardMaterial({ color: 0x334155, roughness: 0.4 });
      const mesh = new THREE.Mesh(geo, mat);
      mesh.position.set(x, y, z);
      mesh.castShadow = true;
      scene.add(mesh);
    }

    function createDiningTable(x, y, z) {
      const top = new THREE.Mesh(new THREE.BoxGeometry(6, 0.2, 3), new THREE.MeshStandardMaterial({ color: 0x78350f }));
      top.position.set(x, y, z);
      top.castShadow = true;
      scene.add(top);
    }

    function createBed(x, y, z) {
      const bed = new THREE.Mesh(new THREE.BoxGeometry(3, 0.6, 4), new THREE.MeshStandardMaterial({ color: 0x1e293b }));
      bed.position.set(x, y, z);
      bed.castShadow = true;
      scene.add(bed);
    }

    function createHouseWalls() {
      const wallMat = new THREE.MeshStandardMaterial({ color: 0x2e1a47, roughness: 0.9 });
      // Outer perimeter walls
      const wall1 = new THREE.Mesh(new THREE.BoxGeometry(80, 8, 1), wallMat);
      wall1.position.set(0, 4, -40);
      scene.add(wall1);

      const wall2 = new THREE.Mesh(new THREE.BoxGeometry(80, 8, 1), wallMat);
      wall2.position.set(0, 4, 40);
      scene.add(wall2);

      const wall3 = new THREE.Mesh(new THREE.BoxGeometry(1, 8, 80), wallMat);
      wall3.position.set(-40, 4, 0);
      scene.add(wall3);

      const wall4 = new THREE.Mesh(new THREE.BoxGeometry(1, 8, 80), wallMat);
      wall4.position.set(40, 4, 0);
      scene.add(wall4);
    }

    // --- AVATAR 3D BUILDER ---
    function createAvatarMesh(data) {
      const group = new THREE.Group();
      const mat = new THREE.MeshStandardMaterial({ color: data.outfitColor, roughness: 0.6 });
      const skinMat = new THREE.MeshStandardMaterial({ color: data.skinColor, roughness: 0.5 });
      const hairMat = new THREE.MeshStandardMaterial({ color: data.hairColor, roughness: 0.8 });

      // Torso
      const torso = new THREE.Mesh(new THREE.BoxGeometry(0.8, 1.1, 0.4), mat);
      torso.position.y = 1.1;
      torso.castShadow = true;
      group.add(torso);

      // Head
      const head = new THREE.Mesh(new THREE.BoxGeometry(0.6, 0.6, 0.6), skinMat);
      head.position.y = 1.95;
      head.castShadow = true;
      group.add(head);

      // Hair
      const hair = new THREE.Mesh(new THREE.BoxGeometry(0.64, 0.25, 0.64), hairMat);
      hair.position.y = 2.3;
      group.add(hair);

      // Limbs
      const armMat = mat;
      const leftArm = new THREE.Mesh(new THREE.BoxGeometry(0.25, 0.9, 0.25), armMat);
      leftArm.position.set(-0.55, 1.1, 0);
      group.add(leftArm);

      const rightArm = new THREE.Mesh(new THREE.BoxGeometry(0.25, 0.9, 0.25), armMat);
      rightArm.position.set(0.55, 1.1, 0);
      group.add(rightArm);

      const legMat = new THREE.MeshStandardMaterial({ color: 0x111827 });
      const leftLeg = new THREE.Mesh(new THREE.BoxGeometry(0.3, 0.9, 0.3), legMat);
      leftLeg.position.set(-0.2, 0.45, 0);
      group.add(leftLeg);

      const rightLeg = new THREE.Mesh(new THREE.BoxGeometry(0.3, 0.9, 0.3), legMat);
      rightLeg.position.set(0.2, 0.45, 0);
      group.add(rightLeg);

      group.userData = { id: data.id, name: data.name };
      scene.add(group);
      return group;
    }

    // --- REAL MULTIPLAYER NETWORK SYNC (MQTT / Firebase) ---
    let myRoomCode = 'BB-SEASON-1';
    function initNetworkSync() {
      // Connect to public MQTT Broker for realtime multiplayer positioning & chat
      const brokerUrl = 'wss://broker.hivemq.com:8884/mqtt';
      mqttClient = mqtt.connect(brokerUrl);

      mqttClient.on('connect', () => {
        isConnected = true;
        document.getElementById('lobby-online-pill').innerText = '🌐 Live Multiplayer Connected';
        document.getElementById('lobby-online-pill').className = 'px-2.5 py-1 rounded-full text-[10px] font-bold bg-emerald-5
