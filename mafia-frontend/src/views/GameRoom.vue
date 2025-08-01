<template>
  <div class="game-room">
    <div class="game-header">
      <h1>
        {{ room.name }}
      </h1>
      <ThemeToggle />
      <div class="room-status">
        <span>Гравців: {{ room.players_number }}/{{ room.max_players_number }}</span>
        <span v-if="isOwner" class="owner-badge">Ви власник</span>
      </div>
      <div class="game-phase">
        <span>Фаза: {{ gamePhase === 'day' ? 'День' : gamePhase === 'night' ? 'Ніч' : 'Очікування' }}</span>
        <span v-if="currentRound > 0">Раунд: {{ currentRound }}</span>
      </div>
    </div>

    <div class="game-content">
      <div class="players-list card">
        <h2>Гравці</h2>
        <div class="players-grid">
          <div v-for="player in players" :key="player.id || player.username" class="player-card" :class="{ 'dead': !player.is_alive }">
            <div class="player-avatar">
              {{ player.id }}
            </div>
            <div class="player-info">
              <span class="player-name">{{ player.username || 'Unknown' }}</span>
              <div class="player-status">
                <span v-if="player.is_owner" class="owner-badge" title="Власник">
                  <svg width="18" height="18" viewBox="0 0 24 24" fill="none"><path d="M2 19h20M4 19l2-8h12l2 8M7 11V7a5 5 0 0 1 10 0v4" stroke="#ff1744" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/><ellipse cx="12" cy="7" rx="3" ry="2" fill="#ffd600"/></svg>
                  Власник
                </span>
                <span v-if="player.is_ready" class="ready-badge" title="Готовий">
                  <svg width="18" height="18" viewBox="0 0 24 24" fill="none"><circle cx="12" cy="12" r="10" fill="#43a047"/><path d="M8 12l2 2l4-4" stroke="#fff" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/></svg>
                  Готовий
                </span>
                <span v-if="!player.is_alive" class="dead-badge" title="Мертвий">
                  <svg width="18" height="18" viewBox="0 0 24 24" fill="none"><circle cx="12" cy="12" r="10" fill="#e53935"/><path d="M9 9l6 6M15 9l-6 6" stroke="#fff" stroke-width="2" stroke-linecap="round"/></svg>
                  Мертвий
                </span>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div class="game-chat card">
        <div class="chat-messages" ref="chatMessages">
          <div v-for="message in messages" :key="message.id" class="message" :class="{
            'isPersonal': message.type === 'personal',
            'isSystem': message.type === 'system',
            'isError': message.type === 'error'
          }">
            <template v-if="message.type === 'chat'">
            <span class="message-sender">{{ message.username }}:</span>
              <span class="message-text">{{ message.message }}</span>
            </template>
            <template v-else-if="message.type === 'system'">
              <span class="message-text system-message">{{ message.message }}</span>
            </template>
            <template v-else-if="message.type === 'error'">
              <span class="message-text error-message">{{ message.message }}</span>
            </template>
            <template v-else>
              <span class="message-text">{{ message.message }}</span>
            </template>
          </div>
        </div>
        <div class="chat-input">
          <input
            type="text"
            v-model="newMessage"
            @keyup.enter="sendMessage"
            placeholder="Введіть повідомлення..."
            class="input"
          />
          <button @click="sendMessage" class="button">Надіслати</button>
        </div>
      </div>
    </div>

    <div class="game-controls">
      <button 
        v-if="!isGameStarted"
        @click="toggleReady" 
        class="button"
        :class="{ 'ready': isReady }"
      >
        {{ isReady ? 'Готовий' : 'Готуватися' }}
      </button>
      <button 
        v-if="isOwner && canStartGame" 
        @click="startGame" 
        class="start-game-button"
      >
        Почати гру
      </button>
      <div v-else-if="isOwner" class="start-game-info">
        <template v-if="players.length < room.min_players_number">
          Очікування гравців ({{ players.length }}/{{ room.min_players_number }})
        </template>
        <template v-else-if="!players.every(p => p.is_ready)">
          Очікування готовності гравців
        </template>
      </div>
      <button
        v-if="canVote && gamePhase === 'day'"
        @click="showVoteModal = true"
        class="button vote-button"
      >
        Голосувати
      </button>
    </div>

    <!-- Модальне вікно ролі -->
    <div v-if="showRoleModal" class="modal">
      <div class="modal-content">
        <h3>Ваша роль</h3>
        <p class="role-name">{{ 
          myRole === 'mafia' ? 'Мафія' : 
          myRole === 'doctor' ? 'Лікар' : 
          myRole === 'detective' ? 'Детектив' : 
          'Мирний житель' 
        }}</p>
        
        <div v-if="myRole === 'mafia' && otherMafia.length > 0" class="other-mafia">
          <h4>Інші мафійники:</h4>
          <ul>
            <li v-for="mafia in otherMafia" :key="mafia.id">
              {{ mafia.name }}
            </li>
          </ul>
        </div>
        
        <button @click="showRoleModal = false" class="button">Зрозуміло</button>
      </div>
    </div>

    <!-- Модальне вікно голосування -->
    <div v-if="showVoteModal" class="modal">
      <div class="modal-content">
        <h3>Голосування</h3>
        <p class="vote-description">Оберіть гравця, якого хочете вигнати:</p>
        <div class="vote-options">
          <div v-for="player in alivePlayers" :key="player.id" 
            class="vote-option" 
            @click="vote(player.id)">
            <div class="vote-option-avatar">{{ player.id }}</div>
            <div class="vote-option-name">{{ player.name }}</div>
          </div>
        </div>
        <button @click="showVoteModal = false" class="button secondary">Скасувати</button>
      </div>
    </div>

    <!-- Модальне вікно нічних дій -->
    <div v-if="showNightActionModal" class="modal">
      <div class="modal-content">
        <h3>{{ 
          myRole === 'mafia' ? 'Виберіть жертву' : 
          myRole === 'doctor' ? 'Виберіть кого лікувати' : 
          'Виберіть кого перевірити' 
        }}</h3>
        <div class="night-actions">
          <div v-for="player in alivePlayers" :key="player.id" 
            class="action-option" 
            @click="performNightAction(player.id)">
            <div class="action-option-avatar">{{ player.id }}</div>
            <div class="action-option-name">{{ player.name }}</div>
          </div>
        </div>
        <button @click="showNightActionModal = false" class="button secondary">Скасувати</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed, nextTick } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import api from '@/services/api'
import ThemeToggle from '@/components/ThemeToggle.vue'

const route = useRoute()
const router = useRouter()
const room = ref({})
const players = ref([])
const messages = ref([])
const newMessage = ref('')
const isReady = ref(false)
const isGameStarted = ref(false)
const chatMessages = ref(null)
const ws = ref(null)
const reconnectAttempts = ref(0)
const maxReconnectAttempts = 3
const reconnectTimeout = ref(null)
const updateInterval = ref(null)
const roomOwner = ref(null)
const gamePhase = ref('waiting')
const currentRound = ref(0)
const myRole = ref(null)
const otherMafia = ref([])
const selectedPlayer = ref(null)
const showRoleModal = ref(false)
const showVoteModal = ref(false)
const showNightActionModal = ref(false)

const isOwner = computed(() => {
  const userId = parseInt(localStorage.getItem('userId'))
  return room.value.owner === userId
})

const canStartGame = computed(() => {
  const playersCount = players.value.length
  const minPlayers = room.value?.min_players_number || 6
  const allReady = players.value.every(p => p.is_ready)
  
  console.log('Can start game check:', {
    playersCount,
    minPlayers,
    allReady,
    canStart: playersCount >= minPlayers && allReady,
    players: players.value.map(p => ({ id: p.id, name: p.name, is_ready: p.is_ready }))
  })
  
  return playersCount >= minPlayers && allReady
})

const alivePlayers = computed(() => {
    return players.value.filter(player => player.is_alive)
})

const canPerformNightAction = computed(() => {
    if (gamePhase.value !== 'night') return false
    if (!myRole.value) return false
    return ['mafia', 'doctor', 'detective'].includes(myRole.value)
})

const canVote = computed(() => {
    return gamePhase.value === 'day' && myRole.value
})

const gameStatus = computed(() => {
    if (gamePhase.value === 'waiting') return 'Очікування гравців'
    if (gamePhase.value === 'day') return 'Денна фаза'
    if (gamePhase.value === 'night') return 'Нічна фаза'
    return 'Гра завершена'
})

const fetchRoom = async () => {
  try {
    const response = await api.get(`/api/rooms/${route.params.id}`)
    room.value = response.data
    roomOwner.value = response.data.owner
    console.log('Room data updated:', {
      room: room.value,
      owner: roomOwner.value,
      userId: localStorage.getItem('userId')
    })
    await fetchPlayers()
  } catch (error) {
    console.error('Помилка отримання інформації про кімнату:', error)
    router.push('/rooms')
  }
}

const fetchPlayers = async () => {
  try {
    const response = await api.get(`/api/rooms/${route.params.id}/players`)
    players.value = response.data.map(player => ({
      ...player,
      is_owner: player.id === room.value.owner,
      is_alive: player.is_alive ?? true
    }))
    console.log('Players data:', players.value)
    console.log('Can start game check:', {
      playersCount: players.value.length,
      minPlayers: room.value.min_players_number,
      allReady: players.value.every(player => player.is_ready)
    })
  } catch (error) {
    console.error('Помилка отримання списку гравців:', error)
  }
}

const fetchMessages = async () => {
  try {
    const response = await api.get(`/api/rooms/${route.params.id}/messages`)
    messages.value = response.data.map(msg => ({
      id: msg.id,
      username: msg.username,
      text: msg.message,
      timestamp: msg.created_at
    }))
    // Прокручуємо чат вниз після завантаження повідомлень
    setTimeout(() => {
      if (chatMessages.value) {
        chatMessages.value.scrollTop = chatMessages.value.scrollHeight
      }
    }, 0)
  } catch (error) {
    console.error('Помилка отримання повідомлень:', error)
  }
}

const connectWebSocket = () => {
  if (ws.value && (ws.value.readyState === WebSocket.OPEN || ws.value.readyState === WebSocket.CONNECTING)) {
    console.log('WebSocket already connected or connecting')
    return
  }

  if (reconnectTimeout.value) {
    clearTimeout(reconnectTimeout.value)
  }

  const token = localStorage.getItem('token')
  if (!token) {
    console.error('No token found in localStorage')
    router.push('/login')
    return
  }

  if (!route.params.id) {
    console.error('No room ID found')
    router.push('/rooms')
    return
  }

  const wsUrl = `ws://localhost:8000/api/ws/room/${route.params.id}?token=${token}`
  console.log('Attempting WebSocket connection to:', wsUrl)
  
  try {
    ws.value = new WebSocket(wsUrl)
    
    ws.value.onopen = () => {
      console.log('WebSocket connected successfully')
      reconnectAttempts.value = 0
      if (reconnectTimeout.value) {
        clearTimeout(reconnectTimeout.value)
        reconnectTimeout.value = null
      }
      fetchRoom()
      fetchPlayers()
      fetchMessages()
    }
    
    ws.value.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data)
        console.log('Received WebSocket message:', data)
        handleWebSocketMessage(event)
      } catch (error) {
        console.error('Error parsing message:', error)
      }
    }
    
    ws.value.onerror = (error) => {
      console.error('WebSocket error:', error)
      if (error.target && error.target.readyState) {
        console.log('WebSocket state at error:', error.target.readyState)
      }
    }
    
    ws.value.onclose = (event) => {
      console.log('WebSocket closed:', event.code, event.reason)
      
      if (reconnectTimeout.value) {
        clearTimeout(reconnectTimeout.value)
      }
      
      switch (event.code) {
        case 4000:
          console.error('No token provided')
          router.push('/login')
          return
        case 4001:
          console.error('Invalid token')
          router.push('/login')
          return
        case 4002:
          console.error('Room not found')
          router.push('/rooms')
          return
        case 4003:
          console.error('Failed to add player to room')
          router.push('/rooms')
          return
      }
      
      if (event.code !== 1000 && reconnectAttempts.value < maxReconnectAttempts && route.params.id) {
        reconnectAttempts.value++
        console.log(`Reconnection attempt ${reconnectAttempts.value}/${maxReconnectAttempts}`)
        reconnectTimeout.value = setTimeout(connectWebSocket, 2000 * reconnectAttempts.value)
      } else if (reconnectAttempts.value >= maxReconnectAttempts) {
        console.error('Maximum reconnection attempts reached')
        router.push('/rooms')
      }
    }
  } catch (error) {
    console.error('Error creating WebSocket:', error)
    if (reconnectAttempts.value < maxReconnectAttempts && route.params.id) {
      reconnectAttempts.value++
      reconnectTimeout.value = setTimeout(connectWebSocket, 2000 * reconnectAttempts.value)
    }
  }
}

const handleWebSocketMessage = (event) => {
  try {
    const data = JSON.parse(event.data);
    console.log('Received WebSocket message:', data);
  
    switch (data.type) {
      case 'room_state':
        console.log('Room state update:', data);
        gamePhase.value = data.room.phase;
        currentRound.value = data.room.round;
        players.value = data.players;
        break;

      case 'player_joined':
        console.log('Player joined:', data);
        players.value = data.players;
        messages.value.push({
          type: 'system',
          message: `${data.username} приєднався до гри`
        });
        break;

      case 'player_left':
        console.log('Player left:', data);
        players.value = data.players;
        messages.value.push({
          type: 'system',
          message: `${data.username} покинув гру`
        });
        break;

      case 'player_ready':
        console.log('Player ready state changed:', data);
        players.value = data.players;
        const player = players.value.find(p => p.id === data.player_id);
        if (player) {
          messages.value.push({
            type: 'system',
            message: `${player.name} ${player.is_ready ? 'готовий' : 'не готовий'} до гри`
          });
        }
        break;

      case 'role_assigned':
        console.log('Role assigned:', data);
        myRole.value = data.role;
        showRoleModal.value = true;
        
        if (data.role === 'mafia' && data.other_mafia) {
          otherMafia.value = data.other_mafia;
        }
        
        const roleMessage = {
          type: 'personal',
          message: `Ваша роль: ${data.role}`
        };
        messages.value.push(roleMessage);
        break;

      case 'game_started':
        console.log('Game started:', data);
        gamePhase.value = data.phase;
        currentRound.value = data.round;
        isGameStarted.value = true;
        players.value = data.players;
        messages.value.push({
          type: 'system',
          message: 'Гра почалася!'
        });
        break;

      case 'phase_change':
        console.log('Phase change:', data);
        gamePhase.value = data.phase;
        currentRound.value = data.round;
        
        if (data.phase === 'night') {
          if (['mafia', 'doctor', 'detective'].includes(myRole.value)) {
            showNightActionModal.value = true;
          }
        }
        
        messages.value.push({
          type: 'system',
          message: data.phase === 'night' ? 'Настала ніч' : 'Настав день'
        });
        break;

      case 'chat':
        console.log('Chat message:', data);
        messages.value.push({
          type: 'chat',
          username: data.username,
          message: data.message
        });
        break;

      case 'system':
        console.log('System message:', data);
        messages.value.push({
          type: 'system',
          message: data.message
        });
        break;

      case 'error':
        console.error('Error message:', data);
        messages.value.push({
          type: 'error',
          message: data.message
        });
        break;

      default:
        console.warn('Unknown message type:', data.type);
    }

    // Прокручуємо чат вниз після оновлення повідомлень
    nextTick(() => {
      const chatContainer = document.querySelector('.chat-messages');
      if (chatContainer) {
        chatContainer.scrollTop = chatContainer.scrollHeight;
      }
    });
  } catch (error) {
    console.error('Error handling WebSocket message:', error);
  }
};

const sendMessage = () => {
  if (!newMessage.value.trim() || !ws.value) return
  
  if (ws.value.readyState === WebSocket.OPEN) {
    const message = {
      type: 'chat',
      payload: {
        message: newMessage.value
      }
    }
    
    try {
      ws.value.send(JSON.stringify(message))
      newMessage.value = ''
    } catch (error) {
      console.error('Помилка відправки повідомлення:', error)
      if (reconnectAttempts.value < maxReconnectAttempts) {
        connectWebSocket()
      }
    }
  } else {
    console.warn('WebSocket не підключено')
    if (reconnectAttempts.value < maxReconnectAttempts) {
      connectWebSocket()
    }
  }
}

const toggleReady = async () => {
  if (!ws.value || ws.value.readyState !== WebSocket.OPEN) {
    console.warn('WebSocket не підключено')
    return
  }
  
  const message = {
    type: 'toggle_ready',
    payload: {
      is_ready: !isReady.value
    }
  }
  
  console.log('Sending ready toggle:', message)
  
  try {
    ws.value.send(JSON.stringify(message))
    isReady.value = !isReady.value
    console.log('Ready state changed to:', isReady.value)
  } catch (error) {
    console.error('Помилка відправки статусу готовності:', error)
  }
}

const sendWebSocketMessage = async (message) => {
  if (!ws.value || ws.value.readyState !== WebSocket.OPEN) {
    console.warn('WebSocket не підключено')
    throw new Error('WebSocket не підключено')
  }
  
  try {
    ws.value.send(JSON.stringify(message))
  } catch (error) {
    console.error('Помилка відправки повідомлення:', error)
    throw error
  }
}

const startGame = async () => {
  if (!canStartGame.value) {
    console.log('Cannot start game:', {
      playersCount: players.value.length,
      minPlayers: room.value?.min_players_number || 6,
      allReady: players.value.every(p => p.is_ready)
    })
    return
  }
  
  try {
    const message = {
      type: 'start_game',
      payload: {
        room_id: route.params.id
    }
    }
    console.log('Sending start game message:', message)
      ws.value.send(JSON.stringify(message))
      console.log('Start game message sent successfully')
  } catch (error) {
    console.error('Error starting game:', error)
  }
}

const performNightAction = async (targetId) => {
  if (!myRole.value) return;
  
  try {
    const message = {
      type: 'night_action',
      payload: {
        actor_id: parseInt(localStorage.getItem('userId')),
        target_id: targetId
      }
    };
    
    if (ws.value && ws.value.readyState === WebSocket.OPEN) {
      await ws.value.send(JSON.stringify(message));
      showNightActionModal.value = false;
      player.value.is_ready = True
      messages.value.push({
        type: 'system',
        message: `Ви виконали нічну дію як ${myRole.value}`
      });
    } else {
      console.warn('WebSocket не підключено');
    }
  } catch (error) {
    console.error('Error performing night action:', error);
  }
};

const vote = async (targetId) => {
  try {
    const message = {
      type: 'vote',
      payload: {
        player_id: localStorage.getItem('userId'),
        target_id: targetId
      }
    }
    
    if (ws.value && ws.value.readyState === WebSocket.OPEN) {
      ws.value.send(JSON.stringify(message))
      showVoteModal.value = false
    } else {
      console.warn('WebSocket не підключено')
    }
  } catch (error) {
    console.error('Error voting:', error)
  }
}

onMounted(() => {
  console.log('Component mounted')
  connectWebSocket()
  fetchMessages()
  
  updateInterval.value = setInterval(() => {
    if (route.params.id) {
      console.log('Update interval tick')
      fetchRoom()
    }
  }, 5000)
})

onUnmounted(() => {
  if (ws.value) {
    ws.value.close()
  }
  if (updateInterval.value) {
    clearInterval(updateInterval.value)
  }
  if (reconnectTimeout.value) {
    clearTimeout(reconnectTimeout.value)
  }
})
</script>

<style scoped>
.game-room {
  font-family: var(--font-main);
  background: var(--color-bg);
  color: var(--color-text);
  min-height: 100vh;
  padding: 0;
  margin: 0;
  transition: background var(--transition), color var(--transition);
}
.game-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 2rem 2rem 1rem 2rem;
  background: var(--color-card);
  box-shadow: var(--color-shadow);
  border-bottom: 2px solid var(--color-accent);
}
.game-header h1 {
  font-family: 'Bebas Neue', 'Montserrat', cursive;
  font-size: 2.5rem;
  letter-spacing: 2px;
  color: var(--color-accent);
  text-shadow: 0 2px 12px var(--color-accent-glow);
  margin: 0;
}
.room-status {
  font-size: 1.1rem;
  color: var(--color-text);
  display: flex;
  align-items: center;
  gap: 1rem;
}
.owner-badge {
  background: var(--color-owner);
  color: #222;
  padding: 2px 10px;
  border-radius: 6px;
  font-weight: bold;
  margin-left: 1rem;
  box-shadow: 0 0 8px #ff1744cc;
}
.game-content {
  display: flex;
  gap: 2rem;
  padding: 2rem;
  justify-content: center;
}
.players-list {
  background: var(--color-card);
  border-radius: 16px;
  box-shadow: var(--color-shadow);
  padding: 1.5rem;
  min-width: 300px;
  max-width: 340px;
  flex-shrink: 0;
}
.players-list h2 {
  color: var(--color-accent);
  margin-bottom: 1rem;
  font-size: 1.3rem;
  letter-spacing: 1px;
}
.players-grid {
  display: flex;
  flex-direction: column;
  gap: 18px;
}
.player-card {
  display: flex;
  align-items: center;
  gap: 18px;
  padding: 14px 18px;
  border-radius: 10px;
  background: var(--color-secondary);
  box-shadow: 0 2px 12px #0004;
  border: 2px solid transparent;
  transition: border var(--transition), box-shadow var(--transition), background var(--transition), opacity .3s;
  position: relative;
  animation: fadeIn .7s;
}
.player-card:hover {
  border: 2px solid var(--color-accent);
  box-shadow: 0 0 16px var(--color-accent-glow);
}
.player-card.dead {
  opacity: 0.5;
  background: var(--color-dead);
  border: 2px solid var(--color-danger);
  text-decoration: line-through;
}
.player-avatar {
  width: 48px;
  height: 48px;
  border-radius: 50%;
  background: var(--color-avatar);
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  font-size: 1.5rem;
  border: 2px solid var(--color-accent);
  box-shadow: 0 0 12px var(--color-accent-glow);
  transition: box-shadow var(--transition);
}
.player-info {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 2px;
}
.player-name {
  font-weight: 700;
  color: var(--color-text);
  font-size: 1.15rem;
  letter-spacing: 1px;
}
.player-status {
  display: flex;
  gap: 8px;
  margin-top: 2px;
}
.ready-badge {
  background: var(--color-success);
  color: #fff;
  padding: 2px 8px;
  border-radius: 4px;
  font-size: 0.9em;
  font-weight: bold;
  box-shadow: 0 0 8px #43a04799;
}
.dead-badge {
  background: var(--color-danger);
  color: #fff;
  padding: 2px 8px;
  border-radius: 4px;
  font-size: 0.9em;
  font-weight: bold;
  box-shadow: 0 0 8px #e5393599;
}
.game-chat {
  flex: 1;
  background: var(--color-card);
  border-radius: 16px;
  box-shadow: var(--color-shadow);
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
  min-width: 350px;
}
.chat-messages {
  flex: 1;
  overflow-y: auto;
  margin-bottom: 1.5rem;
  padding: 1rem;
  background: var(--color-chat-bg);
  border-radius: 10px;
  box-shadow: 0 1px 8px #0002;
  font-size: 1.05rem;
  max-height: 350px;
  scroll-behavior: smooth;
}
.message {
  margin-bottom: 10px;
  animation: fadeIn .5s;
  padding: 8px 12px;
  border-radius: 8px;
  background: var(--color-chat-bg);
}
.message.isPersonal {
  background: var(--color-accent);
  color: white;
  font-weight: bold;
  border: 2px solid var(--color-accent-glow);
  box-shadow: 0 0 12px var(--color-accent-glow);
}
.message-sender {
  font-weight: 600;
  margin-right: 8px;
  color: var(--color-accent);
}
.message-system {
  color: var(--color-chat-system);
  font-style: italic;
  text-shadow: 0 0 8px var(--color-accent-glow);
}
.chat-input {
  display: flex;
  gap: 10px;
}
.input {
  flex: 1;
  padding: 10px 14px;
  border: 1.5px solid var(--color-border);
  border-radius: 6px;
  background: var(--color-secondary);
  color: var(--color-text);
  font-size: 1.1rem;
  transition: border var(--transition), background var(--transition);
}
.input:focus {
  border: 1.5px solid var(--color-accent);
  background: var(--color-card);
  outline: none;
}
.button {
  padding: 12px 28px;
  border: none;
  border-radius: 8px;
  background: var(--color-btn);
  color: var(--color-btn-text);
  font-size: 1.1rem;
  font-weight: 700;
  cursor: pointer;
  box-shadow: 0 2px 12px #d32f2f33;
  transition: background var(--transition), box-shadow var(--transition), transform .1s;
  margin: 0 4px;
  letter-spacing: 1px;
}
.button:hover, .button:focus {
  background: var(--color-btn-hover);
  box-shadow: 0 4px 24px var(--color-accent-glow);
  transform: translateY(-2px) scale(1.04);
}
.button.ready {
  background: var(--color-success);
}
.button.start-game {
  background: var(--color-owner);
  color: #222;
  font-size: 1.15rem;
  font-weight: bold;
  box-shadow: 0 0 16px #ffd60099;
}
.button:disabled, .button[disabled] {
  background: #888;
  color: #eee;
  cursor: not-allowed;
  opacity: 0.7;
  box-shadow: none;
}
.game-controls {
  margin-top: 2rem;
  display: flex;
  gap: 18px;
  justify-content: center;
}
.start-game-button {
  background: #ff0000 !important;
  color: white !important;
  padding: 10px 20px !important;
  border: none !important;
  border-radius: 5px !important;
  cursor: pointer !important;
  margin-left: 10px !important;
  font-size: 16px !important;
  font-weight: bold !important;
  box-shadow: 0 2px 4px rgba(0,0,0,0.2) !important;
}

.start-game-button:hover {
  background: #cc0000 !important;
  transform: translateY(-2px) !important;
  box-shadow: 0 4px 8px rgba(0,0,0,0.3) !important;
}
@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px);}
  to { opacity: 1; transform: none;}
}

.game-phase {
  display: flex;
  gap: 10px;
}

.modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
}

.modal-content {
  background: white;
  padding: 20px;
  border-radius: 8px;
  min-width: 300px;
}

.vote-options, .night-actions {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
  gap: 10px;
  margin: 15px 0;
}

.vote-option, .action-option {
  padding: 10px;
  background: #f5f5f5;
  border-radius: 4px;
  cursor: pointer;
  text-align: center;
}

.vote-option:hover, .action-option:hover {
  background: #e0e0e0;
}

.other-mafia {
  margin-top: 15px;
  padding: 10px;
  background: rgba(255, 0, 0, 0.1);
  border-radius: 5px;
}

.other-mafia h4 {
  color: #ff0000;
  margin-bottom: 10px;
}

.other-mafia ul {
  list-style: none;
  padding: 0;
}

.other-mafia li {
  padding: 5px 0;
  color: #ff0000;
  font-weight: bold;
}

.vote-button {
  background: var(--color-accent);
  color: white;
  font-weight: bold;
  box-shadow: 0 0 12px var(--color-accent-glow);
}

.vote-description {
  color: var(--color-text);
  margin-bottom: 1rem;
  text-align: center;
}

.vote-option {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px;
  background: var(--color-secondary);
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
}

.vote-option:hover {
  background: var(--color-accent);
  color: white;
  transform: translateY(-2px);
  box-shadow: 0 4px 12px var(--color-accent-glow);
}

.vote-option-avatar {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background: var(--color-avatar);
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  font-size: 1.1rem;
}

.vote-option-name {
  font-weight: 500;
}

.message.isSystem {
  background: var(--color-system);
  color: var(--color-text);
  font-style: italic;
  border: 1px solid var(--color-system-border);
  box-shadow: 0 0 8px var(--color-system-glow);
}

.message.isError {
  background: var(--color-error);
  color: white;
  font-weight: bold;
  border: 1px solid var(--color-error-border);
  box-shadow: 0 0 8px var(--color-error-glow);
}

.system-message {
  color: var(--color-system-text);
  font-style: italic;
}

.error-message {
  color: var(--color-error-text);
  font-weight: bold;
}

.start-game-info {
  background: var(--color-secondary);
  padding: 10px 20px;
  border-radius: 5px;
  color: var(--color-text);
  font-size: 14px;
  margin-left: 10px;
  border: 1px solid var(--color-border);
  display: flex;
  align-items: center;
  gap: 8px;
}

.start-game-info::before {
  content: '⏳';
  font-size: 16px;
}

.role-name {
  font-size: 1.5rem;
  font-weight: bold;
  color: var(--color-accent);
  text-align: center;
  margin: 1rem 0;
  padding: 1rem;
  background: var(--color-secondary);
  border-radius: 8px;
  border: 2px solid var(--color-accent);
}

.action-option {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px;
  background: var(--color-secondary);
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
}

.action-option:hover {
  background: var(--color-accent);
  color: white;
  transform: translateY(-2px);
  box-shadow: 0 4px 12px var(--color-accent-glow);
}

.action-option-avatar {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background: var(--color-avatar);
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  font-size: 1.1rem;
}

.action-option-name {
  font-weight: 500;
}
</style>
