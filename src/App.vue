<template>
  <main class="app">
    <header class="hero">
      <h1>
        <svg class="logo" viewBox="0 0 24 24" aria-hidden="true">
          <path d="M12 3v10.55c-.59-.34-1.27-.55-2-.55-2.21 0-4 1.79-4 4s1.79 4 4 4 4-1.79 4-4V7h4V3h-6z" />
        </svg>
        Music Playlist Generator
      </h1>
      <p class="tagline">Pick a genre and let AI build you a 5-track playlist.</p>
    </header>

    <div class="controls">
      <div class="select-wrap" :class="{ open: dropdownOpen }">
        <button
          type="button"
          class="select-trigger"
          :class="{ placeholder: !genre }"
          aria-haspopup="listbox"
          :aria-expanded="dropdownOpen"
          @click="dropdownOpen = !dropdownOpen"
        >
          {{ genre || "Choose a genre…" }}
        </button>
        <ul v-if="dropdownOpen" class="options" role="listbox">
          <li
            v-for="g in genres"
            :key="g"
            class="option"
            :class="{ selected: g === genre }"
            role="option"
            :aria-selected="g === genre"
            @click="selectGenre(g)"
          >
            {{ g }}
          </li>
        </ul>
      </div>
      <button :disabled="!genre || loading" @click="generate">
        {{ loading ? 'Generating…' : 'Generate' }}
      </button>
    </div>

    <!-- while the request is in flight -->
    <p v-if="loading" class="state spinner">Loading your playlist…</p>

    <!-- something went wrong -->
    <div v-else-if="error" class="state error">
      <p>⚠️ {{ error }}</p>
      <button class="retry" @click="generate">Try again</button>
    </div>

    <!-- got a playlist -->
    <section v-else-if="playlist.length" class="results">
      <div class="results-head">
        <h2>{{ resultGenre }} playlist</h2>
        <span class="badge" :class="source">
          {{ source === 'ai' ? 'AI-generated' : 'Sample list' }}
        </span>
      </div>
      <ol class="playlist">
        <li v-for="(song, i) in playlist" :key="i" class="song">
          <span class="num">{{ i + 1 }}</span>
          <span class="meta">
            <strong class="title">{{ song.title }}</strong>
            <span class="artist">{{ song.artist }}</span>
          </span>
        </li>
      </ol>
    </section>

    <!-- nothing generated yet -->
    <p v-else class="state hint">Your playlist will appear here.</p>
  </main>
</template>

<script>
// backend url, override with VITE_API_URL when deploying
const API_URL =
  (import.meta.env.VITE_API_URL || "http://127.0.0.1:8000") + "/api/playlist";

export default {
  name: "App",
  data() {
    return {
      genres: ["Pop", "Jazz", "Rock", "Hip-Hop", "Classical", "Afrobeats"],
      genre: "",
      dropdownOpen: false,
      playlist: [],
      resultGenre: "",
      source: "",
      loading: false,
      error: "",
    };
  },
  mounted() {
    document.addEventListener("click", this.onDocClick);
  },
  beforeUnmount() {
    document.removeEventListener("click", this.onDocClick);
  },
  methods: {
    selectGenre(g) {
      this.genre = g;
      this.dropdownOpen = false;
    },
    onDocClick(e) {
      // close dropdown on an outside click
      if (!this.$el.contains(e.target)) this.dropdownOpen = false;
    },
    async generate() {
      this.loading = true;
      this.error = "";
      this.playlist = [];
      try {
        const res = await fetch(API_URL, {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ genre: this.genre }),
        });
        if (!res.ok) throw new Error(`Request failed (${res.status})`);
        const data = await res.json();
        this.playlist = data.playlist;
        this.resultGenre = data.genre;
        this.source = data.source;
      } catch (e) {
        this.error =
          "Couldn't load your playlist. Make sure the backend is running, then try again.";
      } finally {
        this.loading = false;
      }
    },
  },
};
</script>

<style scoped>
.hero {
  text-align: center;
  margin-bottom: 2rem;
}
h1 {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  font-size: 2rem;
  font-weight: 800;
  margin: 0 0 0.35rem;
  color: var(--text);
  letter-spacing: -0.03em;
}
.logo {
  width: 1.7rem;
  height: 1.7rem;
  flex-shrink: 0;
  fill: var(--accent);
}
.tagline {
  color: var(--muted);
  margin: 0;
}

.controls {
  display: flex;
  gap: 0.6rem;
  margin-bottom: 1.5rem;
}
.select-wrap {
  position: relative;
  flex: 1;
}
/* custom chevron so we can animate it */
.select-wrap::after {
  content: "";
  position: absolute;
  right: 1.1rem;
  top: 50%;
  width: 0.5rem;
  height: 0.5rem;
  border-right: 2px solid var(--muted);
  border-bottom: 2px solid var(--muted);
  transform: translateY(-65%) rotate(45deg);
  transition: transform 0.25s ease, border-color 0.15s ease;
  pointer-events: none;
}
.select-wrap.open::after {
  transform: translateY(-35%) rotate(225deg);
  border-color: var(--accent);
}
.select-trigger {
  width: 100%;
  text-align: left;
  padding: 0.7rem 2.5rem 0.7rem 0.9rem;
  background: var(--card);
  color: var(--text);
  border: 2px solid var(--border);
  border-radius: 10px;
  font-size: 1rem;
  font-family: inherit;
  font-weight: 500;
  cursor: pointer;
}
.select-trigger.placeholder {
  color: var(--muted);
}
/* just the border on hover, not the whole box */
.select-trigger:hover:not(:disabled) {
  background: var(--card);
  border-color: var(--accent);
  transform: none;
}
.select-wrap.open .select-trigger {
  border-color: var(--accent);
}

/* the open menu */
.options {
  position: absolute;
  top: calc(100% + 0.4rem);
  left: 0;
  right: 0;
  z-index: 10;
  margin: 0;
  padding: 0.35rem;
  list-style: none;
  background: var(--card);
  border: 1px solid var(--border);
  border-radius: 14px;
  box-shadow: 0 12px 28px rgba(0, 0, 0, 0.45);
}
.option {
  padding: 0.6rem 0.75rem;
  border-radius: 8px;
  font-size: 0.95rem;
  cursor: pointer;
  transition: background 0.12s ease, color 0.12s ease;
}
.option:hover {
  background: #2a2a2a;
}
.option.selected {
  color: var(--accent);
  font-weight: 600;
}
button {
  padding: 0.7rem 1.6rem;
  border: none;
  border-radius: 999px;
  font-size: 1rem;
  font-weight: 700;
  color: #000;
  background: var(--accent);
  cursor: pointer;
  transition: transform 0.1s ease, background 0.15s ease;
}
button:hover:not(:disabled) {
  background: var(--accent-hover);
  transform: scale(1.03);
}
button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.state {
  text-align: center;
  padding: 2rem 1rem;
  color: var(--muted);
}
.spinner {
  animation: pulse 1.2s ease-in-out infinite;
}
@keyframes pulse {
  50% { opacity: 0.45; }
}
.error {
  color: #ff8a8a;
}
.retry {
  margin-top: 0.75rem;
}

.results-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 0.75rem;
}
.results-head h2 {
  font-size: 1.2rem;
  margin: 0;
}
.badge {
  font-size: 0.72rem;
  font-weight: 600;
  padding: 0.25rem 0.6rem;
  border-radius: 999px;
  border: 1px solid var(--border);
  color: var(--muted);
}
.badge.ai {
  color: var(--accent);
  border-color: var(--accent);
}

.playlist {
  list-style: none;
  padding: 0;
  margin: 0;
}
.song {
  display: flex;
  align-items: center;
  gap: 0.9rem;
  padding: 0.85rem 1rem;
  background: var(--card);
  border: 1px solid var(--border);
  border-radius: 12px;
  margin-bottom: 0.6rem;
  transition: background 0.15s ease;
}
.song:hover {
  background: #222222;
}
.num {
  width: 1.6rem;
  height: 1.6rem;
  flex-shrink: 0;
  display: grid;
  place-items: center;
  border-radius: 50%;
  background: rgba(29, 185, 84, 0.18);
  color: var(--accent);
  font-size: 0.85rem;
  font-weight: 700;
}
.meta {
  display: flex;
  flex-direction: column;
}
.title {
  font-size: 1rem;
}
.artist {
  color: var(--muted);
  font-size: 0.88rem;
}
</style>
