<template>
  <nav class="navbar" aria-label="Головна навігація">
    <div class="navbar-container">
      <router-link to="/" class="navbar-brand">
        Mafia Online
      </router-link>

      <button class="burger" @click="toggleMenu" :aria-expanded="menuOpen.toString()" aria-label="Відкрити меню">
        <span :class="{ 'open': menuOpen }"></span>
        <span :class="{ 'open': menuOpen }"></span>
        <span :class="{ 'open': menuOpen }"></span>
      </button>

      <div class="navbar-menu" :class="{ open: menuOpen }">
        <router-link to="/" class="navbar-link" @click="closeMenu">Головна</router-link>
        <router-link to="/rooms" class="navbar-link" @click="closeMenu">Кімнати</router-link>
        <router-link to="/leaderboard" class="navbar-link" @click="closeMenu">Таблиця лідерів</router-link>
      </div>

      <div class="navbar-actions">
        <ThemeToggle />
        <template v-if="isLoggedIn">
          <router-link to="/profile" class="navbar-link" @click="closeMenu">
            {{ username }}
          </router-link>
          <button @click="logout" class="btn btn-danger">Вийти</button>
        </template>
        <template v-else>
          <router-link to="/login" class="btn" @click="closeMenu">Увійти</router-link>
          <router-link to="/register" class="btn btn-secondary" @click="closeMenu">Реєстрація</router-link>
        </template>
      </div>
    </div>
  </nav>
</template>

<script setup>
import { ref, computed } from 'vue'
import { useRouter } from 'vue-router'
import { useAuthStore } from '@/stores/auth'
import { useToast } from 'vue-toastification'
import ThemeToggle from './ThemeToggle.vue'

const router = useRouter()
const authStore = useAuthStore()
const toast = useToast()

const isLoggedIn = computed(() => authStore.isLoggedIn)
const username = computed(() => authStore.getUser?.username)

const menuOpen = ref(false)
const toggleMenu = () => { menuOpen.value = !menuOpen.value }
const closeMenu = () => { menuOpen.value = false }

const logout = async () => {
  try {
    await authStore.logout()
    toast.success('Ви успішно вийшли з системи')
    router.push('/login')
  } catch (error) {
    toast.error('Помилка при виході з системи')
  }
}
</script>

<style scoped>
.navbar {
  background-color: var(--bg-secondary);
  box-shadow: var(--shadow-sm);
  padding: var(--spacing-md) var(--spacing-lg);
  width: 100%;
  position: relative;
  z-index: 10;
}

.navbar-container {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  box-sizing: border-box;
}

.navbar-brand {
  font-size: var(--font-size-xl);
  font-weight: bold;
  color: var(--primary-color);
  text-decoration: none;
  transition: color 0.2s;
}

.navbar-brand:hover {
  color: var(--accent-primary);
}

.burger {
  display: none;
  flex-direction: column;
  gap: 4px;
  background: none;
  border: none;
  cursor: pointer;
  margin-left: 1rem;
  z-index: 20;
}
.burger span {
  width: 28px;
  height: 3px;
  background: var(--text-primary);
  border-radius: 2px;
  transition: all 0.3s cubic-bezier(.4,2,.6,1);
}
.burger span.open:nth-child(1) {
  transform: translateY(7px) rotate(45deg);
}
.burger span.open:nth-child(2) {
  opacity: 0;
}
.burger span.open:nth-child(3) {
  transform: translateY(-7px) rotate(-45deg);
}

.navbar-menu {
  display: flex;
  gap: var(--spacing-lg);
  transition: max-height 0.4s cubic-bezier(.4,2,.6,1);
}

.navbar-link {
  color: var(--text-primary);
  text-decoration: none;
  transition: color 0.2s, background 0.2s, box-shadow 0.2s;
  border-radius: 6px;
  padding: 0.3rem 0.8rem;
}

.navbar-link:hover, .navbar-link:focus {
  color: var(--primary-color);
  background: var(--bg-tertiary);
  box-shadow: 0 2px 8px rgba(0,0,0,0.04);
}

.navbar-actions {
  display: flex;
  align-items: center;
  gap: var(--spacing-md);
}

.btn-secondary {
  background-color: var(--bg-tertiary);
  color: var(--text-primary);
}

.btn-secondary:hover {
  background-color: var(--bg-tertiary);
  opacity: 0.9;
}

@media (max-width: 900px) {
  .navbar-menu {
    position: absolute;
    top: 100%;
    left: 0;
    right: 0;
    background: var(--bg-secondary);
    flex-direction: column;
    align-items: flex-start;
    gap: 0;
    max-height: 0;
    overflow: hidden;
    box-shadow: 0 8px 24px rgba(0,0,0,0.08);
    border-radius: 0 0 12px 12px;
    transition: max-height 0.4s cubic-bezier(.4,2,.6,1);
  }
  .navbar-menu.open {
    max-height: 300px;
    padding: 0.5rem 1rem 1rem 1rem;
  }
  .burger {
    display: flex;
  }
}
</style> 