<script setup>
import { ref, computed, watch } from 'vue'

const allCards = ref([])
const currentQueue = ref([])
const nextPhaseQueue = ref([])

const isFlipped = ref(false)
const showTermFirst = ref(true)
const sessionFinished = ref(false)
const isShuffled = ref(false)
const cardKey = ref(0) 

// Modal & Search state
const isModalOpen = ref(false)
const searchQuery = ref('')

// Analytics state
const mistakesCounts = ref({})

const hasCards = computed(() => allCards.value.length > 0)

// Counters
const memorizedCount = computed(() => allCards.value.length - (currentQueue.value.length + nextPhaseQueue.value.length))
const notMemorizedCount = computed(() => currentQueue.value.length + nextPhaseQueue.value.length)

const currentCard = computed(() => currentQueue.value[0])

// Filtered cards for the modal
const filteredCards = computed(() => {
  const query = searchQuery.value.toLowerCase().trim()
  if (!query) return allCards.value
  return allCards.value.filter(card => 
    card.term.toLowerCase().includes(query) || 
    card.definition.toLowerCase().includes(query)
  )
})

// Analytics for struggled cards
const struggledCards = computed(() => {
  return allCards.value
    .filter(card => (mistakesCounts.value[card.id] || 0) >= 2)
    .sort((a, b) => mistakesCounts.value[b.id] - mistakesCounts.value[a.id])
})

const flip = () => {
  isFlipped.value = !isFlipped.value
}

const markMemorized = () => {
  advanceCard()
}

const markNotMemorized = () => {
  const currentId = currentQueue.value[0].id
  mistakesCounts.value[currentId] = (mistakesCounts.value[currentId] || 0) + 1
  
  // Push the exact card into the next phase bucket
  nextPhaseQueue.value.push(currentQueue.value[0])
  advanceCard()
}

const advanceCard = () => {
  isFlipped.value = false
  cardKey.value++
  
  currentQueue.value.shift() // Remove from current queue
  
  if (currentQueue.value.length === 0) {
    if (nextPhaseQueue.value.length > 0) {
      startNextPhase()
    } else {
      sessionFinished.value = true
    }
  }
}

const startNextPhase = () => {
  let nextCards = [...nextPhaseQueue.value]
  // Always shuffle for the next round
  for (let i = nextCards.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [nextCards[i], nextCards[j]] = [nextCards[j], nextCards[i]]
  }
  currentQueue.value = nextCards
  nextPhaseQueue.value = []
  // Auto-enable shuffle toggle since we are now permanently shuffled in future phases
  isShuffled.value = true 
}

watch(isShuffled, (shuffled) => {
  if (shuffled) {
    let temp = [...currentQueue.value]
    for (let i = temp.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [temp[i], temp[j]] = [temp[j], temp[i]]
    }
    currentQueue.value = temp
  } else {
    currentQueue.value.sort((a, b) => a.id - b.id)
  }
  cardKey.value++
  isFlipped.value = false
})

const restart = () => {
  let deck = [...allCards.value]
  if (isShuffled.value) {
    for (let i = deck.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [deck[i], deck[j]] = [deck[j], deck[i]]
    }
  }
  currentQueue.value = deck
  nextPhaseQueue.value = []
  sessionFinished.value = false
  isFlipped.value = false
  mistakesCounts.value = {}
  cardKey.value++
}

const handleFileUpload = (event) => {
  const file = event.target.files[0];
  if (!file) return;

  const reader = new FileReader();
  reader.onload = (e) => {
    try {
      const importedCards = JSON.parse(e.target.result);
      if (Array.isArray(importedCards) && importedCards.length > 0) {
        const formattedCards = importedCards.map((card, index) => ({
          id: card.id || index + Date.now(),
          term: card.term || 'No term provided',
          definition: card.definition || 'No definition provided'
        }));
        
        allCards.value = formattedCards;
        isShuffled.value = false;
        searchQuery.value = '';
        mistakesCounts.value = {};
        restart();
      } else {
        alert('Invalid JSON format or empty deck. Expected an array of flashcards.');
      }
    } catch (err) {
      alert('Failed to parse JSON file.');
    }
  };
  reader.readAsText(file);
  event.target.value = '';
};
</script>

<template>
  <div class="flashcard-app">
    
    <!-- Top Header & Actions -->
    <div class="header-actions">
      <h1>Flashcards</h1>
      <div class="header-buttons">
        <button class="view-all-btn" v-if="hasCards" @click="isModalOpen = true">View All Cards</button>
        <label class="upload-btn">
          Upload JSON Deck
          <input type="file" accept=".json" @change="handleFileUpload" hidden />
        </label>
      </div>
    </div>

    <!-- Empty State (No pre-loaded cards) -->
    <div v-if="!hasCards" class="empty-state">
      <h2>Welcome to Agentic Flashcards!</h2>
      <p>Your deck is empty. Please upload a JSON deck to begin your review session.</p>
    </div>

    <!-- Active Session Area -->
    <template v-else>
      <!-- Stats Bar -->
      <div class="stats-bar" v-if="!sessionFinished">
        <div class="stat memorized-stat">
          <span class="stat-value">{{ memorizedCount }}</span>
          <span class="stat-label">Memorized</span>
        </div>
        <div class="stat not-memorized-stat">
          <span class="stat-value">{{ notMemorizedCount }}</span>
          <span class="stat-label">Remaining</span>
        </div>
      </div>

      <!-- Controls (Toggles) -->
      <div v-if="!sessionFinished" class="controls">
        <label class="toggle-label">
          <input type="checkbox" v-model="showTermFirst" @change="isFlipped = false" />
          <span class="toggle-text">Term First</span>
        </label>
        
        <label class="toggle-label">
          <input type="checkbox" v-model="isShuffled" />
          <span class="toggle-text">Shuffle Deck</span>
        </label>
      </div>

      <!-- Card Area -->
      <div v-if="!sessionFinished && currentCard" class="card-container">
        <div class="card" :key="cardKey" @click="flip" :class="{ flipped: isFlipped }">
          <div class="card-inner">
            <div class="card-front">
              <h2>{{ showTermFirst ? currentCard.term : currentCard.definition }}</h2>
            </div>
            <div class="card-back">
              <h2>{{ showTermFirst ? currentCard.definition : currentCard.term }}</h2>
            </div>
          </div>
        </div>
        
        <div class="actions">
          <button class="not-memorized" @click="markNotMemorized">Not Memorized</button>
          <button class="memorized" @click="markMemorized">Memorized</button>
        </div>
      </div>

      <!-- Finished Screen -->
      <div v-else class="finished-screen">
        <h2>Review Complete! 🎉</h2>
        <p>You have successfully memorized all the cards.</p>
        
        <div v-if="struggledCards.length > 0" class="struggled-section">
          <h3>Terms you struggled with:</h3>
          <div class="struggled-grid">
            <div v-for="card in struggledCards" :key="card.id" class="struggled-card">
              <div class="struggled-term">{{ card.term }}</div>
              <div class="struggled-count">Missed {{ mistakesCounts[card.id] }} times</div>
            </div>
          </div>
        </div>

        <button class="restart-btn" @click="restart">Review Again</button>
      </div>
    </template>

    <!-- View All Modal -->
    <div v-if="isModalOpen" class="modal-overlay">
      <div class="modal-content">
        <div class="modal-header">
          <h2>All Flashcards ({{ allCards.length }})</h2>
          <input 
            type="text" 
            v-model="searchQuery" 
            class="search-input" 
            placeholder="Search terms or definitions..." 
          />
          <button class="close-btn" @click="isModalOpen = false">✕</button>
        </div>
        <div class="modal-body modal-grid">
          <div v-for="card in filteredCards" :key="card.id" class="list-card">
            <h3 class="list-term">{{ card.term }}</h3>
            <p class="list-def">{{ card.definition }}</p>
          </div>
          <div v-if="filteredCards.length === 0" class="no-results">
            No cards match your search query.
          </div>
        </div>
      </div>
    </div>

  </div>
</template>

<style scoped>
.flashcard-app {
  text-align: center;
  width: 100vw;
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
}

.header-actions {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
}

.header-actions h1 {
  margin: 0;
  color: #333;
}

.header-buttons {
  display: flex;
  gap: 10px;
}

.view-all-btn, .upload-btn {
  background-color: #3b82f6;
  color: white;
  padding: 8px 16px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 600;
  font-size: 0.9rem;
  transition: background-color 0.2s;
  box-shadow: 0 2px 4px rgba(59, 130, 246, 0.3);
}

.view-all-btn:hover, .upload-btn:hover {
  background-color: #2563eb;
}

.empty-state {
  padding: 60px 40px;
  background: white;
  border-radius: 16px;
  box-shadow: 0 10px 25px rgba(0,0,0,0.05);
  margin-top: 40px;
}

.empty-state h2 {
  color: #374151;
  margin-bottom: 16px;
}

.empty-state p {
  color: #6b7280;
}

/* Modal Styles */
.modal-overlay {
  position: fixed;
  top: 0; left: 0; right: 0; bottom: 0;
  background: rgba(0, 0, 0, 0.6);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  backdrop-filter: blur(4px); 
}

.modal-content {
  background: white;
  width: 95%;
  max-width: 1100px;
  max-height: 85vh;
  border-radius: 12px;
  display: flex;
  flex-direction: column;
  box-shadow: 0 20px 40px rgba(0,0,0,0.3);
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 24px;
  border-bottom: 1px solid #e5e7eb;
}

.modal-header h2 {
  margin: 0;
  font-size: 1.25rem;
  color: #1f2937;
  white-space: nowrap;
}

.search-input {
  flex: 1;
  margin: 0 20px;
  padding: 10px 16px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  font-size: 1rem;
  outline: none;
  background: #f9fafb;
  transition: border-color 0.2s, background 0.2s;
}

.search-input:focus {
  background: white;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.15);
}

.close-btn {
  background: none;
  border: none;
  font-size: 1.5rem;
  color: #6b7280;
  cursor: pointer;
  padding: 0;
  line-height: 1;
  border-radius: 4px;
}

.close-btn:hover {
  color: #ef4444;
}

.modal-body {
  padding: 24px;
  overflow-y: auto;
  text-align: left;
}

.modal-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 20px;
}

.list-card {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 20px;
  display: flex;
  flex-direction: column;
  transition: transform 0.1s, box-shadow 0.1s;
}

.list-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0,0,0,0.05);
}

.list-term {
  margin: 0 0 10px 0;
  font-size: 1.15rem;
  color: #0f172a;
}

.list-def {
  margin: 0;
  color: #475569;
  line-height: 1.5;
  flex-grow: 1;
}

.no-results {
  grid-column: span 3;
  text-align: center;
  color: #64748b;
  padding: 60px;
  font-size: 1.1rem;
}

/* Responsiveness for Modal Grid */
@media (max-width: 900px) {
  .modal-grid {
    grid-template-columns: repeat(2, 1fr);
  }
}
@media (max-width: 600px) {
  .modal-header {
    flex-direction: column;
    gap: 10px;
    align-items: stretch;
  }
  .search-input {
    margin: 0;
  }
  .modal-grid {
    grid-template-columns: 1fr;
  }
  .close-btn {
    position: absolute;
    top: 16px;
    right: 20px;
  }
}

/* Base App Styles */
.stats-bar {
  display: flex;
  justify-content: center;
  gap: 40px;
  margin-bottom: 24px;
  padding: 16px;
  background: white;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.05);
}

.stat {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.stat-value {
  font-size: 1.5rem;
  font-weight: bold;
}

.stat-label {
  font-size: 0.85rem;
  color: #6b7280;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-top: 4px;
}

.memorized-stat .stat-value {
  color: #10b981;
}

.not-memorized-stat .stat-value {
  color: #ef4444;
}

.controls {
  display: flex;
  justify-content: space-between;
  margin-bottom: 24px;
  align-items: center;
  background: white;
  padding: 12px 20px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
}

.toggle-label {
  display: flex;
  align-items: center;
  cursor: pointer;
  font-weight: 500;
  color: #555;
}

.toggle-label input {
  margin-right: 8px;
  width: 16px;
  height: 16px;
  cursor: pointer;
}

.restart-btn {
  background-color: #6366f1;
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 6px;
  cursor: pointer;
  font-weight: 600;
  transition: background-color 0.2s;
  margin-top: 10px;
}

.restart-btn:hover {
  background-color: #4f46e5;
}

.card-container {
  perspective: 1000px;
  margin-bottom: 30px;
}

.card {
  width: 100%;
  height: 350px;
  cursor: pointer;
  position: relative;
  transition: transform 0.6s;
  transform-style: preserve-3d;
}

.card.flipped .card-inner {
  transform: rotateY(180deg);
}

.card-inner {
  position: relative;
  width: 100%;
  height: 100%;
  text-align: center;
  transition: transform 0.6s;
  transform-style: preserve-3d;
  box-shadow: 0 10px 20px rgba(0,0,0,0.1);
  border-radius: 16px;
}

.card-front, .card-back {
  position: absolute;
  width: 100%;
  height: 100%;
  backface-visibility: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 30px;
  box-sizing: border-box;
  border-radius: 16px;
  background-color: white;
  font-size: 1.2rem;
  color: #1f2937;
}

.card-back {
  transform: rotateY(180deg);
  background-color: #f8fafc;
  border: 2px solid #e2e8f0;
  color: #334155;
}

.actions {
  margin-top: 30px;
  display: flex;
  justify-content: center;
  gap: 20px;
}

.actions button {
  flex: 1;
  padding: 14px 24px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 16px;
  font-weight: bold;
  transition: transform 0.1s, opacity 0.2s;
}

.actions button:active {
  transform: scale(0.98);
}

.memorized {
  background-color: #10b981;
  color: white;
  box-shadow: 0 4px 6px rgba(16, 185, 129, 0.2);
}

.memorized:hover {
  background-color: #059669;
}

.not-memorized {
  background-color: #ef4444;
  color: white;
  box-shadow: 0 4px 6px rgba(239, 68, 68, 0.2);
}

.not-memorized:hover {
  background-color: #dc2626;
}

.finished-screen {
  padding: 40px 40px;
  background: white;
  border-radius: 16px;
  box-shadow: 0 10px 25px rgba(0,0,0,0.05);
}

.finished-screen h2 {
  color: #10b981;
  margin-bottom: 16px;
}

.finished-screen p {
  color: #4b5563;
  margin-bottom: 10px;
}

/* Analytics Styles */
.struggled-section {
  margin: 24px 0;
  text-align: left;
  background: #fff1f2;
  border: 1px solid #fecdd3;
  border-radius: 12px;
  padding: 20px;
}

.struggled-section h3 {
  margin-top: 0;
  color: #be123c;
  font-size: 1.1rem;
  margin-bottom: 16px;
}

.struggled-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
  gap: 12px;
  max-height: 250px;
  overflow-y: auto;
}

.struggled-card {
  background: white;
  padding: 12px;
  border-radius: 8px;
  box-shadow: 0 1px 3px rgba(0,0,0,0.05);
  border: 1px solid #ffe4e6;
}

.struggled-term {
  font-weight: 600;
  color: #1f2937;
  margin-bottom: 4px;
}

.struggled-count {
  font-size: 0.85rem;
  color: #e11d48;
  font-weight: 500;
}
</style>
