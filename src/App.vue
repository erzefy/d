<script setup>
import { ref, onMounted, computed } from 'vue'
import { io } from 'socket.io-client'

const socket = io('http://localhost:3000')

const gameState = ref(null)
const selectedCard = ref(null)
const lobbies = ref([])
const currentLobby = ref(null)
const playerName = ref('')
const status = ref('')
const showLobbyForm = ref(true)
const isDefending = computed(() => 
  gameState.value?.nextDefender === socket.id
)
const canOnlyTakeCards = computed(() => {
  if (!gameState.value || gameState.value.nextDefender !== socket.id) return false;
  // Если хотя бы одной картой можно отбиться, возвращаем false
  return !gameState.value.players.self.cards.some(canDefendWith);
})

const createLobby = () => {
  if (!playerName.value) {
    status.value = 'Введите имя игрока';
    return;
  }
  socket.emit('createLobby', { 
    name: `Лобби ${playerName.value}`,
    playerName: playerName.value
  });
  showLobbyForm.value = false;
}

const joinLobby = (lobby) => {
  if (!playerName.value) {
    status.value = 'Введите имя игрока';
    return;
  }
  socket.emit('joinLobby', { 
    lobbyId: lobby.id,
    playerName: playerName.value
  });
  showLobbyForm.value = false;
}

const toggleReady = () => {
  socket.emit('toggleReady');
}

const startGame = () => {
  if (currentLobby.value && currentLobby.value.hostId === socket.id) {
    socket.emit('startGame', { lobbyId: currentLobby.value.id });
  }
}

const endTurn = () => {
  if (gameState.value?.currentTurn === socket.id) {
    socket.emit('endTurn', {
      lobbyId: gameState.value.lobbyId
    });
  }
}

const passTurn = () => {
  if (gameState.value?.currentTurn === socket.id) {
    socket.emit('passTurn', {
      lobbyId: gameState.value.lobbyId
    });
  }
}

const takingCards = ref(false);

const takeCards = () => {
  if (
    (gameState.value?.canTakeCards && gameState.value?.nextDefender === socket.id) ||
    (canOnlyTakeCards.value && gameState.value?.nextDefender === socket.id)
  ) {
    takingCards.value = true;
    socket.emit('takeCards', {
      lobbyId: gameState.value.lobbyId
    });
  }
}

// Сброс анимации после обновления состояния
onMounted(() => {
  socket.emit('getLobbies');

  socket.on('lobbiesList', (data) => {
    lobbies.value = data;
  });

  socket.on('lobbyCreated', ({ lobby }) => {
    currentLobby.value = lobby;
    status.value = `Лобби создано: ${lobby.id}`;
  });

  socket.on('playerJoined', ({ lobby }) => {
    currentLobby.value = lobby;
    status.value = 'Игрок присоединился к лобби';
  });

  socket.on('playerLeft', ({ lobby, playerId }) => {
    currentLobby.value = lobby;
    status.value = 'Игрок покинул лобби';
  });

  socket.on('playerStatusUpdated', ({ lobby }) => {
    currentLobby.value = lobby;
  });

  socket.on('lobbiesUpdated', (data) => {
    lobbies.value = data;
  });

  socket.on('gameStarted', (state) => {
    gameState.value = state;
    status.value = 'Игра началась!';
  });

  socket.on('gameStateUpdated', (state) => {
    gameState.value = state;
    takingCards.value = false;
    updateGameStatus();
  });
  socket.on('gameEnded', ({ winner }) => {
    if (winner === socket.id) {
      status.value = 'Поздравляем! Вы победили!';
    } else if (gameState.value?.players.others.find(p => p.id === winner)) {
      status.value = `Игрок ${gameState.value.players.others.find(p => p.id === winner).name} победил!`;
    } else {
      status.value = 'Игра окончена!';
    }
    gameState.value = null;
    currentLobby.value = null;
    showLobbyForm.value = true;
  });

  socket.on('connect', () => {
    status.value = 'Подключено к серверу';
  });

  socket.on('disconnect', () => {
    status.value = 'Отключено от сервера';
    gameState.value = null;
    currentLobby.value = null;
    showLobbyForm.value = true;
  });
});

const updateGameStatus = () => {
  if (!gameState.value) return;
  
  if (gameState.value.status === 'taking_cards') {
    status.value = gameState.value.nextDefender === socket.id 
      ? 'Вы должны взять карты' 
      : 'Игрок берет карты';
  } else if (gameState.value.currentTurn === socket.id) {
    status.value = isDefending.value ? 'Ваш ход: защищайтесь' : 'Ваш ход: атакуйте';
  } else {
    status.value = isDefending.value ? 'Ожидание атаки' : 'Ход игрока';
  }
}

const getSuitSymbol = (suit) => {
  const symbols = {
    'hearts': '♥',
    'diamonds': '♦',
    'clubs': '♣',
    'spades': '♠'
  };
  return symbols[suit] || suit;
}

const canAttackWith = (card) => {
  if (!gameState.value || gameState.value.currentTurn !== socket.id) return false;

  // Первый ход или новая атака (стол пустой) - можно ходить любой картой
  if (gameState.value.table.attacking.length === 0) {
    return true;
  }

  // Нельзя подкидывать больше 6 карт
  if (gameState.value.table.attacking.length >= 6) return false;

  // Нельзя подкидывать больше, чем карт у защищающегося
  const defender = gameState.value.players.others.find(
    p => p.id === gameState.value.nextDefender
  );
  if (defender && defender.cardCount <= gameState.value.table.attacking.length) {
    return false;
  }

  // Подкидывать можно только карты тех же достоинств, что уже есть на столе
  const validRanks = new Set([
    ...gameState.value.table.attacking.map(c => c.rank),
    ...gameState.value.table.defending.filter(c => c !== null).map(c => c.rank)
  ]);

  // Только если ход не завершён (все карты не отбиты)
  if (gameState.value.table.defending.length === gameState.value.table.attacking.length) return false;

  return validRanks.has(card.rank);
}

const canDefendWith = (card) => {
  if (!gameState.value) {
    console.log('No game state');
    return false;
  }
  
  if (gameState.value.nextDefender !== socket.id) {
    console.log('Not a defender', gameState.value.nextDefender, socket.id);
    return false;
  }

  const lastAttackingCard = gameState.value.table.attacking[
    gameState.value.table.defending.length
  ];
  
  if (!lastAttackingCard) {
    console.log('No attacking card to defend against');
    return false;
  }

  console.log('Defending card:', card);
  console.log('Against card:', lastAttackingCard);
  console.log('Trump suit:', gameState.value.trump.suit);

  // Если атакующая карта козырная
  if (lastAttackingCard.suit === gameState.value.trump.suit) {
    // Можно бить только старшим козырем
    const canDefend = card.suit === gameState.value.trump.suit && card.value > lastAttackingCard.value;
    console.log('Defending against trump with:', card.suit, card.value, canDefend);
    return canDefend;
  }

  // Если карта защиты козырная, а атакующая нет - можно бить
  if (card.suit === gameState.value.trump.suit && lastAttackingCard.suit !== gameState.value.trump.suit) {
    console.log('Defending with trump against non-trump');
    return true;
  }

  // В остальных случаях можно бить только старшей картой той же масти
  const canDefend = card.suit === lastAttackingCard.suit && card.value > lastAttackingCard.value;
  console.log('Regular defense:', card.suit, card.value, 'vs', lastAttackingCard.suit, lastAttackingCard.value, canDefend);
  return canDefend;
}

const playCard = (card) => {
  if (!gameState.value || !gameState.value.lobbyId) return;

  const isDefendingNow = gameState.value.nextDefender === socket.id;
  console.log('Is defending:', isDefendingNow); // Debug log
  
  // Проверяем, может ли игрок ходить
  if (isDefendingNow) {
    // Для защиты:
    if (!canDefendWith(card)) {
      status.value = 'Этой картой нельзя отбиться!';
      console.log('Cannot defend with card:', card); // Debug log
      return;
    }
  } else {
    // Для атаки:
    if (gameState.value.currentTurn !== socket.id) {
      status.value = 'Сейчас не ваш ход!';
      return;
    }
    if (gameState.value.players.self.hasPickedUpCards) {
      status.value = 'Вы должны пропустить ход после взятия карт!';
      return;
    }
    if (!canAttackWith(card)) {
      status.value = 'Этой картой сейчас нельзя ходить!';
      return;
    }
  }

  socket.emit('playCard', {
    lobbyId: gameState.value.lobbyId,
    card,
    isDefending: isDefendingNow
  });
  selectedCard.value = card;
}
</script>

<template>
  <div class="game-container">
    <!-- Форма входа -->
    <div v-if="showLobbyForm" class="lobby-form">
      <div class="input-group">
        <input v-model="playerName" 
               placeholder="Введите ваше имя"
               @keyup.enter="createLobby" />
      </div>
      
      <!-- Список доступных лобби -->
      <div class="lobbies-list" v-if="lobbies.length > 0">
        <h3>Доступные лобби:</h3>
        <div v-for="lobby in lobbies" :key="lobby.id" class="lobby-item">
          <span>{{ lobby.name }} ({{ lobby.players.length }}/{{ lobby.maxPlayers }})</span>
          <button @click="joinLobby(lobby)" 
                  :disabled="!playerName">
            Присоединиться
          </button>
        </div>
      </div>
      
      <button @click="createLobby" 
              :disabled="!playerName" 
              class="create-lobby-btn">
        Создать новое лобби
      </button>
    </div>

    <!-- Лобби -->
    <div v-else-if="currentLobby && !gameState" class="lobby-view">
      <h2>{{ currentLobby.name }}</h2>
      <div class="players-list">
        <div v-for="player in currentLobby.players" 
             :key="player.id" 
             class="player-item"
             :class="{ host: player.id === currentLobby.hostId }">
          <span>{{ player.name }}</span>
          <span v-if="player.id === currentLobby.hostId">(Хост)</span>
          <span v-if="player.isReady" class="ready-status">Готов</span>
        </div>
      </div>
      
      <div class="lobby-controls">
        <button v-if="currentLobby.hostId === socket.id"
                @click="startGame"
                :disabled="currentLobby.players.length < 2">
          Начать игру
        </button>
        <button v-else
                @click="toggleReady">
          {{ currentLobby.players.find(p => p.id === socket.id)?.isReady ? 'Не готов' : 'Готов' }}
        </button>
      </div>
    </div>

    <!-- Игровой стол -->



    <div v-else-if="gameState" class="game-board">      <!-- Статус игры -->
      <div class="game-info">
        <div class="status">{{ status }}</div>
        <div class="trump-info" v-if="gameState.trump">
          Козырь: <span :class="gameState.trump.suit">{{ getSuitSymbol(gameState.trump.suit) }}</span>
        </div>
        <div class="turn-info">
          <template v-if="gameState.status === 'playing'">
            <span v-if="gameState.currentTurn === socket.id">
              {{ isDefending ? 'Ваш ход: Отбейтесь от карт' : 'Ваш ход: Атакуйте' }}
            </span>
            <span v-else>
              {{ isDefending ? 'Ожидайте атаки' : 'Ход игрока: ' + (gameState.players.others.find(p => p.id === gameState.currentTurn)?.name || 'Противник') }}
            </span>
          </template>
          <span v-else-if="gameState.status === 'taking_cards'">
            {{ gameState.nextDefender === socket.id ? 'Вы берете карты' : 'Игрок берет карты' }}
          </span>
        </div>
      </div>

      <!-- Карты других игроков -->
      <div class="other-players">
        <div v-for="player in gameState.players.others" 
             :key="player.id"
             class="opponent"
             :class="{ current: player.id === gameState.currentTurn }">
          <div class="player-name">{{ player.name }}</div>
          <div class="opponent-hand">
            <div v-for="i in player.cardCount" 
                 :key="i" 
                 class="card card-back" />
          </div>
        </div>
      </div>

      <!-- Игровой стол -->
      <div class="table">
        <div class="deck-area">
          <div v-if="gameState.deck.length" class="card card-back deck">
            {{ gameState.deck.length }}
          </div>
          <div class="trump-card" v-if="gameState.trump">
            <div class="card" :class="gameState.trump.suit">
              {{ gameState.trump.rank }}
            </div>
          </div>
        </div>
        <div class="playing-field">
          <div v-for="(attack, index) in gameState.table.attacking" 
               :key="'attack' + index" 
               class="card-pair">
            <div class="card" :class="[attack.suit, takingCards ? 'taking' : '']">
              <div class="card-content">
                <div class="card-top">{{ attack.rank }}</div>
                <div class="card-suit">
                  {{ getSuitSymbol(attack.suit) }}
                </div>
                <div class="card-bottom">{{ attack.rank }}</div>
              </div>
            </div>
            <div v-if="gameState.table.defending[index]" 
                 class="card defending"
                 :class="[gameState.table.defending[index].suit, takingCards ? 'taking' : '']">
              <div class="card-content">
                <div class="card-top">{{ gameState.table.defending[index].rank }}</div>
                <div class="card-suit">
                  {{ getSuitSymbol(gameState.table.defending[index].suit) }}
                </div>
                <div class="card-bottom">{{ gameState.table.defending[index].rank }}</div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Контролы игры -->
      <div class="game-actions" v-if="gameState.currentTurn === socket.id || gameState.nextDefender === socket.id">
        <button v-if="gameState.currentTurn === socket.id" @click="endTurn" :disabled="gameState.status === 'taking_cards'">
          Завершить ход
        </button>
        
        <button v-if="gameState.canTakeCards && gameState.nextDefender === socket.id" @click="takeCards">
          Взять карты
        </button>
      </div>

      <!-- Карты игрока -->
      <div class="player-hand">
        <div v-for="card in gameState.players.self.cards" 
             :key="`${card.suit}-${card.rank}`"
             class="card"
             :class=" [
               card.suit, 
               { 
                 'selected': selectedCard && selectedCard.suit === card.suit && selectedCard.rank === card.rank,
                 'playable': gameState.currentTurn === socket.id
               }
             ]"             @click="playCard(card)">
          <div class="card-content">
            <div class="card-top">{{ card.rank }}</div>
            <div class="card-suit">
              {{ getSuitSymbol(card.suit) }}
            </div>
            <div class="card-bottom">{{ card.rank }}</div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>

.app-container {
  width: 100vw;
  height: 100vh;
  background-image: url('/image/background_2.jpg');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
}

.game-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

.lobby-form {
  display: flex;
  flex-direction: column;
  gap: 20px;
  align-items: center;
  padding: 20px;
}

.input-group {
  display: flex;
  gap: 10px;
}

.input-group input {
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 1em;
}

.lobbies-list {
  width: 100%;
  max-width: 600px;
}

.lobby-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px;
  border: 1px solid #ccc;
  margin: 5px 0;
  border-radius: 4px;
}

.lobby-view {
  text-align: center;
  padding: 20px;
}

.players-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin: 20px 0;
}

.player-item {
  display: flex;
  gap: 10px;
  align-items: center;
  justify-content: center;
}

.ready-status {
  color: #4CAF50;
}

.host::after {
  content: "👑";
  margin-left: 5px;
}

.lobby-controls {
  margin-top: 20px;
}

.game-board {
  padding: px;
  background: #076324;
  border-radius: 10px;
  box-shadow: 0 0 10px rgba(0,0,0,0.3);
}

.other-players {
  display: flex;
  justify-content: space-around;
  flex-wrap: wrap;
  gap: 20px;
  margin-bottom: 20px;
}

.opponent {
  text-align: center;
  padding: 10px;
  border-radius: 8px;
  background: rgba(255,255,255,0.1);
}

.opponent.current {
  background: rgba(255,255,0,0.2);
}

.player-name {
  color: white;
  margin-bottom: 5px;
}

.card {
  width: 100px;
  height: 150px;
  background: white;
  border: 2px solid #333;
  border-radius: 10px;
  margin: 5px;
  position: relative;
  box-shadow: 2px 2px 5px rgba(0, 0, 0, 0.2);
}

.card-back {
  background: linear-gradient(45deg, #1a1a1a, #333);
  border: 0.5px solid gold;
  color: transparent;
}

.deck {
  left: left;
  top: 50%;
}


.trump-card {
  position: relative;
  left: 10px;
  top: -20px;
  z-index: 1;
  transform: rotate(-90deg);
  margin-left: 0;
  color: #e4e4e4;
}

.deck-area {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 0;
  margin-bottom: 20px;
  position: absolute;
  left: 0;
  top: 0;
  z-index: 2;
}

.card.deck {
  position: relative;
  z-index: 2;
  margin-bottom: -30px;
  color: #ffffff; /* Ярко-синий цвет для цифры количества карт */
  font-weight: bold;
  font-size: 1.5em;
}

.table {
  display: flex;
  flex-direction: row;
  align-items: flex-start;
  width: 100%;
  position: relative;
  min-height: 200px;
}

.playing-field {
  display: flex;
  flex-direction: row;
  justify-content: center;
  align-items: flex-end;
  min-height: 200px;
  padding: 20px;
  width: 100%;
  position: relative;
}

.playing-field .card-pair {
  position: relative;
  width: 100px;
  height: 150px;
  margin-left: -40px; /* увеличено расстояние между парами */
  z-index: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.playing-field .card-pair:first-child {
  margin-left: 0;
}

.playing-field .card {
  position: absolute;
  left: 0;
  top: 0;
}

.playing-field .defending {
  top: 30px;
  left: 20px;
  z-index: 2;
  box-shadow: 2px 2px 5px rgba(0,0,0,0.3);
}

/* Улучшаем отображение мастей */
.hearts, .diamonds {
  color: #ff4444;
  text-shadow: 0 0 2px rgba(255,255,255,0.5);
}

.clubs, .spades {
  color: #333;
  text-shadow: 0 0 2px rgba(255,255,255,0.5);
}

button {
  padding: 10px 20px;
  border: none;
  border-radius: 5px;
  background: #4CAF50;
  color: white;
  cursor: pointer;
  font-size: 1em;
  transition: background 0.3s;
}

button:hover:not(:disabled) {
  background: #45a049;
}

button:disabled {
  background: #cccccc;
  cursor: not-allowed;
  opacity: 0.7;
}

.card-content {
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  padding: 5px;
}

.card-top {
  text-align: left;
}

.card-suit {
  font-size: 24px;
  text-align: center;
}

.card-bottom {
  text-align: right;
  transform: rotate(180deg);
}

/* Анимация для взятия карт */
@keyframes shake {
  0%, 100% { transform: translateX(0); }
  25% { transform: translateX(-5px); }
  75% { transform: translateX(5px); }
}

.card.taking {
  animation: shake 0.5s ease-in-out;
}

.player-hand {
  display: flex;
  flex-direction: row;
  gap: 10px;
  justify-content: center;
  min-height: 160px;
  margin-top: 20px;
}

.opponent-hand {
  display: flex;
  flex-direction: row;
  gap: 10px;
  justify-content: center;
  min-height: 160px;
}

.table {
  display: flex;
  flex-direction: row;
  align-items: flex-start;
  width: 100%;
  position: relative;
  min-height: 200px;
}

.playing-field {
  display: flex;
  flex-direction: row;
  justify-content: center;
  align-items: flex-end;
  min-height: 200px;
  padding: 20px;
  width: 100%;
  position: relative;
}
</style>
