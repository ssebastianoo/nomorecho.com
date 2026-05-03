<script lang="ts">
  import bass from "$lib/assets/daw/y-u-8-me/intro-bass.opus";
  import drums from "$lib/assets/daw/y-u-8-me/intro-drums.opus";
  import guitar from "$lib/assets/daw/y-u-8-me/intro-guitar.opus";
  import vocals from "$lib/assets/daw/y-u-8-me/intro-vocals.opus";
  import choir from "$lib/assets/daw/y-u-8-me/intro-choir.opus";

  const BPM = 155;
  const BEAT_DURATION = 60 / BPM;       // seconds per beat
  const BEATS_PER_BAR = 4;
  const BAR_DURATION = BEAT_DURATION * BEATS_PER_BAR; // seconds per bar

  // ── Helpers ───────────────────────────────────────────────────────────────
  function gcd(a: number, b: number): number { return b === 0 ? a : gcd(b, a % b); }
  function lcm(a: number, b: number): number { return (a * b) / gcd(a, b); }

  /** Snap an audio buffer duration to the nearest bar count (min 1 bar). */
  function snapToBars(duration: number): number {
    return Math.max(1, Math.round(duration / BAR_DURATION));
  }

  // ── Types ─────────────────────────────────────────────────────────────────
  type TrackColor = "cyan" | "amber" | "lime" | "violet" | "rose";

  interface SampleDef {
    id: string;
    label: string;
    src: string;
    color: TrackColor;
    icon: string;
  }

  interface LoopTrack {
    id: string;
    label: string;
    src: string;
    color: TrackColor;
    icon: string;
    node: AudioBufferSourceNode | null;
    gainNode: GainNode | null;
    buffer: AudioBuffer | null;
    bars: number;          // detected bar count (4 or 8, etc.)
    trackDuration: number; // bars × BAR_DURATION in seconds
    volume: number;
    muted: boolean;
    loading: boolean;
  }

  // ── Sample palette (drag sources) ─────────────────────────────────────────
  const PALETTE: SampleDef[] = [
    { id: "bass",   label: "BASS",   src: bass,   color: "cyan",   icon: "◎" },
    { id: "drums",  label: "DRUMS",  src: drums,  color: "amber",  icon: "◈" },
    { id: "guitar", label: "GUITAR", src: guitar, color: "lime",   icon: "◇" },
    { id: "vocals", label: "VOCALS", src: vocals, color: "violet", icon: "◉" },
    { id: "choir",  label: "CHOIR",  src: choir,  color: "rose",   icon: "◐" },
  ];

  // ── State ──────────────────────────────────────────────────────────────────
  let audioCtx = $state<AudioContext | null>(null);
  let masterGain = $state<GainNode | null>(null);
  let loopTracks = $state<LoopTrack[]>([]);
  let isPlaying = $state(false);
  let loopStartTime = $state(0);
  let currentBeat = $state(0); // position within the global cycle (0 … globalBeats-1)
  let masterVolume = $state(0.85);
  let dragOverZone = $state(false);
  let draggingId = $state<string | null>(null);
  let tickerInterval = $state<ReturnType<typeof setInterval> | null>(null);

  /**
   * Global cycle = LCM of all loaded track bar-counts.
   * Falls back to 8 bars when no tracks are loaded.
   * A 4-bar track and an 8-bar track → 8-bar cycle (4-bar repeats twice per cycle).
   */
  let globalBars = $derived(
    loopTracks.length === 0
      ? 8
      : loopTracks
          .filter(t => !t.loading && t.bars > 0)
          .reduce((acc, t) => lcm(acc, t.bars), 1) || 8
  );
  let globalDuration = $derived(globalBars * BAR_DURATION);
  let globalBeats = $derived(globalBars * BEATS_PER_BAR);

  // ── Audio helpers ──────────────────────────────────────────────────────────
  function ensureContext() {
    if (!audioCtx) {
      audioCtx = new AudioContext();
      masterGain = audioCtx.createGain();
      masterGain.gain.value = masterVolume;
      masterGain.connect(audioCtx.destination);
    }
    return { ctx: audioCtx!, master: masterGain! };
  }

  async function loadBuffer(src: string): Promise<AudioBuffer> {
    const { ctx } = ensureContext();
    const res = await fetch(src);
    const ab = await res.arrayBuffer();
    return ctx.decodeAudioData(ab);
  }

  /**
   * Schedule a single track's looping source, synced to loopStartTime.
   * Each track loops over its OWN duration (4-bar or 8-bar), so a 4-bar
   * track repeats twice inside an 8-bar global cycle without any drift.
   */
  function scheduleTrack(track: LoopTrack) {
    if (!audioCtx || !masterGain || !track.buffer) return;

    const source = audioCtx.createBufferSource();
    source.buffer = track.buffer;
    source.loop = true;
    // Loop boundary = exact bar-snapped duration, not the raw buffer length
    source.loopStart = 0;
    source.loopEnd = track.trackDuration;

    const gain = audioCtx.createGain();
    gain.gain.value = track.muted ? 0 : track.volume;
    source.connect(gain);
    gain.connect(masterGain);

    // Sync offset: where are we inside THIS track's loop right now?
    const now = audioCtx.currentTime;
    const globalElapsed = isPlaying ? (now - loopStartTime) % globalDuration : 0;
    const offset = globalElapsed % track.trackDuration;

    source.start(now, offset);
    track.node = source;
    track.gainNode = gain;
  }

  function stopTrackNode(track: LoopTrack) {
    try { track.node?.stop(); } catch (_) {}
    track.node = null;
    track.gainNode = null;
  }

  // ── Transport ──────────────────────────────────────────────────────────────
  async function togglePlay() {
    ensureContext();
    if (audioCtx!.state === "suspended") await audioCtx!.resume();

    if (isPlaying) {
      for (const t of loopTracks) stopTrackNode(t);
      if (tickerInterval) clearInterval(tickerInterval);
      tickerInterval = null;
      isPlaying = false;
      currentBeat = 0;
    } else {
      loopStartTime = audioCtx!.currentTime;
      for (const t of loopTracks) {
        if (t.buffer) scheduleTrack(t);
      }
      isPlaying = true;
      startTicker();
    }
  }

  function startTicker() {
    tickerInterval = setInterval(() => {
      if (!audioCtx || !isPlaying) return;
      const elapsed = (audioCtx.currentTime - loopStartTime) % globalDuration;
      currentBeat = Math.floor(elapsed / BEAT_DURATION) % globalBeats;
    }, 30);
  }

  function stop() {
    if (!isPlaying) return;
    for (const t of loopTracks) stopTrackNode(t);
    if (tickerInterval) clearInterval(tickerInterval);
    tickerInterval = null;
    isPlaying = false;
    currentBeat = 0;
    loopStartTime = 0;
  }

  // ── Drag & Drop ────────────────────────────────────────────────────────────
  function onDragStart(e: DragEvent, id: string) {
    draggingId = id;
    e.dataTransfer!.effectAllowed = "copy";
    e.dataTransfer!.setData("text/plain", id);
  }

  function onDragOver(e: DragEvent) {
    e.preventDefault();
    e.dataTransfer!.dropEffect = "copy";
    dragOverZone = true;
  }

  function onDragLeave() { dragOverZone = false; }

  async function onDrop(e: DragEvent) {
    e.preventDefault();
    dragOverZone = false;
    const id = e.dataTransfer?.getData("text/plain") ?? draggingId;
    if (!id) return;
    await addTrack(id);
    draggingId = null;
  }

  async function addTrack(id: string) {
    if (loopTracks.find(t => t.id === id)) return;
    const def = PALETTE.find(p => p.id === id);
    if (!def) return;

    ensureContext();

    const track: LoopTrack = {
      ...def,
      node: null,
      gainNode: null,
      buffer: null,
      bars: 0,
      trackDuration: 0,
      volume: 0.8,
      muted: false,
      loading: true,
    };
    loopTracks = [...loopTracks, track];

    try {
      const buf = await loadBuffer(def.src);
      const idx = loopTracks.findIndex(t => t.id === id);
      if (idx === -1) return;

      const bars = snapToBars(buf.duration);
      const trackDuration = bars * BAR_DURATION;

      loopTracks[idx].buffer = buf;
      loopTracks[idx].bars = bars;
      loopTracks[idx].trackDuration = trackDuration;
      loopTracks[idx].loading = false;

      // If already playing, join in at the right phase immediately
      if (isPlaying) scheduleTrack(loopTracks[idx]);
    } catch (err) {
      console.error("Failed to load", id, err);
      loopTracks = loopTracks.filter(t => t.id !== id);
    }
  }

  function removeTrack(id: string) {
    const idx = loopTracks.findIndex(t => t.id === id);
    if (idx === -1) return;
    stopTrackNode(loopTracks[idx]);
    loopTracks = loopTracks.filter(t => t.id !== id);
  }

  function toggleMute(id: string) {
    const idx = loopTracks.findIndex(t => t.id === id);
    if (idx === -1) return;
    const t = loopTracks[idx];
    t.muted = !t.muted;
    if (t.gainNode) t.gainNode.gain.setTargetAtTime(t.muted ? 0 : t.volume, audioCtx!.currentTime, 0.02);
    loopTracks = [...loopTracks];
  }

  function setVolume(id: string, vol: number) {
    const idx = loopTracks.findIndex(t => t.id === id);
    if (idx === -1) return;
    loopTracks[idx].volume = vol;
    if (!loopTracks[idx].muted && loopTracks[idx].gainNode) {
      loopTracks[idx].gainNode!.gain.setTargetAtTime(vol, audioCtx!.currentTime, 0.02);
    }
    loopTracks = [...loopTracks];
  }

  function setMasterVolume(v: number) {
    masterVolume = v;
    if (masterGain) masterGain.gain.setTargetAtTime(v, audioCtx!.currentTime, 0.02);
  }

  /**
   * For a track with N bars, returns which beat cells (out of globalBeats)
   * are "active" — i.e. within one of its repetitions in the cycle.
   * Used to shade the per-track beat visualizer so short tracks show repeated blocks.
   */
  function trackBeatActive(track: LoopTrack, beat: number): boolean {
    if (!isPlaying || track.muted || track.loading) return false;
    const trackBeats = track.bars * BEATS_PER_BAR;
    return (currentBeat % trackBeats) === (beat % trackBeats) && beat === currentBeat;
  }

  /** Returns true if this beat cell falls inside the track's own loop length */
  function beatInTrackLoop(track: LoopTrack, beat: number): boolean {
    if (track.loading || track.bars === 0) return true;
    const trackBeats = track.bars * BEATS_PER_BAR;
    return (beat % trackBeats) < trackBeats; // always true, but used for styling repeats
  }

  /** Beat index within the track's own loop for fill coloring */
  function localBeat(track: LoopTrack, beat: number): number {
    if (track.bars === 0) return beat;
    return beat % (track.bars * BEATS_PER_BAR);
  }

  function isRepeatBar(track: LoopTrack, beat: number): boolean {
    if (track.bars === 0 || track.loading) return false;
    const trackBeats = track.bars * BEATS_PER_BAR;
    return beat >= trackBeats && beat % trackBeats === 0;
  }
</script>

<div class="daw">
	<!-- ── Header ── -->
	<header class="daw-header">
		<div class="logo">
			<span class="logo-mark">◈</span>
			<span class="logo-text">LOOP<em>CRAFT</em></span>
		</div>
		<div class="bpm-badge">{BPM} BPM · {globalBars} BARS</div>
		<div class="header-right">
			<label class="master-vol-label">MASTER</label>
			<input
				type="range"
				min="0"
				max="1"
				step="0.01"
				value={masterVolume}
				oninput={(e) => setMasterVolume(+(e.target as HTMLInputElement).value)}
				class="vol-slider master-slider"
			/>
		</div>
	</header>

	<!-- ── Beat grid ── -->
	<div class="beat-grid" style="grid-template-columns: repeat({globalBeats}, 1fr)">
		{#each Array(globalBeats) as _, i}
			<div
				class="beat-cell"
				class:beat-active={isPlaying && currentBeat === i}
				class:beat-bar-start={i % BEATS_PER_BAR === 0}
			>
				{#if i % BEATS_PER_BAR === 0}
					<span class="bar-num">{Math.floor(i / BEATS_PER_BAR) + 1}</span>
				{/if}
			</div>
		{/each}
	</div>

	<!-- ── Transport ── -->
	<div class="transport">
		<button class="btn-stop" onclick={stop} disabled={!isPlaying} aria-label="Stop">
			<svg width="14" height="14" viewBox="0 0 14 14"
				><rect x="1" y="1" width="12" height="12" /></svg
			>
		</button>
		<button class="btn-play" onclick={togglePlay} aria-label={isPlaying ? 'Pause' : 'Play'}>
			{#if isPlaying}
				<svg width="18" height="18" viewBox="0 0 18 18">
					<rect x="2" y="1" width="5" height="16" />
					<rect x="11" y="1" width="5" height="16" />
				</svg>
			{:else}
				<svg width="18" height="18" viewBox="0 0 18 18">
					<polygon points="2,1 16,9 2,17" />
				</svg>
			{/if}
		</button>
		<div class="playhead-info">
			{#if isPlaying}
				<span class="ph-bar">BAR {Math.floor(currentBeat / BEATS_PER_BAR) + 1}</span>
				<span class="ph-beat">BEAT {(currentBeat % BEATS_PER_BAR) + 1}</span>
				<span class="ph-cycle">/ {globalBars} BARS</span>
			{:else}
				<span class="ph-idle">STOPPED</span>
			{/if}
		</div>
	</div>

	<!-- ── Main area ── -->
	<div class="main">
		<!-- Sample Palette -->
		<aside class="palette">
			<div class="section-label">SAMPLES</div>
			<div class="palette-hint">drag to loop</div>
			<div class="palette-items">
				{#each PALETTE as s}
					{@const alreadyAdded = loopTracks.some((t) => t.id === s.id)}
					<div
						class="palette-chip color-{s.color}"
						class:chip-added={alreadyAdded}
						draggable={!alreadyAdded}
						ondragstart={(e) => onDragStart(e, s.id)}
						role="button"
						tabindex="0"
						aria-label="Drag {s.label} to loop"
					>
						<span class="chip-icon">{s.icon}</span>
						<span class="chip-label">{s.label}</span>
						{#if alreadyAdded}
							<span class="chip-check">✓</span>
						{:else}
							<span class="chip-drag">⠿</span>
						{/if}
					</div>
				{/each}
			</div>
		</aside>

		<!-- Loop zone -->
		<section
			class="loop-zone"
			class:drop-active={dragOverZone}
			ondragover={onDragOver}
			ondragleave={onDragLeave}
			ondrop={onDrop}
			aria-label="Loop drop zone"
		>
			{#if loopTracks.length === 0}
				<div class="empty-state">
					<div class="empty-icon">⊕</div>
					<div class="empty-text">Drop samples here to build your loop</div>
				</div>
			{:else}
				<div class="tracks">
					{#each loopTracks as track (track.id)}
						<div class="track color-{track.color}" class:track-muted={track.muted}>
							<!-- Track info -->
							<div class="track-left">
								<span class="track-icon">{track.icon}</span>
								<span class="track-name">{track.label}</span>
							</div>

							<!-- Beat visualizer -->
							<div class="track-beats" style="grid-template-columns: repeat({globalBeats}, 1fr)">
								{#each Array(globalBeats) as _, i}
									{@const active = trackBeatActive(track, i)}
									{@const isRepeat = isRepeatBar(track, i)}
									<div
										class="track-beat"
										class:tb-active={active}
										class:tb-bar={i % BEATS_PER_BAR === 0}
										class:tb-repeat={isRepeat}
									></div>
								{/each}
								{#if track.loading}
									<div class="loading-bar"><span>LOADING…</span></div>
								{/if}
							</div>

							<!-- Controls -->
							<div class="track-controls">
								{#if !track.loading && track.bars > 0}
									<span class="bars-badge">{track.bars}B</span>
								{/if}
								<button
									class="mute-btn"
									class:muted={track.muted}
									onclick={() => toggleMute(track.id)}
									title={track.muted ? 'Unmute' : 'Mute'}
								>
									{track.muted ? 'M' : 'M'}
								</button>
								<input
									type="range"
									min="0"
									max="1"
									step="0.01"
									value={track.volume}
									oninput={(e) => setVolume(track.id, +(e.target as HTMLInputElement).value)}
									class="vol-slider"
									title="Volume"
								/>
								<button class="remove-btn" onclick={() => removeTrack(track.id)} title="Remove"
									>✕</button
								>
							</div>
						</div>
					{/each}
				</div>
			{/if}
		</section>
	</div>
</div>

<style>
  /* ── Fonts & Root ── */
  @import url('https://fonts.googleapis.com/css2?family=IBM+Plex+Mono:ital,wght@0,400;0,600;0,700;1,400&display=swap');

  :root {
    --bg0: #0a0a0b;
    --bg1: #111113;
    --bg2: #18181c;
    --bg3: #222228;
    --border: #2a2a32;
    --border-bright: #3a3a46;
    --text: #c8c8d4;
    --text-dim: #555566;
    --text-bright: #eeeef8;

    --cyan:   #00e5ff;
    --amber:  #ffb300;
    --lime:   #aaff00;
    --violet: #b085ff;
    --rose:   #ff4d82;

    --track-h: 58px;
    --radius: 2px;
  }

  * { box-sizing: border-box; margin: 0; padding: 0; }

  .daw {
    font-family: 'IBM Plex Mono', monospace;
    background: var(--bg0);
    color: var(--text);
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    border: 1px solid var(--border);
  }

  /* ── Header ── */
  .daw-header {
    display: flex;
    align-items: center;
    gap: 20px;
    padding: 12px 20px;
    background: var(--bg1);
    border-bottom: 1px solid var(--border);
  }

  .logo {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .logo-mark {
    font-size: 1.4rem;
    color: var(--cyan);
    line-height: 1;
  }

  .logo-text {
    font-size: 0.9rem;
    font-weight: 700;
    letter-spacing: 0.18em;
    color: var(--text-bright);
  }

  .logo-text em {
    font-style: normal;
    color: var(--cyan);
  }

  .bpm-badge {
    font-size: 0.65rem;
    letter-spacing: 0.1em;
    color: var(--text-dim);
    border: 1px solid var(--border-bright);
    padding: 3px 8px;
    border-radius: var(--radius);
  }

  .header-right {
    margin-left: auto;
    display: flex;
    align-items: center;
    gap: 10px;
  }

  .master-vol-label {
    font-size: 0.6rem;
    letter-spacing: 0.15em;
    color: var(--text-dim);
  }

  /* ── Beat Grid ── */
  .beat-grid {
    display: grid;
    /* grid-template-columns set inline via style= */
    height: 28px;
    background: var(--bg1);
    border-bottom: 2px solid var(--border);
    padding: 0 0 0 200px; /* align with track content */
  }

  .beat-cell {
    position: relative;
    border-left: 1px solid var(--border);
    display: flex;
    align-items: flex-end;
    padding-bottom: 3px;
    transition: background 0.04s;
  }

  .beat-cell.beat-bar-start {
    border-left-color: var(--border-bright);
  }

  .beat-cell.beat-active {
    background: rgba(0, 229, 255, 0.08);
  }

  .bar-num {
    font-size: 0.55rem;
    color: var(--text-dim);
    padding-left: 4px;
    letter-spacing: 0.05em;
  }

  /* ── Transport ── */
  .transport {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 10px 20px;
    background: var(--bg1);
    border-bottom: 1px solid var(--border);
  }

  .btn-play, .btn-stop {
    display: flex;
    align-items: center;
    justify-content: center;
    border: 1px solid var(--border-bright);
    background: var(--bg2);
    color: var(--text-bright);
    cursor: pointer;
    border-radius: var(--radius);
    transition: background 0.12s, border-color 0.12s;
  }

  .btn-play {
    width: 44px;
    height: 36px;
  }

  .btn-stop {
    width: 34px;
    height: 36px;
    color: var(--text-dim);
  }

  .btn-play svg { fill: var(--cyan); }
  .btn-stop svg { fill: var(--text-dim); }

  .btn-play:hover { background: var(--bg3); border-color: var(--cyan); }
  .btn-stop:hover:not(:disabled) { background: var(--bg3); }
  .btn-stop:disabled { opacity: 0.3; cursor: default; }

  .playhead-info {
    display: flex;
    gap: 12px;
    font-size: 0.7rem;
    letter-spacing: 0.1em;
  }

  .ph-bar { color: var(--cyan); }
  .ph-beat { color: var(--text-dim); }
  .ph-idle { color: var(--text-dim); }

  /* ── Main layout ── */
  .main {
    display: flex;
    flex: 1;
    overflow: hidden;
  }

  /* ── Palette ── */
  .palette {
    width: 140px;
    flex-shrink: 0;
    background: var(--bg1);
    border-right: 1px solid var(--border);
    padding: 14px 10px;
    display: flex;
    flex-direction: column;
    gap: 6px;
  }

  .section-label {
    font-size: 0.6rem;
    letter-spacing: 0.18em;
    color: var(--text-dim);
    margin-bottom: 2px;
  }

  .palette-hint {
    font-size: 0.55rem;
    color: var(--text-dim);
    margin-bottom: 8px;
    font-style: italic;
  }

  .palette-items {
    display: flex;
    flex-direction: column;
    gap: 5px;
  }

  .palette-chip {
    display: flex;
    align-items: center;
    gap: 7px;
    padding: 7px 8px;
    border-radius: var(--radius);
    border: 1px solid var(--border);
    background: var(--bg2);
    cursor: grab;
    user-select: none;
    transition: border-color 0.12s, background 0.12s, opacity 0.12s;
    font-size: 0.65rem;
    letter-spacing: 0.08em;
  }

  .palette-chip:active { cursor: grabbing; }

  .palette-chip.chip-added {
    opacity: 0.4;
    cursor: default;
  }

  /* Color accents on chips */
  .color-cyan   { border-left: 2px solid var(--cyan); }
  .color-amber  { border-left: 2px solid var(--amber); }
  .color-lime   { border-left: 2px solid var(--lime); }
  .color-violet { border-left: 2px solid var(--violet); }
  .color-rose   { border-left: 2px solid var(--rose); }

  .palette-chip:not(.chip-added):hover {
    background: var(--bg3);
  }

  .palette-chip.color-cyan:not(.chip-added):hover   { border-color: var(--cyan); }
  .palette-chip.color-amber:not(.chip-added):hover  { border-color: var(--amber); }
  .palette-chip.color-lime:not(.chip-added):hover   { border-color: var(--lime); }
  .palette-chip.color-violet:not(.chip-added):hover { border-color: var(--violet); }
  .palette-chip.color-rose:not(.chip-added):hover   { border-color: var(--rose); }

  .chip-icon { font-size: 0.85rem; }
  .chip-label { flex: 1; }
  .chip-check { color: #44ff88; font-size: 0.7rem; }
  .chip-drag { color: var(--text-dim); font-size: 0.8rem; }

  /* ── Loop Zone ── */
  .loop-zone {
    flex: 1;
    background: var(--bg0);
    padding: 16px;
    overflow-y: auto;
    transition: background 0.15s;
    min-height: 200px;
    border: 2px dashed transparent;
    border-radius: var(--radius);
    margin: 8px;
  }

  .loop-zone.drop-active {
    background: rgba(0, 229, 255, 0.03);
    border-color: rgba(0, 229, 255, 0.3);
  }

  .empty-state {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    height: 200px;
    gap: 12px;
    opacity: 0.35;
  }

  .empty-icon {
    font-size: 2.5rem;
    color: var(--text-dim);
  }

  .empty-text {
    font-size: 0.7rem;
    letter-spacing: 0.1em;
    color: var(--text-dim);
  }

  /* ── Tracks ── */
  .tracks {
    display: flex;
    flex-direction: column;
    gap: 6px;
  }

  .track {
    display: flex;
    align-items: center;
    height: var(--track-h);
    background: var(--bg2);
    border: 1px solid var(--border);
    border-radius: var(--radius);
    overflow: hidden;
    transition: opacity 0.15s;
  }

  .track.track-muted {
    opacity: 0.45;
  }

  .track-left {
    width: 100px;
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 0 12px;
    border-right: 1px solid var(--border);
    height: 100%;
  }

  .track-icon {
    font-size: 1rem;
  }

  .track-name {
    font-size: 0.62rem;
    letter-spacing: 0.12em;
    color: var(--text-bright);
    font-weight: 600;
  }

  /* Color left border per track */
  .track.color-cyan   { border-left: 3px solid var(--cyan); }
  .track.color-amber  { border-left: 3px solid var(--amber); }
  .track.color-lime   { border-left: 3px solid var(--lime); }
  .track.color-violet { border-left: 3px solid var(--violet); }
  .track.color-rose   { border-left: 3px solid var(--rose); }

  .track.color-cyan .track-icon   { color: var(--cyan); }
  .track.color-amber .track-icon  { color: var(--amber); }
  .track.color-lime .track-icon   { color: var(--lime); }
  .track.color-violet .track-icon { color: var(--violet); }
  .track.color-rose .track-icon   { color: var(--rose); }

  /* ── Beat mini-visualizer ── */
  .track-beats {
    flex: 1;
    display: grid;
    /* grid-template-columns set inline via style= */
    height: 100%;
    position: relative;
    overflow: hidden;
  }

  .track-beat {
    border-left: 1px solid var(--border);
    transition: background 0.04s;
  }

  .track-beat.tb-bar {
    border-left-color: var(--border-bright);
  }

  /* Active beat highlight per color */
  .track.color-cyan   .track-beat.tb-active { background: rgba(0,229,255,0.22); }
  .track.color-amber  .track-beat.tb-active { background: rgba(255,179,0,0.22); }
  .track.color-lime   .track-beat.tb-active { background: rgba(170,255,0,0.22); }
  .track.color-violet .track-beat.tb-active { background: rgba(176,133,255,0.22); }
  .track.color-rose   .track-beat.tb-active { background: rgba(255,77,130,0.22); }

  .track-beat.tb-repeat {
    border-left: 1px dashed var(--border-bright);
    opacity: 0.6;
  }

  .bars-badge {
    font-size: 0.55rem;
    letter-spacing: 0.08em;
    color: var(--text-dim);
    border: 1px solid var(--border);
    padding: 2px 4px;
    border-radius: var(--radius);
    flex-shrink: 0;
  }

  .ph-cycle {
    color: var(--text-dim);
    font-size: 0.65rem;
  }

  .loading-bar {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    background: rgba(10,10,11,0.7);
    font-size: 0.6rem;
    letter-spacing: 0.15em;
    color: var(--text-dim);
    animation: pulse 1s infinite;
  }

  @keyframes pulse {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.4; }
  }

  /* ── Track Controls ── */
  .track-controls {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 0 10px;
    height: 100%;
    border-left: 1px solid var(--border);
    background: var(--bg1);
    flex-shrink: 0;
  }

  .mute-btn {
    font-family: 'IBM Plex Mono', monospace;
    font-size: 0.6rem;
    font-weight: 700;
    letter-spacing: 0.1em;
    padding: 3px 6px;
    border-radius: var(--radius);
    border: 1px solid var(--border-bright);
    background: var(--bg2);
    color: var(--text-dim);
    cursor: pointer;
    transition: background 0.1s, color 0.1s, border-color 0.1s;
  }

  .mute-btn.muted {
    background: var(--amber);
    color: #000;
    border-color: var(--amber);
  }

  .mute-btn:hover:not(.muted) {
    background: var(--bg3);
    color: var(--text-bright);
  }

  .remove-btn {
    font-size: 0.7rem;
    padding: 3px 6px;
    border-radius: var(--radius);
    border: 1px solid var(--border);
    background: transparent;
    color: var(--text-dim);
    cursor: pointer;
    transition: background 0.1s, color 0.1s, border-color 0.1s;
  }

  .remove-btn:hover {
    background: rgba(255,77,130,0.15);
    border-color: var(--rose);
    color: var(--rose);
  }

  /* ── Volume Sliders ── */
  .vol-slider {
    -webkit-appearance: none;
    appearance: none;
    height: 2px;
    border-radius: 1px;
    background: var(--border-bright);
    cursor: pointer;
    outline: none;
  }

  .vol-slider { width: 70px; }
  .master-slider { width: 100px; }

  .vol-slider::-webkit-slider-thumb {
    -webkit-appearance: none;
    width: 10px;
    height: 10px;
    border-radius: 50%;
    background: var(--text-bright);
    border: 1px solid var(--border-bright);
    cursor: pointer;
    transition: background 0.1s;
  }

  .vol-slider:hover::-webkit-slider-thumb {
    background: var(--cyan);
  }

  .master-slider::-webkit-slider-thumb {
    background: var(--cyan);
  }
</style>
