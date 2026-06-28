<script setup lang="ts">
import { ref } from "vue";

import type { Itinerary } from "./types";
import History from "./views/History.vue";
import Home from "./views/Home.vue";
import Result from "./views/Result.vue";

const currentView = ref<"home" | "result" | "history">("home");
const latestItinerary = ref<Itinerary | null>(null);
const transitioning = ref(false);

function switchView(view: "home" | "result" | "history") {
  if (view === currentView.value) return;
  transitioning.value = true;
  setTimeout(() => {
    currentView.value = view;
    setTimeout(() => {
      transitioning.value = false;
    }, 50);
  }, 180);
}

function handleGenerated(itinerary: Itinerary) {
  latestItinerary.value = itinerary;
  switchView("result");
}

function openTrip(itinerary: Itinerary) {
  latestItinerary.value = itinerary;
  switchView("result");
}

function updateCurrentItinerary(itinerary: Itinerary) {
  latestItinerary.value = itinerary;
  switchView("result");
}
</script>

<template>
  <div class="app-shell">
    <div class="app-shell__glow app-shell__glow--left"></div>
    <div class="app-shell__glow app-shell__glow--right"></div>
    <div class="app-shell__glow app-shell__glow--center"></div>

    <header class="hero">
      <div class="hero__inner">
        <div class="hero__badge">
          <span class="hero__badge-dot"></span>
          AI-Powered Travel Planner
        </div>
        <h1 class="hero__title">
          <span class="hero__title-icon">🗺️</span>
          智行云图
        </h1>
        <p class="hero__subtitle">AI 智能旅行规划平台 · 让每一次旅行都从容不迫</p>

        <nav class="hero__tabs">
          <button
            :class="['hero__tab', { 'hero__tab--active': currentView === 'home' }]"
            @click="switchView('home')"
          >
            <span class="hero__tab-icon">✈️</span>
            <span>规划行程</span>
          </button>
          <button
            :class="[
              'hero__tab',
              { 'hero__tab--active': currentView === 'result' },
              { 'hero__tab--disabled': !latestItinerary }
            ]"
            :disabled="!latestItinerary"
            @click="switchView('result')"
          >
            <span class="hero__tab-icon">📋</span>
            <span>查看结果</span>
          </button>
          <button
            :class="['hero__tab', { 'hero__tab--active': currentView === 'history' }]"
            @click="switchView('history')"
          >
            <span class="hero__tab-icon">📚</span>
            <span>历史记录</span>
          </button>
        </nav>
      </div>
    </header>

    <main class="page-content">
      <Transition name="page-fade" mode="out-in">
        <Home
          v-if="currentView === 'home'"
          key="home"
          @generated="handleGenerated"
        />
        <Result
          v-else-if="currentView === 'result'"
          key="result"
          :itinerary="latestItinerary"
          @back-home="switchView('home')"
          @view-history="switchView('history')"
          @updated="updateCurrentItinerary"
        />
        <History
          v-else
          key="history"
          :active="currentView === 'history'"
          @open-trip="openTrip"
        />
      </Transition>
    </main>

    <footer class="app-footer">
      <span>智行云图</span>
    </footer>
  </div>
</template>

<style scoped>
:global(body) {
  margin: 0;
  min-width: 320px;
  font-family: "Inter", "Noto Sans SC", "Microsoft YaHei", "PingFang SC", "Segoe UI", system-ui, sans-serif;
  background: #f0f4ff;
  color: #1e293b;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

:global(*) {
  box-sizing: border-box;
}

:global(::selection) {
  background: rgba(99, 102, 241, 0.2);
  color: #3730a3;
}

:global(::-webkit-scrollbar) {
  width: 6px;
  height: 6px;
}

:global(::-webkit-scrollbar-track) {
  background: transparent;
}

:global(::-webkit-scrollbar-thumb) {
  background: rgba(99, 102, 241, 0.2);
  border-radius: 20px;
}

:global(::-webkit-scrollbar-thumb:hover) {
  background: rgba(99, 102, 241, 0.35);
}

.app-shell {
  position: relative;
  min-height: 100vh;
  padding: 0 24px 32px;
  overflow: hidden;
  background:
    radial-gradient(ellipse 80% 60% at 50% -10%, rgba(99, 102, 241, 0.06) 0%, transparent 60%),
    radial-gradient(ellipse 60% 50% at 85% 50%, rgba(139, 92, 246, 0.05) 0%, transparent 50%),
    radial-gradient(ellipse 50% 50% at 15% 80%, rgba(59, 130, 246, 0.04) 0%, transparent 50%),
    linear-gradient(180deg, #f8fafc 0%, #f0f4ff 30%, #ede9fe 70%, #fafafe 100%);
}

.app-shell__glow {
  position: fixed;
  border-radius: 50%;
  filter: blur(80px);
  opacity: 0.4;
  pointer-events: none;
  z-index: 0;
}

.app-shell__glow--left {
  top: -180px;
  left: -120px;
  width: 500px;
  height: 500px;
  background: rgba(129, 140, 248, 0.35);
  animation: glow-pulse-left 8s ease-in-out infinite;
}

.app-shell__glow--right {
  top: 40%;
  right: -160px;
  width: 450px;
  height: 450px;
  background: rgba(167, 139, 250, 0.28);
  animation: glow-pulse-right 10s ease-in-out infinite;
}

.app-shell__glow--center {
  bottom: -120px;
  left: 50%;
  transform: translateX(-50%);
  width: 400px;
  height: 200px;
  background: rgba(99, 102, 241, 0.12);
  filter: blur(100px);
  animation: glow-pulse-center 12s ease-in-out infinite;
}

@keyframes glow-pulse-left {
  0%, 100% { opacity: 0.35; transform: scale(1); }
  50% { opacity: 0.55; transform: scale(1.08); }
}

@keyframes glow-pulse-right {
  0%, 100% { opacity: 0.28; transform: scale(1); }
  50% { opacity: 0.48; transform: scale(1.1); }
}

@keyframes glow-pulse-center {
  0%, 100% { opacity: 0.1; }
  50% { opacity: 0.2; }
}

/* ── Hero / Header ── */
.hero {
  position: relative;
  z-index: 1;
  padding-top: 36px;
  margin-bottom: 32px;
}

.hero__inner {
  max-width: 1280px;
  margin: 0 auto;
  text-align: center;
  padding: 48px 40px 36px;
  border-radius: 28px;
  background: linear-gradient(135deg, #6366f1 0%, #7c3aed 40%, #8b5cf6 70%, #a855f7 100%);
  box-shadow:
    0 4px 6px -1px rgba(99, 102, 241, 0.1),
    0 10px 15px -3px rgba(99, 102, 241, 0.08),
    0 20px 40px -8px rgba(99, 102, 241, 0.15),
    0 40px 80px -12px rgba(124, 58, 237, 0.12);
  position: relative;
  overflow: hidden;
}

.hero__inner::before {
  content: "";
  position: absolute;
  inset: 0;
  border-radius: 28px;
  background:
    radial-gradient(circle at 20% 30%, rgba(255, 255, 255, 0.1) 0%, transparent 50%),
    radial-gradient(circle at 80% 70%, rgba(255, 255, 255, 0.06) 0%, transparent 50%);
  pointer-events: none;
}

.hero__inner::after {
  content: "";
  position: absolute;
  top: -100px;
  right: -80px;
  width: 260px;
  height: 260px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.04);
  pointer-events: none;
}

.hero__badge {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 6px 16px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.18);
  backdrop-filter: blur(12px);
  color: rgba(255, 255, 255, 0.95);
  font-size: 12px;
  font-weight: 600;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  border: 1px solid rgba(255, 255, 255, 0.15);
}

.hero__badge-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: #34d399;
  box-shadow: 0 0 8px rgba(52, 211, 153, 0.6);
  animation: dot-pulse 2s ease-in-out infinite;
}

@keyframes dot-pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.4; }
}

.hero__title {
  margin: 20px 0 8px;
  color: #ffffff;
  font-size: 44px;
  font-weight: 800;
  letter-spacing: -0.02em;
  line-height: 1.15;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 14px;
}

.hero__title-icon {
  font-size: 40px;
  filter: drop-shadow(0 2px 8px rgba(0, 0, 0, 0.15));
}

.hero__subtitle {
  margin: 0 0 28px;
  color: rgba(255, 255, 255, 0.75);
  font-size: 15px;
  font-weight: 500;
  letter-spacing: 0.02em;
}

.hero__tabs {
  display: inline-flex;
  gap: 6px;
  padding: 6px;
  border-radius: 16px;
  background: rgba(255, 255, 255, 0.12);
  backdrop-filter: blur(16px);
  border: 1px solid rgba(255, 255, 255, 0.12);
}

.hero__tab {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  border: none;
  border-radius: 12px;
  padding: 10px 22px;
  background: transparent;
  color: rgba(255, 255, 255, 0.8);
  font-size: 14px;
  font-weight: 600;
  font-family: inherit;
  cursor: pointer;
  transition: all 0.25s cubic-bezier(0.4, 0, 0.2, 1);
  white-space: nowrap;
}

.hero__tab:hover:not(:disabled) {
  background: rgba(255, 255, 255, 0.1);
  color: #ffffff;
}

.hero__tab--active {
  background: #ffffff;
  color: #6366f1;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.hero__tab--active:hover {
  background: #ffffff;
  color: #6366f1;
}

.hero__tab--disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.hero__tab-icon {
  font-size: 15px;
}

/* ── Page content ── */
.page-content {
  position: relative;
  z-index: 1;
  max-width: 1280px;
  margin: 0 auto;
}

/* ── Page transitions ── */
.page-fade-enter-active {
  transition: all 0.25s cubic-bezier(0.4, 0, 0.2, 1);
}

.page-fade-leave-active {
  transition: all 0.18s cubic-bezier(0.4, 0, 0.2, 1);
}

.page-fade-enter-from {
  opacity: 0;
  transform: translateY(12px);
}

.page-fade-leave-to {
  opacity: 0;
  transform: translateY(-8px);
}

/* ── Footer ── */
.app-footer {
  position: relative;
  z-index: 1;
  text-align: center;
  padding: 36px 0 8px;
  color: #94a3b8;
  font-size: 12px;
  font-weight: 500;
}

/* ── Responsive ── */
@media (max-width: 768px) {
  .app-shell {
    padding: 0 12px 24px;
  }

  .hero {
    padding-top: 16px;
    margin-bottom: 20px;
  }

  .hero__inner {
    padding: 32px 20px 28px;
    border-radius: 20px;
  }

  .hero__title {
    font-size: 30px;
    gap: 8px;
  }

  .hero__title-icon {
    font-size: 28px;
  }

  .hero__subtitle {
    font-size: 13px;
    margin-bottom: 22px;
  }

  .hero__tab {
    padding: 8px 14px;
    font-size: 13px;
    gap: 4px;
  }

  .hero__tab-icon {
    font-size: 13px;
  }
}
</style>
