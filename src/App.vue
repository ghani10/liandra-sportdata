<template>
  <div class="tournament-app dark-theme">
    <!-- Header -->
    <header class="header">
      <div class="header-content">
        <h1>
          <span class="pulse-dot"></span> 
          {{ tournamentName }}
        </h1>
        
        <div class="header-actions">
          <!-- Toggle Auto Sync -->
          <label class="auto-sync-toggle" title="Update otomatis setiap 10 detik">
            <span class="label-text">Auto-Sync</span>
            <div class="toggle-switch">
              <input type="checkbox" v-model="isAutoSync" @change="handleAutoSyncToggle">
              <span class="slider"></span>
            </div>
          </label>

          <!-- Tombol Refresh Manual -->
          <button @click="refreshData" class="action-btn refresh-btn" :disabled="isBackgroundLoading">
            <span class="icon" :class="{ 'spin': isBackgroundLoading }">🔄</span>
            <span class="btn-text">Refresh</span>
          </button>

          <!-- Tombol Kembali -->
          <button 
            v-if="currentView === 'bracket'" 
            @click="backToList" 
            class="action-btn back-btn"
          >
            <span class="icon">&larr;</span>
            <span class="btn-text">Daftar Kelas</span>
          </button>
        </div>
      </div>
    </header>

    <!-- State: Loading Global (Hanya muncul saat pertama kali load / pindah halaman) -->
    <div v-if="isLoading" class="loading-container">
      <div class="spinner"></div>
      <p>Mengambil data turnamen...</p>
    </div>

    <!-- VIEW 1: Menu Utama / List Kategori -->
    <main v-else-if="currentView === 'list'" class="menu-container fade-in">
      <div class="section-header">
        <div class="title-area">
          <h2 class="section-title">Daftar Kategori <span>Bagan (Draws)</span></h2>
          <span class="category-count" v-if="categoryList.length">{{ categoryList.length }} Kelas Pertandingan Tersedia</span>
        </div>
        
        <div class="sort-controls" v-if="categoryList.length">
          <label for="sortOption">Urutkan: </label>
          <div class="select-wrapper">
            <select id="sortOption" v-model="sortBy" class="sort-select">
              <option value="id_asc">Sesuai Nomor (ID)</option>
              <option value="name_asc">Nama Kelas (A - Z)</option>
              <option value="name_desc">Nama Kelas (Z - A)</option>
              <option value="type">Tipe (Perorangan / Beregu)</option>
            </select>
          </div>
        </div>
      </div>
      
      <div class="category-grid">
        <div 
          v-for="category in sortedCategoryList" 
          :key="category.catid" 
          class="category-card"
          @click="fetchBracketDetail(category.catid)"
        >
          <div class="card-glow"></div>
          <div class="card-icon">🥋</div>
          <div class="card-info">
            <h3>{{ category.name }}</h3>
            <div class="card-meta">
              <span class="meta-badge" :class="category.typeLabel === 'BEREGU' ? 'team' : 'individual'">
                {{ category.typeLabel || 'KATEGORI' }}
              </span>
              <span class="status-badge">Lihat Bagan &rarr;</span>
            </div>
          </div>
        </div>
      </div>
      
      <div v-if="categoryList.length === 0" class="empty-state">
        <div class="empty-icon">🔌</div>
        <p>Belum ada data kategori. Pastikan API lokal Sportdata Anda berjalan.</p>
      </div>
    </main>

    <!-- VIEW 2: Detail Bagan (Bracket) -->
    <main v-else-if="currentView === 'bracket'" class="bracket-container fade-in">
      <div class="bracket-header">
        <h2>{{ tournamentData?.CategoryID || 'Kategori' }}</h2>
      </div>

      <div class="bracket-tree-wrapper">
        <div class="bracket-scroll-area">
          
          <!-- Looping Kolom Putaran -->
          <div class="round-column" v-for="(round, rIdx) in bracketRounds" :key="round.round">
            <h3 class="round-title">Putaran {{ round.round }}</h3>
            
            <div class="matches-wrapper">
              <div class="match-node" v-for="(match, mIdx) in round.match" :key="mIdx">
                <!-- Match Card -->
                <div class="match-card" @click="openMatchDetails(match, round.round)">
                  <div class="competitor aka" :class="{ 'is-empty': match.id_red === 0, 'is-winner': match.winner === 'red' || match.winner === 1 }">
                    <span class="color-indicator red"></span>
                    <span class="name">{{ formatName(match.text_red, match.id_red) }}</span>
                    <span class="score" v-if="match.points_red !== null">{{ match.points_red }}</span>
                  </div>
                  <div class="divider"></div>
                  <div class="competitor ao" :class="{ 'is-empty': match.id_blue === 0, 'is-winner': match.winner === 'blue' || match.winner === 2 }">
                    <span class="color-indicator blue"></span>
                    <span class="name">{{ formatName(match.text_blue, match.id_blue) }}</span>
                    <span class="score" v-if="match.points_blue !== null">{{ match.points_blue }}</span>
                  </div>
                </div>
                <!-- Garis Relasi -->
                <div class="connector-line"></div>
              </div>
            </div>
          </div>

          <!-- KOLOM FINAL -->
          <div class="round-column final-column" v-if="bracketRounds && bracketRounds.length > 0">
            <h3 class="round-title champion-title">🏆 CHAMPION</h3>
            <div class="matches-wrapper">
              <div class="match-node final-node">
                <div class="match-card champion-card" @click="showChampionModal" :class="{'has-winner': tournamentChampion.color !== 'gray'}">
                  <div class="card-glow gold"></div>
                  <div class="competitor winner-slot" :class="tournamentChampion.color">
                    <span class="champion-icon" v-if="tournamentChampion.color === 'gray'">⏳</span>
                    <span class="champion-icon win" v-else>🥇</span>
                    <span class="name">{{ tournamentChampion.name }}</span>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <div v-if="!bracketRounds || bracketRounds.length === 0" class="empty-state">
            <div class="empty-icon">📋</div>
            <p>Data bagan belum tersedia atau belum di-drawing.</p>
          </div>
        </div>
      </div>
    </main>

    <!-- Modal Popup Detail Peserta -->
    <div class="modal-overlay" v-if="selectedMatch" @click.self="closeModal">
      <div class="modal-content scale-in">
        <div class="modal-header">
          <h3>Match Info <span>(R{{ selectedRound }})</span></h3>
          <button class="close-btn" @click="closeModal">&times;</button>
        </div>
        <div class="modal-body">
          <div class="fighter-detail red-side">
            <h4>AKA (Merah)</h4>
            <p class="fighter-name">{{ formatName(selectedMatch.text_red, selectedMatch.id_red, true) }}</p>
            <p class="fighter-id" v-if="selectedMatch.id_red !== 0">ID: {{ selectedMatch.id_red }}</p>
          </div>
          <div class="vs-badge">VS</div>
          <div class="fighter-detail blue-side">
            <h4>AO (Biru)</h4>
            <p class="fighter-name">{{ formatName(selectedMatch.text_blue, selectedMatch.id_blue, true) }}</p>
            <p class="fighter-id" v-if="selectedMatch.id_blue !== 0">ID: {{ selectedMatch.id_blue }}</p>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, onUnmounted } from 'vue';

const API_BASE = import.meta.env.VITE_API_BASE_URL;

// --- State Management ---
const currentView = ref('list'); 
const currentCatId = ref(null); // Menyimpan ID kategori yang sedang dibuka
const isLoading = ref(false);
const isBackgroundLoading = ref(false); // Untuk indikator refresh tanpa block layar
const tournamentName = ref('Sportdata Live Event');
const sortBy = ref('id_asc'); 

const categoryList = ref([]);
const tournamentData = ref(null);
const bracketRounds = ref([]);
const selectedMatch = ref(null);
const selectedRound = ref(null);

// --- State Auto-Sync ---
const isAutoSync = ref(true);
let syncInterval = null;

// --- Computed Properties ---
const sortedCategoryList = computed(() => {
  let list = [...categoryList.value]; 
  switch (sortBy.value) {
    case 'name_asc': return list.sort((a, b) => a.name.localeCompare(b.name));
    case 'name_desc': return list.sort((a, b) => b.name.localeCompare(a.name));
    case 'id_asc': return list.sort((a, b) => a.catid - b.catid);
    case 'type': return list.sort((a, b) => {
        const typeA = a.typeLabel || '';
        const typeB = b.typeLabel || '';
        if (typeA === typeB) return a.name.localeCompare(b.name);
        return typeA.localeCompare(typeB);
      });
    default: return list;
  }
});

const tournamentChampion = computed(() => {
  if (!bracketRounds.value || bracketRounds.value.length === 0) return { name: "TBD (Belum Ada Juara)", color: 'gray' };
  const lastRound = bracketRounds.value[bracketRounds.value.length - 1];
  if (!lastRound || !lastRound.match || lastRound.match.length === 0) return { name: "TBD (Belum Ada Juara)", color: 'gray' };
  const finalMatch = lastRound.match[0];
  if (!finalMatch) return { name: "TBD (Belum Ada Juara)", color: 'gray' };

  if (finalMatch.winner === "red" || finalMatch.winner === 1) return { name: formatName(finalMatch.text_red, finalMatch.id_red), color: 'red' };
  if (finalMatch.winner === "blue" || finalMatch.winner === 2) return { name: formatName(finalMatch.text_blue, finalMatch.id_blue), color: 'blue' };
  return { name: "TBD (Menunggu Final)", color: 'gray' };
});

// --- Fetch Functions ---
const fetchCategoryList = async (isBackground = false) => {
  if (!isBackground) isLoading.value = true;
  else isBackgroundLoading.value = true;

  try {
    const response = await fetch(`${API_BASE}/api/draw/categories`);
    const result = await response.json();
    if (response.ok && result.success) categoryList.value = result.data;
  } catch (error) {
    console.warn('API fetch failed.', error);
  } finally {
    if (!isBackground) isLoading.value = false;
    else isBackgroundLoading.value = false;
  }
};

const fetchBracketDetail = async (catid, isBackground = false) => {
  if (!isBackground) isLoading.value = true;
  else isBackgroundLoading.value = true;

  currentCatId.value = catid;

  try {
    const response = await fetch(`${API_BASE}/api/draw/${catid}`);
    const result = await response.json();
    
    if (response.ok && result.success) {
      const rawDrawData = result.data.Draws;
      tournamentData.value = { event: rawDrawData.event, CategoryID: rawDrawData.CategoryID };
      
      // Normalisasi Data (Array vs Object issue dari API)
      let normalizedRounds = [];
      let roundsRaw = rawDrawData.Draw?.round;
      if (roundsRaw) {
        const roundsArray = Array.isArray(roundsRaw) ? roundsRaw : [roundsRaw];
        normalizedRounds = roundsArray.map(r => ({
          ...r,
          match: Array.isArray(r.match) ? r.match : (r.match ? [r.match] : [])
        }));
      }

      bracketRounds.value = normalizedRounds;
      tournamentName.value = rawDrawData.event;
      currentView.value = 'bracket';
    } else {
      if(!isBackground) alert(result.message || 'Gagal memuat detail bagan.');
    }
  } catch (error) {
    console.error('Error fetching bracket:', error);
  } finally {
    if (!isBackground) isLoading.value = false;
    else isBackgroundLoading.value = false;
  }
};

// --- Refresh Logic & Auto-Sync ---
const refreshData = async () => {
  if (currentView.value === 'list') {
    await fetchCategoryList(true); // true = refresh di background
  } else if (currentView.value === 'bracket' && currentCatId.value) {
    await fetchBracketDetail(currentCatId.value, true);
  }
};

const handleAutoSyncToggle = () => {
  if (isAutoSync.value) {
    // Start interval tiap 10 detik
    syncInterval = setInterval(() => {
      refreshData();
    }, 10000);
  } else {
    // Stop interval
    if (syncInterval) clearInterval(syncInterval);
  }
};

// --- Utilities ---
const backToList = () => {
  currentView.value = 'list';
  currentCatId.value = null;
  bracketRounds.value = [];
};

const formatName = (text, id, fullName = false) => {
  if (id === 0 || !text) return "BYE / TBD";
  if (fullName) return text.replace(/_/g, ' ');
  const parts = text.split('-');
  let name = parts[0].replace(/_/g, ' ').trim();
  if (name.length > 20) name = name.substring(0, 18) + '...';
  return name;
};

const openMatchDetails = (match, roundNum) => {
  if (match.id_red === 0 && match.id_blue === 0) return; 
  selectedMatch.value = match;
  selectedRound.value = roundNum;
};

const showChampionModal = () => {
  if (tournamentChampion.value.color === 'gray') return; 
  alert(`🏆 SELAMAT!\nJuara 1: ${tournamentChampion.value.name}`);
};

const closeModal = () => {
  selectedMatch.value = null;
  selectedRound.value = null;
};

// --- Lifecycle ---
onMounted(() => {
  fetchCategoryList();
  handleAutoSyncToggle(); // Mulai auto-sync saat aplikasi dibuka
});

onUnmounted(() => {
  if (syncInterval) clearInterval(syncInterval);
});
</script>

<style scoped>
/* ==========================================================================
   CSS VARIABLES & DESIGN SYSTEM (DARK THEME)
   ========================================================================== */
:root {
  --bg-main: #0B1121;
  --bg-card: #151e32;
  --bg-card-hover: #1e293b;
  --text-main: #f8fafc;
  --text-muted: #94a3b8;
  --accent: #3b82f6;
  --accent-glow: rgba(59, 130, 246, 0.4);
  --border-color: #334155;
  --aka-red: #ef4444;
  --ao-blue: #3b82f6;
  --champion-gold: #fbbf24;
}

/* ==========================================================================
   GLOBAL & ANIMATIONS
   ========================================================================== */
.dark-theme { font-family: 'Inter', system-ui, -apple-system, sans-serif; background-color: #0B1121; background: radial-gradient(circle at top, #111827 0%, #0B1121 100%); min-height: 100vh; color: #f8fafc; }
::-webkit-scrollbar { width: 8px; height: 8px; }
::-webkit-scrollbar-track { background: #0B1121; }
::-webkit-scrollbar-thumb { background: #334155; border-radius: 4px; }
::-webkit-scrollbar-thumb:hover { background: #475569; }

.fade-in { animation: fadeIn 0.4s ease-out; }
.scale-in { animation: scaleIn 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275); }

@keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
@keyframes scaleIn { from { opacity: 0; transform: scale(0.9); } to { opacity: 1; transform: scale(1); } }
@keyframes spin { to { transform: rotate(360deg); } }
@keyframes pulse { 0% { box-shadow: 0 0 0 0 rgba(16, 185, 129, 0.7); } 70% { box-shadow: 0 0 0 10px rgba(16, 185, 129, 0); } 100% { box-shadow: 0 0 0 0 rgba(16, 185, 129, 0); } }

.icon.spin { display: inline-block; animation: spin 1s linear infinite; }

/* ==========================================================================
   HEADER & ACTIONS
   ========================================================================== */
.header { background: rgba(15, 23, 42, 0.7); backdrop-filter: blur(12px); -webkit-backdrop-filter: blur(12px); border-bottom: 1px solid #334155; padding: 1rem 2rem; position: sticky; top: 0; z-index: 100; }
.header-content { display: flex; justify-content: space-between; align-items: center; max-width: 1400px; margin: auto; flex-wrap: wrap; gap: 1rem;}
.header-content h1 { margin: 0; font-size: 1.4rem; font-weight: 700; display: flex; align-items: center; gap: 0.75rem; letter-spacing: -0.5px; }

.pulse-dot { width: 10px; height: 10px; background-color: #10b981; border-radius: 50%; display: inline-block; animation: pulse 2s infinite; }

.header-actions { display: flex; align-items: center; gap: 1rem; }

/* Tombol Global */
.action-btn { background: transparent; color: #94a3b8; border: 1px solid #334155; padding: 0.5rem 1rem; border-radius: 8px; cursor: pointer; font-weight: 600; transition: all 0.3s ease; display: flex; align-items: center; gap: 0.5rem; }
.action-btn:hover:not(:disabled) { background: #1e293b; color: #f8fafc; border-color: #475569; }
.action-btn:disabled { opacity: 0.5; cursor: not-allowed; }

/* Toggle Auto Sync */
.auto-sync-toggle { display: flex; align-items: center; gap: 0.5rem; cursor: pointer; color: #94a3b8; font-size: 0.9rem; font-weight: 600;}
.toggle-switch { position: relative; width: 40px; height: 20px; }
.toggle-switch input { opacity: 0; width: 0; height: 0; }
.slider { position: absolute; cursor: pointer; top: 0; left: 0; right: 0; bottom: 0; background-color: #334155; transition: .4s; border-radius: 20px; }
.slider:before { position: absolute; content: ""; height: 14px; width: 14px; left: 3px; bottom: 3px; background-color: white; transition: .4s; border-radius: 50%; }
input:checked + .slider { background-color: #3b82f6; }
input:checked + .slider:before { transform: translateX(20px); }


/* ==========================================================================
   LOADING & EMPTY STATES
   ========================================================================== */
.loading-container { display: flex; flex-direction: column; align-items: center; justify-content: center; height: 70vh; color: #94a3b8; }
.spinner { width: 40px; height: 40px; border: 3px solid #334155; border-top-color: #3b82f6; border-radius: 50%; animation: spin 1s linear infinite; margin-bottom: 1rem; }
.empty-state { text-align: center; padding: 4rem 2rem; color: #64748b; background: rgba(30, 41, 59, 0.3); border-radius: 12px; border: 1px dashed #334155; margin-top: 2rem; }
.empty-icon { font-size: 3rem; margin-bottom: 1rem; opacity: 0.5; }

/* ==========================================================================
   VIEW 1: MENU LIST KATEGORI
   ========================================================================== */
.menu-container { max-width: 1200px; margin: 2rem auto; padding: 0 1.5rem; }
.section-header { display: flex; justify-content: space-between; align-items: flex-end; margin-bottom: 2rem; border-bottom: 1px solid #334155; padding-bottom: 1rem; }
.title-area { display: flex; flex-direction: column; }
.section-title { font-size: 1.5rem; color: #f8fafc; margin: 0 0 0.25rem 0; font-weight: 700; }
.section-title span { color: #3b82f6; }
.category-count { color: #94a3b8; font-size: 0.9rem; }
.sort-controls { display: flex; align-items: center; gap: 0.75rem; font-size: 0.9rem; color: #cbd5e1; }
.select-wrapper { position: relative; }
.sort-select { appearance: none; background-color: #151e32; color: #f8fafc; border: 1px solid #334155; padding: 0.5rem 2rem 0.5rem 1rem; border-radius: 8px; font-family: inherit; font-weight: 500; cursor: pointer; outline: none; transition: border-color 0.2s; }
.sort-select:focus, .sort-select:hover { border-color: #3b82f6; }
.category-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(300px, 1fr)); gap: 1.5rem; }
.category-card { background: #151e32; border: 1px solid #334155; border-radius: 12px; padding: 1.5rem; display: flex; align-items: center; gap: 1.25rem; cursor: pointer; position: relative; overflow: hidden; transition: all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1); }
.category-card::before { content: ''; position: absolute; top: 0; left: 0; width: 4px; height: 100%; background: #334155; transition: background 0.3s; }
.category-card:hover { transform: translateY(-5px); border-color: #475569; background: #1e293b; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.5); }
.category-card:hover::before { background: #3b82f6; }
.card-icon { font-size: 1.8rem; background: rgba(59, 130, 246, 0.1); padding: 0.75rem; border-radius: 10px; border: 1px solid rgba(59, 130, 246, 0.2); }
.card-info { flex: 1; }
.card-info h3 { margin: 0 0 0.5rem 0; font-size: 1rem; color: #f8fafc; line-height: 1.4; font-weight: 600; }
.card-meta { display: flex; justify-content: space-between; align-items: center; }
.meta-badge { font-size: 0.7rem; padding: 0.25rem 0.6rem; border-radius: 6px; font-weight: 700; letter-spacing: 0.5px; }
.meta-badge.individual { background: rgba(148, 163, 184, 0.1); color: #cbd5e1; border: 1px solid rgba(148, 163, 184, 0.2); }
.meta-badge.team { background: rgba(245, 158, 11, 0.1); color: #fbbf24; border: 1px solid rgba(245, 158, 11, 0.2); }
.status-badge { font-size: 0.8rem; font-weight: 600; color: #3b82f6; opacity: 0; transition: opacity 0.3s; transform: translateX(-10px); }
.category-card:hover .status-badge { opacity: 1; transform: translateX(0); }

/* ==========================================================================
   VIEW 2: DETAIL BAGAN (BRACKET TREE)
   ========================================================================== */
.bracket-container { max-width: 100%; padding: 2rem 1.5rem; overflow: hidden; }
.bracket-header { margin-bottom: 3rem; text-align: center; }
.bracket-header h2 { display: inline-block; background: rgba(59, 130, 246, 0.1); color: #60a5fa; padding: 0.5rem 2rem; border-radius: 999px; font-size: 1.1rem; border: 1px solid rgba(59, 130, 246, 0.3); }
.bracket-tree-wrapper { background: #151e32; border-radius: 16px; padding: 3rem 2rem 2rem 2rem; border: 1px solid #334155; box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.5), inset 0 1px 0 rgba(255,255,255,0.05); }
.bracket-scroll-area { display: flex; overflow-x: auto; padding-bottom: 1rem; gap: 4rem; align-items: stretch; cursor: grab; }
.bracket-scroll-area:active { cursor: grabbing; }
.round-column { display: flex; flex-direction: column; min-width: 260px; justify-content: space-around; position: relative; }
.round-title { text-align: center; font-size: 0.9rem; color: #94a3b8; font-weight: 600; text-transform: uppercase; letter-spacing: 1px; position: absolute; top: -50px; left: 0; width: 100%; }
.champion-title { color: #fbbf24; text-shadow: 0 0 10px rgba(251, 191, 36, 0.3); }
.matches-wrapper { display: flex; flex-direction: column; justify-content: space-around; flex-grow: 1; }
.match-node { position: relative; padding: 12px 0; display: flex; align-items: center; }
.round-column:not(:first-child) .match-node::before { content: ''; position: absolute; left: -2rem; top: 50%; width: 2rem; border-top: 2px solid #475569; z-index: 0; }
.round-column:not(:last-child) .match-node::after { content: ''; position: absolute; right: -2rem; top: 50%; width: 2rem; border-top: 2px solid #475569; z-index: 0; }
.match-card { background: #1e293b; border-radius: 8px; border: 1px solid #334155; cursor: pointer; transition: all 0.2s ease; width: 100%; position: relative; z-index: 1; box-shadow: 0 4px 6px -1px rgba(0,0,0,0.3); }
.match-card:hover { border-color: #60a5fa; box-shadow: 0 0 15px rgba(59, 130, 246, 0.2); transform: scale(1.02); }
.competitor { display: flex; align-items: center; padding: 0.7rem 0.8rem; font-size: 0.85rem; font-weight: 500; }
.competitor.is-empty { color: #64748b; font-style: italic; background: rgba(15, 23, 42, 0.5); }
.competitor.is-winner { font-weight: 800; background: rgba(255, 255, 255, 0.05); color: #fff; }
.divider { height: 1px; background-color: #334155; }
.color-indicator { width: 12px; height: 12px; border-radius: 50%; margin-right: 10px; flex-shrink: 0; box-shadow: 0 0 8px currentColor; }
.color-indicator.red { background-color: #ef4444; color: rgba(239, 68, 68, 0.6); }
.color-indicator.blue { background-color: #3b82f6; color: rgba(59, 130, 246, 0.6); }
.name { flex-grow: 1; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.score { font-weight: 800; font-size: 1rem; font-family: monospace; padding-left: 10px; }
.champion-card { background: #1e293b; border-color: #475569; }
.champion-card.has-winner { border: 1px solid #fbbf24; background: linear-gradient(145deg, #1e293b, #422006); box-shadow: 0 0 20px rgba(251, 191, 36, 0.2); }
.winner-slot { justify-content: center; padding: 1.5rem 1rem; font-size: 1rem; color: #94a3b8; }
.champion-card.has-winner .winner-slot { color: #fef3c7; font-weight: 800; text-shadow: 0 2px 4px rgba(0,0,0,0.5); }
.champion-icon { margin-right: 10px; font-size: 1.2rem; }
.champion-icon.win { font-size: 1.5rem; filter: drop-shadow(0 0 5px rgba(251, 191, 36, 0.8)); }

/* ==========================================================================
   MODAL POPUP
   ========================================================================== */
.modal-overlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(11, 17, 33, 0.85); display: flex; justify-content: center; align-items: center; z-index: 1000; backdrop-filter: blur(5px); }
.modal-content { background: #151e32; border-radius: 16px; width: 90%; max-width: 450px; border: 1px solid #334155; box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.5); overflow: hidden; }
.modal-header { background: #1e293b; padding: 1.25rem 1.5rem; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #334155; }
.modal-header h3 { margin: 0; font-size: 1.2rem; font-weight: 600; color: #f8fafc; }
.modal-header h3 span { color: #64748b; font-size: 0.9rem; }
.close-btn { background: none; border: none; font-size: 1.8rem; cursor: pointer; color: #94a3b8; line-height: 1; transition: color 0.2s; }
.close-btn:hover { color: #ef4444; }
.modal-body { padding: 2rem 1.5rem; display: flex; flex-direction: column; gap: 1rem; }
.fighter-detail { padding: 1.25rem; border-radius: 12px; text-align: center; border: 1px solid; }
.red-side { background: rgba(239, 68, 68, 0.05); border-color: rgba(239, 68, 68, 0.3); }
.blue-side { background: rgba(59, 130, 246, 0.05); border-color: rgba(59, 130, 246, 0.3); }
.fighter-detail h4 { margin: 0 0 0.5rem 0; font-size: 0.8rem; text-transform: uppercase; letter-spacing: 1px; color: #94a3b8; }
.red-side h4 { color: #f87171; } .blue-side h4 { color: #60a5fa; }
.fighter-name { font-size: 1.1rem; font-weight: 700; color: #f8fafc; margin: 0; line-height: 1.4; }
.fighter-id { font-size: 0.8rem; color: #64748b; margin-top: 0.5rem; font-family: monospace; }
.vs-badge { align-self: center; background: #0B1121; color: #94a3b8; padding: 0.5rem 1.2rem; border-radius: 999px; font-weight: 900; font-size: 0.85rem; border: 1px solid #334155; box-shadow: 0 0 15px rgba(0,0,0,0.5); z-index: 2; margin: -1rem 0; }

/* ==========================================================================
   MEDIA QUERIES (RESPONSIVE)
   ========================================================================== */
@media (max-width: 768px) {
  .section-header { flex-direction: column; align-items: flex-start; gap: 1.5rem; }
  .sort-controls { width: 100%; justify-content: space-between; }
  .select-wrapper { flex: 1; margin-left: 1rem; }
  .sort-select { width: 100%; }
  
  .header-actions { width: 100%; justify-content: space-between; margin-top: 0.5rem; }
  .btn-text { display: none; }
  
  .bracket-container { padding: 1rem; }
  .bracket-tree-wrapper { padding: 2rem 1rem 1rem 1rem; }
  .round-column { min-width: 220px; }
}
</style>