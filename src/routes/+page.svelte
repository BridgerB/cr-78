<script>
  import { onMount, onDestroy } from 'svelte';

  // Application state
  let sequenceLength = $state(16);
  let sequence = $state({});
  let pos = $state(0);
  let paused = $state(true);
  let bpm = $state(120);
  let swing = $state(1);
  let audioBuffers = {};
  let audioContext = null;
  let baseline = null; // Timing baseline for drift compensation
  let timerId = null;
  let mutedSounds = $state({});

  // Define the drum sounds for CR-78
  const drumSounds = [
    'kicka', 'kickb', 'snarea', 'snareb', 'rim',
    'hihata', 'hihatb', 'hihatc', 'cymbal',
    'bongohigh', 'bongolow', 'congalow', 'cowbell',
    'tamb1', 'tamb2'
  ];

  // Initialize sequence immediately for SSR compatibility
  function createInitialSequence() {
    const seq = {};
    for (const sound of drumSounds) {
      seq[sound] = Array(sequenceLength).fill(false);
    }
    return seq;
  }

  function createInitialMutedSounds() {
    const muted = {};
    for (const sound of drumSounds) {
      muted[sound] = false;
    }
    return muted;
  }

  // Set initial state
  sequence = createInitialSequence();
  mutedSounds = createInitialMutedSounds();

  // Initialize sequence and muted sounds
  onMount(async () => {
    // Set up audio context
    audioContext = new (window.AudioContext || window.webkitAudioContext)();

    // Load audio samples
    await loadAudioSamples();

    // Set up default pattern (simple back beat)
    for (let x = 0; x < sequenceLength; x += 4) {
      sequence.kicka[x] = true;
    }
    for (let x = 0; x < sequenceLength; x += 1) {
      sequence.hihata[x] = true;
    }
  });

  // Update sequence when length changes
  $effect(() => {
    if (sequence && Object.keys(sequence).length > 0) {
      for (const sound in sequence) {
        if (sequence[sound].length !== sequenceLength) {
          const newSequence = Array(sequenceLength).fill(false);
          const minLength = Math.min(sequence[sound].length, sequenceLength);
          for (let i = 0; i < minLength; i++) {
            newSequence[i] = sequence[sound][i];
          }
          sequence[sound] = newSequence;
        }
      }
    }
  });

  // Import all audio files using Vite's glob import
  const audioFiles = import.meta.glob('$lib/assets/audio/*.wav', { eager: true, import: 'default' });

  // Load audio samples
  async function loadAudioSamples() {
    const promises = drumSounds.map(async (sound) => {
      const fileName = `cr78-${formatSoundName(sound)}.wav`;
      const audioPath = audioFiles[`/src/lib/assets/audio/${fileName}`];
      const response = await fetch(audioPath);
      const arrayBuffer = await response.arrayBuffer();
      const audioBuffer = await audioContext.decodeAudioData(arrayBuffer);
      audioBuffers[sound] = audioBuffer;
    });

    await Promise.all(promises);
  }

  // Format sound name for audio file
  function formatSoundName(sound) {
    switch (sound) {
      case 'kicka': return 'Kick';
      case 'kickb': return 'Kick Accent';
      case 'snarea': return 'Snare';
      case 'snareb': return 'Snare Accent';
      case 'rim': return 'Rim Shot';
      case 'hihata': return 'HiHat';
      case 'hihatb': return 'HiHat Accent';
      case 'hihatc': return 'HiHat Metal';
      case 'cymbal': return 'Cymbal';
      case 'bongohigh': return 'Bongo High';
      case 'bongolow': return 'Bongo Low';
      case 'congalow': return 'Conga Low';
      case 'cowbell': return 'Cowbell';
      case 'tamb1': return 'Tamb 1';
      case 'tamb2': return 'Tamb 2';
      default: return sound;
    }
  }

  // Play a sound
  function playSound(sound) {
    if (audioBuffers[sound] && audioContext) {
      const source = audioContext.createBufferSource();
      source.buffer = audioBuffers[sound];
      source.connect(audioContext.destination);
      source.start();
    }
  }

  // Calculate interval time in milliseconds based on BPM
  function calculateInterval() {
    return (60000 / (bpm * 4)); // 4 steps per beat
  }

  // Schedule steps with precise timing following original drummer.js algorithm
  function scheduler() {
    if (paused) return;

    // Initialize baseline on first run
    if (baseline === null) {
      baseline = new Date().getTime();
    }

    // Play sounds for this step
    for (const sound in sequence) {
      if (sequence[sound][pos] && !mutedSounds[sound]) {
        playSound(sound);
      }
    }

    // Move to next position
    pos = (pos + 1) % sequenceLength;

    // Calculate time interval from bpm
    // Original uses: duration = 60000 / (app.bpm*2) for swing timing
    let duration = 60000 / (bpm * 2);

    // is this an on or off beat (checking the NEW position after incrementing)
    const odd = ((pos % 2) !== 0);

    // swing algorithm from original drummer.js
    let percent = 50;
    switch(parseInt(swing)) {
      case 2: percent = 54; break;
      case 3: percent = 58; break;
      case 4: percent = 62; break;
      case 5: percent = 66; break;
      case 6: percent = 71; break;
      case 1:
      default:
        percent = 50; break;
    }
    // Apply original adjustment: percent = 50 + (percent - 50) / 4
    percent = 50 + (percent - 50) / 4;

    let stepDuration;
    if (!odd) {
      stepDuration = duration * ((100 - percent)/100);
    } else {
      stepDuration = duration * (percent/100);
    }

    // Drift compensation algorithm from original drummer.js
    const end = new Date().getTime();
    baseline += stepDuration;

    let nextTick = stepDuration - (end - baseline);
    if (nextTick < 0) {
      nextTick = 0;
    }

    // Schedule next step with drift-compensated duration
    timerId = setTimeout(() => {
      scheduler();
    }, nextTick);
  }

  // Start playback
  async function startPlayback() {
    if (paused) {
      paused = false;
      // Resume audio context if suspended (needed for user interaction)
      if (audioContext && audioContext.state === 'suspended') {
        try {
          await audioContext.resume();
        } catch (e) {
          console.log('Could not resume audio context');
        }
      }
      // Reset position to 0 when starting
      pos = 0;
      // Reset timing baseline for fresh start
      baseline = null;
      scheduler();
    }
  }

  // Stop playback
  function stopPlayback() {
    paused = true;
    if (timerId) {
      clearTimeout(timerId);
      timerId = null;
    }
    // Reset timing baseline
    baseline = null;
  }

  // Clear sequence
  function clearSequence() {
    for (const sound in sequence) {
      sequence[sound] = Array(sequenceLength).fill(false);
    }
  }

  // Handle sequence cell click
  function toggleStep(sound, step) {
    sequence[sound][step] = !sequence[sound][step];
  }

  // Toggle mute for a sound
  function toggleMute(sound) {
    mutedSounds[sound] = !mutedSounds[sound];
  }

  // Clean up on unmount
  onDestroy(() => {
    stopPlayback();
    if (audioContext) {
      audioContext.close();
    }
  });
</script>

<svelte:head>
  <title>CR-78 Drum Machine</title>
</svelte:head>

<div id="app">
  <header>
    <h1>CR-78 Drum Machine</h1>
  </header>

  <div class="machine-image">
    <img src="/cr78.png" alt="Roland CR-78" />
  </div>

  <div class="controls">
      <div class="control-group">
        <button
          onclick={startPlayback}
          class="control-btn play-btn"
          disabled={!paused}
        >
          ▶ Play
        </button>
        <button
          onclick={stopPlayback}
          class="control-btn stop-btn"
          disabled={paused}
        >
          ■ Stop
        </button>
        <button onclick={clearSequence} class="control-btn clear-btn">
          Clear
        </button>
      </div>
      
      <div class="control-group">
        <label for="bpm">BPM:</label>
        <input 
          id="bpm" 
          type="number" 
          min="20" 
          max="200" 
          bind:value={bpm} 
          class="bpm-input"
        />
      </div>
      
      <div class="control-group">
        <label for="swing">Swing:</label>
        <input 
          id="swing" 
          type="number" 
          min="1" 
          max="6" 
          bind:value={swing} 
          class="swing-input"
        />
      </div>
      
      <div class="control-group">
        <label for="length">Length:</label>
        <input 
          id="length" 
          type="number" 
          min="3" 
          max="32" 
          bind:value={sequenceLength} 
          class="length-input"
        />
      </div>
    </div>

    <div id="sequence" class="sequence-container">
      <table class="table sequence-table">
        <thead>
          <tr>
            <th>Sound</th>
            {#each Array(sequenceLength) as _, i}
              <th colspan="1" class="step-header {i === pos ? 'active-step' : ''}">{i + 1}</th>
            {/each}
            <th>Mute</th>
          </tr>
        </thead>
        <tbody>
        {#each drumSounds as sound}
          <tr>
            <th>{sound}</th>
            {#each Array(sequenceLength) as _, j}
              <td class="inactive {j === pos ? 'active' : ''}">
                <input
                  type="checkbox"
                  checked={sequence[sound][j]}
                  onchange={() => toggleStep(sound, j)}
                  class="step-checkbox"
                />
              </td>
            {/each}
            <td>
              <input
                type="checkbox"
                checked={mutedSounds[sound]}
                onchange={() => toggleMute(sound)}
                class="mute-checkbox mute"
              />
            </td>
          </tr>
        {/each}
        </tbody>
      </table>
    </div>
</div>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    font-family: Arial, sans-serif;
    background-color: #f0f0f0;
  }

  #app {
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
  }

  header {
    text-align: center;
    margin-bottom: 20px;
  }

  header h1 {
    color: #333;
    margin: 0;
  }

  .machine-image {
    text-align: center;
    margin-bottom: 20px;
  }

  .machine-image img {
    max-width: 100%;
    max-height: 300px;
  }

  .controls {
    display: flex;
    flex-wrap: wrap;
    gap: 20px;
    margin-bottom: 20px;
    padding: 15px;
    background-color: #e0e0e0;
    border-radius: 8px;
  }

  .control-group {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .control-btn {
    padding: 8px 16px;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 14px;
  }

  .play-btn {
    background-color: #4CAF50;
    color: white;
    padding: 12px 24px;
    font-size: 16px;
    font-weight: bold;
  }

  .play-btn:hover:not(:disabled) {
    background-color: #45a049;
  }

  .play-btn:disabled {
    background-color: #cccccc;
    cursor: not-allowed;
  }

  .stop-btn {
    background-color: #f44336;
    color: white;
  }

  .stop-btn:disabled {
    background-color: #cccccc;
  }

  .clear-btn {
    background-color: #ff9800;
    color: white;
  }

  input[type="number"] {
    width: 80px;
    padding: 5px;
    border: 1px solid #ccc;
    border-radius: 4px;
  }

  .sequence-container {
    overflow-x: auto;
  }

  .sequence-table {
    width: 100%;
    border-collapse: collapse;
    min-width: 800px;
  }

  .sequence-table th,
  .sequence-table td {
    padding: 8px;
    text-align: center;
    border: 1px solid #ddd;
  }

  .sequence-table th {
    background-color: #f2f2f2;
  }

  .step-header.active-step {
    background-color: #b2dfdb;
  }

  .inactive {
    border-left: 1px solid white;
  }

  .active {
    border-left: 1px dotted #888;
  }

  .step-checkbox {
    width: 20px;
    height: 20px;
    cursor: pointer;
  }

  .mute-checkbox {
    width: 20px;
    height: 20px;
    cursor: pointer;
  }

  .mute {
    border-top: medium none !important;
  }

  @media (max-width: 768px) {
    #app {
      padding: 10px;
    }

    .controls {
      flex-direction: column;
      align-items: flex-start;
    }
  }
</style>