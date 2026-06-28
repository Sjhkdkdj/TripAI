<script setup lang="ts">
import { message } from "ant-design-vue";
import { onMounted, ref, watch } from "vue";

import { deleteTrip, getTripDetail, listTrips } from "../services/api";
import type { Itinerary, TripSummaryItem } from "../types";

const props = defineProps<{
  active: boolean;
}>();

const emit = defineEmits<{
  openTrip: [itinerary: Itinerary];
}>();

const loading = ref(false);
const items = ref<TripSummaryItem[]>([]);
const deletingTripId = ref("");

async function loadTrips() {
  loading.value = true;
  try {
    const response = await listTrips();
    items.value = response.items;
  } catch (error) {
    console.error(error);
    message.error("历史列表加载失败。");
  } finally {
    loading.value = false;
  }
}

async function openTrip(tripId: string) {
  try {
    const response = await getTripDetail(tripId);
    emit("openTrip", response.itinerary);
    message.success("已加载已保存行程。");
  } catch (error) {
    console.error(error);
    message.error("读取行程详情失败。");
  }
}

async function removeTrip(tripId: string) {
  const confirmed = window.confirm("确定要删除这条已保存行程吗？删除后无法恢复。");
  if (!confirmed) return;

  deletingTripId.value = tripId;
  try {
    await deleteTrip(tripId);
    items.value = items.value.filter((item) => item.trip_id !== tripId);
    message.success("行程已删除。");
  } catch (error) {
    console.error(error);
    message.error("删除行程失败。");
  } finally {
    deletingTripId.value = "";
  }
}

onMounted(() => {
  if (props.active) void loadTrips();
});

watch(() => props.active, (active) => {
  if (active) void loadTrips();
});
</script>

<template>
  <section class="history-page">
    <!-- Header -->
    <div class="history-header">
      <div class="history-header__info">
        <h2 class="history-header__title">
          <span>📚</span>
          历史行程
        </h2>
        <p class="history-header__desc">浏览和管理之前保存的所有旅行计划</p>
      </div>
      <button class="refresh-btn" @click="loadTrips">
        <span>🔄</span>
        刷新列表
      </button>
    </div>

    <!-- Loading -->
    <div v-if="loading" class="state-card">
      <span class="state-card__icon">⏳</span>
      <span>正在加载历史列表...</span>
    </div>

    <!-- Empty -->
    <div v-else-if="items.length === 0" class="state-card state-card--empty">
      <span class="state-card__icon">📭</span>
      <strong>还没有已保存的行程</strong>
      <p>生成行程后点击"保存行程"即可在这里查看</p>
    </div>

    <!-- Grid -->
    <div v-else class="history-grid">
      <article
        v-for="item in items"
        :key="item.trip_id"
        class="trip-card"
      >
        <div class="trip-card__header">
          <span class="trip-card__dest-icon">📍</span>
          <div>
            <div class="trip-card__dest">{{ item.destination }}</div>
            <div class="trip-card__id">{{ item.trip_id }}</div>
          </div>
        </div>

        <p class="trip-card__summary">{{ item.summary }}</p>

        <div class="trip-card__meta">
          <span v-if="item.updated_at" class="trip-card__time">
            🕐 {{ item.updated_at }}
          </span>
          <span v-else class="trip-card__time">暂无时间记录</span>
        </div>

        <div class="trip-card__actions">
          <button
            class="trip-card__btn trip-card__btn--primary"
            @click="openTrip(item.trip_id)"
          >
            查看详情
          </button>
          <button
            class="trip-card__btn trip-card__btn--danger"
            :disabled="deletingTripId === item.trip_id"
            @click="removeTrip(item.trip_id)"
          >
            {{ deletingTripId === item.trip_id ? "删除中..." : "删除" }}
          </button>
        </div>
      </article>
    </div>
  </section>
</template>

<style scoped>
.history-page {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

/* ── Header ── */
.history-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 16px;
  padding: 24px 28px;
  border-radius: 18px;
  background: #ffffff;
  box-shadow:
    0 1px 3px rgba(0, 0, 0, 0.04),
    0 4px 12px rgba(0, 0, 0, 0.04),
    0 8px 32px rgba(99, 102, 241, 0.06);
  border: 1px solid rgba(99, 102, 241, 0.06);
}

.history-header__title {
  margin: 0;
  font-size: 22px;
  font-weight: 800;
  color: #1e293b;
  display: flex;
  align-items: center;
  gap: 10px;
}

.history-header__desc {
  margin: 4px 0 0;
  font-size: 14px;
  color: #94a3b8;
}

.refresh-btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  border: none;
  border-radius: 12px;
  padding: 10px 20px;
  background: #eef2ff;
  color: #4338ca;
  font-size: 14px;
  font-weight: 600;
  font-family: inherit;
  cursor: pointer;
  transition: all 0.2s ease;
  white-space: nowrap;
}

.refresh-btn:hover {
  background: #e0e7ff;
  box-shadow: 0 2px 8px rgba(99, 102, 241, 0.1);
}

/* ── State cards ── */
.state-card {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 32px 28px;
  border-radius: 18px;
  background: #ffffff;
  box-shadow:
    0 1px 3px rgba(0, 0, 0, 0.04),
    0 4px 12px rgba(0, 0, 0, 0.04),
    0 8px 32px rgba(99, 102, 241, 0.06);
  border: 1px solid rgba(99, 102, 241, 0.06);
  color: #64748b;
  font-size: 14px;
}

.state-card--empty {
  flex-direction: column;
  text-align: center;
  padding: 56px 28px;
}

.state-card--empty strong {
  font-size: 18px;
  color: #334155;
}

.state-card--empty p {
  margin: 4px 0 0;
  color: #94a3b8;
  font-size: 14px;
}

.state-card__icon {
  font-size: 32px;
}

/* ── Trip cards grid ── */
.history-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 16px;
}

/* ── Trip card ── */
.trip-card {
  display: flex;
  flex-direction: column;
  gap: 12px;
  padding: 22px;
  border-radius: 18px;
  background: #ffffff;
  box-shadow:
    0 1px 3px rgba(0, 0, 0, 0.04),
    0 4px 12px rgba(0, 0, 0, 0.04),
    0 8px 32px rgba(99, 102, 241, 0.06);
  border: 1px solid rgba(99, 102, 241, 0.06);
  transition: all 0.3s ease;
}

.trip-card:hover {
  transform: translateY(-2px);
  box-shadow:
    0 2px 6px rgba(0, 0, 0, 0.05),
    0 8px 20px rgba(0, 0, 0, 0.06),
    0 16px 40px rgba(99, 102, 241, 0.1);
  border-color: rgba(99, 102, 241, 0.15);
}

.trip-card__header {
  display: flex;
  align-items: center;
  gap: 10px;
}

.trip-card__dest-icon {
  font-size: 22px;
}

.trip-card__dest {
  font-size: 20px;
  font-weight: 800;
  color: #1e293b;
  line-height: 1.2;
}

.trip-card__id {
  font-size: 11px;
  color: #94a3b8;
  font-family: "SF Mono", "Fira Code", monospace;
  word-break: break-all;
  margin-top: 2px;
}

.trip-card__summary {
  margin: 0;
  color: #475569;
  font-size: 14px;
  line-height: 1.7;
  display: -webkit-box;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.trip-card__meta {
  margin-top: auto;
}

.trip-card__time {
  font-size: 12px;
  color: #94a3b8;
  font-weight: 500;
}

.trip-card__actions {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px;
  padding-top: 4px;
}

.trip-card__btn {
  border: none;
  border-radius: 10px;
  padding: 10px 14px;
  font-size: 13px;
  font-weight: 600;
  font-family: inherit;
  cursor: pointer;
  transition: all 0.2s ease;
}

.trip-card__btn:disabled {
  opacity: 0.6;
  cursor: wait;
}

.trip-card__btn--primary {
  background: linear-gradient(135deg, #6366f1 0%, #7c3aed 100%);
  color: #ffffff;
  box-shadow: 0 2px 8px rgba(99, 102, 241, 0.2);
}

.trip-card__btn--primary:hover:not(:disabled) {
  box-shadow: 0 4px 16px rgba(99, 102, 241, 0.3);
  transform: translateY(-1px);
}

.trip-card__btn--danger {
  background: #fef2f2;
  color: #dc2626;
}

.trip-card__btn--danger:hover:not(:disabled) {
  background: #fee2e2;
}

/* ── Responsive ── */
@media (max-width: 600px) {
  .history-header {
    flex-direction: column;
    align-items: flex-start;
  }

  .history-grid {
    grid-template-columns: 1fr;
  }

  .trip-card__actions {
    grid-template-columns: 1fr;
  }
}
</style>
