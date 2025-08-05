<template>
  <div class="container">
    <h1>Kartu Pertanyaan Tertutup</h1>

    <!-- Langkah 1: Masukkan jumlah pemain -->
    <div v-if="step === 'input-count'">
      <label for="playerCount">Jumlah pemain:</label>
      <input
        v-model.number="playerCount"
        type="number"
        min="3"
        placeholder="Jumlah pemain"
      />
      <button @click="goToNameInput" :disabled="playerCount < 3">Lanjut</button>
    </div>

    <!-- Langkah 2: Masukkan nama pemain -->
    <div v-else-if="step === 'input-names'">
      <div v-for="(name, index) in playerNames" :key="index">
        <input
          v-model="playerNames[index]"
          :placeholder="'Nama pemain ' + (index + 1)"
        />
      </div>
      <button @click="goToModeratorInput" :disabled="!allNamesFilled">
        Lanjut
      </button>
    </div>

    <!-- Langkah 3: Moderator memasukkan pertanyaan dan jumlah impostor -->
    <div v-else-if="step === 'moderator-input'">
      <h2>Moderator: Setup Permainan</h2>

      <div style="margin-bottom: 1rem">
        <p><strong>Jumlah Impostor:</strong></p>
        <input
          v-model.number="impostorCount"
          type="number"
          :min="1"
          :max="maxImpostors"
          placeholder="Jumlah impostor"
          style="width: 200px"
        />
        <p style="font-size: 0.9em; color: #666">
          (Maksimal {{ maxImpostors }} dari {{ playerCount }} pemain)
        </p>
      </div>

      <p>
        <strong>Pertanyaan Mayoritas</strong> (akan diberikan kepada
        {{ playerCount - impostorCount }} pemain):
      </p>
      <textarea
        v-model="question1"
        placeholder="Contoh: Hewan apa yang akan kamu bawa saat hampir kiamat?"
        rows="3"
      ></textarea>

      <p>
        <strong>Pertanyaan Impostor</strong> (akan diberikan kepada
        {{ impostorCount }} pemain):
      </p>
      <textarea
        v-model="question2"
        placeholder="Contoh: Jika kamu menjadi suatu hewan, hewan apakah itu?"
        rows="3"
      ></textarea>

      <button @click="generateQuestions" :disabled="!gameSetupValid">
        Mulai Permainan
      </button>
    </div>

    <!-- Langkah 4: Mainkan (satu per satu) -->
    <div v-else-if="step === 'play'">
      <div v-if="currentPlayerIndex < playerCount">
        <p>
          <strong>{{ playerNames[currentPlayerIndex] }}</strong
          >, klik tombol di bawah untuk melihat pertanyaanmu (jangan dilihat
          oleh orang lain!)
        </p>
        <button @click="revealCard">Lihat Pertanyaan</button>

        <div v-if="cardRevealed" class="card revealed">
          {{ cards[currentPlayerIndex].question }}
        </div>

        <div v-if="cardRevealed">
          <button @click="nextPlayer">Lanjut ke Pemain Berikutnya</button>
        </div>
      </div>

      <!-- Semua pemain sudah melihat pertanyaan -->
      <div v-else>
        <h2>Selesai! Semua pemain sudah melihat pertanyaannya.</h2>
        <button
          @click="showResult = true"
          v-if="!showResult"
          style="margin-right: 5px"
        >
          Reveal Impostor{{ impostorIndices.length > 1 ? "s" : "" }}
        </button>

        <div v-if="showResult" class="reveal-box">
          <h3>Hasil Permainan:</h3>
          <ul>
            <li v-for="(card, index) in cards" :key="index">
              <strong>{{ playerNames[index] }}:</strong>
              "{{ card.question }}"
              <span v-if="impostorIndices.includes(index)" style="color: red"
                >👀 (Impostor)</span
              >
            </li>
          </ul>
        </div>

        <button @click="reset">Main Lagi</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from "vue";

const step = ref("input-count");
const playerCount = ref(3);
const playerNames = ref([]);
const cards = ref([]);
const currentPlayerIndex = ref(0);
const cardRevealed = ref(false);
const showResult = ref(false);
const impostorIndices = ref([]);
const impostorCount = ref(1);
const question1 = ref("");
const question2 = ref("");

const maxImpostors = computed(() => Math.floor(playerCount.value / 2));

// Load saved names from localStorage
function loadSavedNames() {
  const saved = localStorage.getItem("game-kartu-player-names");
  if (saved) {
    return JSON.parse(saved);
  }
  return [];
}

// Save names to localStorage
function saveNames() {
  const namesToSave = playerNames.value.filter((name) => name.trim() !== "");
  localStorage.setItem("game-kartu-player-names", JSON.stringify(namesToSave));
}

function goToNameInput() {
  const savedNames = loadSavedNames();
  playerNames.value = Array(playerCount.value).fill("");

  // Auto-fill with saved names
  for (let i = 0; i < Math.min(savedNames.length, playerCount.value); i++) {
    playerNames.value[i] = savedNames[i];
  }

  // Reset impostor count if it exceeds new max
  if (impostorCount.value > maxImpostors.value) {
    impostorCount.value = 1;
  }
  step.value = "input-names";
}

const allNamesFilled = computed(() =>
  playerNames.value.every((name) => name.trim() !== "")
);

function goToModeratorInput() {
  // Save names when proceeding
  saveNames();
  step.value = "moderator-input";
}

const gameSetupValid = computed(
  () =>
    question1.value.trim() !== "" &&
    question2.value.trim() !== "" &&
    impostorCount.value >= 1 &&
    impostorCount.value <= maxImpostors.value
);

function generateQuestions() {
  const q1 = question1.value.trim();
  const q2 = question2.value.trim();

  // Generate random impostor indices
  const indices = Array.from({ length: playerCount.value }, (_, i) => i);
  const selectedImpostors = [];

  for (let i = 0; i < impostorCount.value; i++) {
    const randomIndex = Math.floor(Math.random() * indices.length);
    selectedImpostors.push(indices[randomIndex]);
    indices.splice(randomIndex, 1);
  }

  impostorIndices.value = selectedImpostors;
  cards.value = [];

  for (let i = 0; i < playerCount.value; i++) {
    cards.value.push({
      question: selectedImpostors.includes(i) ? q2 : q1,
    });
  }

  step.value = "play";
  currentPlayerIndex.value = 0;
  cardRevealed.value = false;
  showResult.value = false;
}

function revealCard() {
  cardRevealed.value = true;
}

function nextPlayer() {
  currentPlayerIndex.value++;
  cardRevealed.value = false;
}

function reset() {
  step.value = "input-count";
  playerCount.value = 3;
  playerNames.value = [];
  cards.value = [];
  currentPlayerIndex.value = 0;
  cardRevealed.value = false;
  showResult.value = false;
  impostorIndices.value = [];
  impostorCount.value = 1;
  question1.value = "";
  question2.value = "";
}
</script>

<style scoped>
.container {
  text-align: center;
  padding: 2rem;
  max-width: 600px;
  margin: auto;
}
input,
textarea {
  margin: 5px;
  padding: 8px;
  width: 80%;
  border: 1px solid #ccc;
  border-radius: 5px;
}
textarea {
  resize: vertical;
  font-family: inherit;
}
button {
  margin-top: 1rem;
  padding: 10px 20px;
}
.card {
  margin-top: 1rem;
  padding: 1rem;
  border-radius: 10px;
  background-color: #f1f1f1;
  border: 2px solid #ccc;
}
.revealed {
  color: #222;
  font-weight: bold;
}
.reveal-box {
  margin-top: 1rem;
  text-align: left;
  padding: 1rem;
  border: 1px solid #ddd;
  background: #fafafa;
  border-radius: 10px;
  color: black;
}
.reveal-box li {
  margin-bottom: 0.5rem;
}
</style>
