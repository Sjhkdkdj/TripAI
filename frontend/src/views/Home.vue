<script setup lang="ts">
import axios from "axios";
import { computed, reactive, ref } from "vue";
import { message } from "ant-design-vue";

import { generateTrip } from "../services/api";
import type { Itinerary, TripRequestPayload } from "../types";

const emit = defineEmits<{
  generated: [itinerary: Itinerary];
}>();

const preferenceOptions = [
  { label: "🏔️ 自然风景", value: "自然风景" },
  { label: "📷 拍照打卡", value: "拍照" },
  { label: "🍜 美食探索", value: "美食" },
  { label: "🏘️ 古镇风情", value: "古镇" },
  { label: "☕ 休闲慢游", value: "休闲" },
];

const dietaryOptions = [
  { label: "少辣", value: "少辣" },
  { label: "不吃香菜", value: "不吃香菜" },
  { label: "不吃葱", value: "不吃葱" },
];

function formatDate(date: Date): string {
  const y = date.getFullYear();
  const m = String(date.getMonth() + 1).padStart(2, "0");
  const d = String(date.getDate()).padStart(2, "0");
  return `${y}-${m}-${d}`;
}

const today = new Date();
const todayPlus2 = new Date(today);
todayPlus2.setDate(todayPlus2.getDate() + 2);

const formState = reactive({
  destination: "大理",
  startDate: formatDate(today),
  endDate: formatDate(todayPlus2),
  travelers: 2,
  budget: 3200,
  hotelLevel: "舒适型",
  pace: "轻松",
  preferences: ["自然风景", "拍照", "美食"],
  dietaryPreferences: ["少辣"],
  notes: "不想太早起床，希望安排一个适合看日落的地点。",
});

const isSubmitting = ref(false);

const dayCount = computed(() => {
  const start = new Date(formState.startDate);
  const end = new Date(formState.endDate);
  const diff = end.getTime() - start.getTime();
  return Number.isNaN(diff) ? 0 : Math.max(Math.floor(diff / 86400000) + 1, 0);
});

async function handleSubmit() {
  const payload: TripRequestPayload = {
    destination: formState.destination,
    start_date: formState.startDate,
    end_date: formState.endDate,
    travelers: formState.travelers,
    budget: formState.budget,
    preferences: formState.preferences,
    pace: formState.pace,
    dietary_preferences: formState.dietaryPreferences,
    hotel_level: formState.hotelLevel,
    special_notes: formState.notes,
  };

  isSubmitting.value = true;
  try {
    const itinerary = await generateTrip(payload);
    message.success("行程生成成功，已切换到结果页。");
    emit("generated", itinerary);
  } catch (error) {
    console.error(error);
    if (axios.isAxiosError(error)) {
      if (error.code === "ECONNABORTED") {
        message.error("行程生成超时，模型返回较慢，请稍后再试。");
      } else if (error.response) {
        message.error(`行程生成失败：后端返回 ${error.response.status}。`);
      } else {
        message.error("行程生成失败，请检查前端到后端的连接。");
      }
    } else {
      message.error("行程生成失败，请检查后端地址或服务状态。");
    }
  } finally {
    isSubmitting.value = false;
  }
}
</script>

<template>
  <section class="home-page">
    <!-- 目的地与日期 -->
    <div class="planner-card">
      <div class="card-header">
        <div class="card-header__icon card-header__icon--blue">📍</div>
        <div>
          <div class="card-header__title">目的地与日期</div>
          <div class="card-header__desc">选择你想去的城市和旅行时间段</div>
        </div>
      </div>

      <div class="form-grid form-grid--5col">
        <div class="form-field">
          <label class="field-label">目的地城市</label>
          <a-input
            v-model:value="formState.destination"
            placeholder="输入目的地"
            size="large"
            class="modern-input"
          />
        </div>
        <div class="form-field">
          <label class="field-label">开始日期</label>
          <a-input
            v-model:value="formState.startDate"
            size="large"
            class="modern-input"
          />
        </div>
        <div class="form-field">
          <label class="field-label">结束日期</label>
          <a-input
            v-model:value="formState.endDate"
            size="large"
            class="modern-input"
          />
        </div>
        <div class="form-field">
          <label class="field-label">出行人数</label>
          <a-input-number
            v-model:value="formState.travelers"
            :min="1"
            size="large"
            style="width: 100%"
            class="modern-input"
          />
        </div>
        <div class="form-field">
          <label class="field-label">旅行天数</label>
          <div class="day-count-badge">
            <span class="day-count-badge__num">{{ dayCount }}</span>
            <span class="day-count-badge__unit">天</span>
          </div>
        </div>
      </div>
    </div>

    <!-- 偏好设置 -->
    <div class="planner-card">
      <div class="card-header">
        <div class="card-header__icon card-header__icon--purple">⚙️</div>
        <div>
          <div class="card-header__title">偏好设置</div>
          <div class="card-header__desc">自定义你的旅行风格和预算</div>
        </div>
      </div>

      <div class="form-grid form-grid--3col">
        <div class="form-field">
          <label class="field-label">旅行节奏</label>
          <a-select
            v-model:value="formState.pace"
            size="large"
            class="modern-select"
            :options="[
              { label: '🕊️  轻松 — 慢享时光', value: '轻松' },
              { label: '🚶  适中 — 张弛有度', value: '适中' },
              { label: '🏃  紧凑 — 高效打卡', value: '紧凑' }
            ]"
          />
        </div>
        <div class="form-field">
          <label class="field-label">住宿档次</label>
          <a-select
            v-model:value="formState.hotelLevel"
            size="large"
            class="modern-select"
            :options="[
              { label: '🏨  舒适型', value: '舒适型' },
              { label: '🏩  高档型', value: '高档型' },
              { label: '🛏️  经济型', value: '经济型' }
            ]"
          />
        </div>
        <div class="form-field">
          <label class="field-label">
            预算范围
            <span class="field-label__hint">(人均/元)</span>
          </label>
          <a-input-number
            v-model:value="formState.budget"
            :min="0"
            :step="500"
            size="large"
            style="width: 100%"
            class="modern-input"
          >
            <template #addonBefore>¥</template>
          </a-input-number>
        </div>
      </div>

      <div class="preference-section">
        <label class="field-label">旅行兴趣偏好</label>
        <div class="preference-chips">
          <label
            v-for="opt in preferenceOptions"
            :key="opt.value"
            class="preference-chip"
            :class="{ 'preference-chip--checked': formState.preferences.includes(opt.value) }"
          >
            <input
              type="checkbox"
              :value="opt.value"
              v-model="formState.preferences"
              class="preference-chip__input"
            />
            <span class="preference-chip__check">✓</span>
            <span>{{ opt.label }}</span>
          </label>
        </div>
      </div>

      <div class="preference-section">
        <label class="field-label">饮食偏好</label>
        <div class="preference-chips">
          <label
            v-for="opt in dietaryOptions"
            :key="opt.value"
            class="preference-chip"
            :class="{ 'preference-chip--checked': formState.dietaryPreferences.includes(opt.value) }"
          >
            <input
              type="checkbox"
              :value="opt.value"
              v-model="formState.dietaryPreferences"
              class="preference-chip__input"
            />
            <span class="preference-chip__check">✓</span>
            <span>{{ opt.label }}</span>
          </label>
        </div>
      </div>
    </div>

    <!-- 额外要求 -->
    <div class="planner-card">
      <div class="card-header">
        <div class="card-header__icon card-header__icon--green">💬</div>
        <div>
          <div class="card-header__title">额外要求</div>
          <div class="card-header__desc">补充其他需求，AI 会尽力满足</div>
        </div>
      </div>
      <a-textarea
        v-model:value="formState.notes"
        :rows="4"
        placeholder="例如：不想太早起床、想看日落、避开游客高峰时段..."
        class="modern-textarea"
      />
    </div>

    <!-- 提交按钮 -->
    <div class="submit-panel">
      <button
        class="submit-btn"
        :class="{ 'submit-btn--loading': isSubmitting }"
        :disabled="isSubmitting"
        @click="handleSubmit"
      >
        <span v-if="!isSubmitting" class="submit-btn__content">
          <svg class="submit-btn__icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M5 12h14M12 5l7 7-7 7"/>
          </svg>
          开始规划行程
        </span>
        <span v-else class="submit-btn__content">
          <span class="submit-btn__spinner"></span>
          AI 正在生成行程...
        </span>
      </button>
      <p class="submit-hint">
        基于真实地理数据与 AI 大模型，自动生成详细的旅行计划
      </p>
    </div>
  </section>
</template>

<style scoped>
.home-page {
  display: grid;
  gap: 16px;
}

/* ── Card ── */
.planner-card {
  padding: 28px;
  border-radius: 20px;
  background: #ffffff;
  box-shadow:
    0 1px 3px rgba(0, 0, 0, 0.04),
    0 4px 12px rgba(0, 0, 0, 0.04),
    0 8px 32px rgba(99, 102, 241, 0.06);
  border: 1px solid rgba(99, 102, 241, 0.06);
  transition: box-shadow 0.3s ease;
}

.planner-card:hover {
  box-shadow:
    0 1px 3px rgba(0, 0, 0, 0.04),
    0 4px 16px rgba(0, 0, 0, 0.06),
    0 12px 40px rgba(99, 102, 241, 0.08);
}

/* ── Card header ── */
.card-header {
  display: flex;
  align-items: flex-start;
  gap: 14px;
  margin-bottom: 24px;
  padding-bottom: 18px;
  border-bottom: 1px solid #f1f5f9;
}

.card-header__icon {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 42px;
  height: 42px;
  border-radius: 14px;
  font-size: 18px;
  flex-shrink: 0;
}

.card-header__icon--blue {
  background: #eff6ff;
  box-shadow: 0 4px 12px rgba(59, 130, 246, 0.1);
}

.card-header__icon--purple {
  background: #f5f3ff;
  box-shadow: 0 4px 12px rgba(139, 92, 246, 0.1);
}

.card-header__icon--green {
  background: #ecfdf5;
  box-shadow: 0 4px 12px rgba(16, 185, 129, 0.1);
}

.card-header__title {
  font-size: 16px;
  font-weight: 700;
  color: #1e293b;
  margin-bottom: 2px;
}

.card-header__desc {
  font-size: 13px;
  color: #94a3b8;
}

/* ── Form grid ── */
.form-grid {
  display: grid;
  gap: 16px;
}

.form-grid--5col {
  grid-template-columns: 2fr 1.2fr 1.2fr 1fr 1fr;
}

.form-grid--3col {
  grid-template-columns: repeat(3, 1fr);
}

.form-field {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

/* ── Labels ── */
.field-label {
  font-size: 13px;
  font-weight: 600;
  color: #475569;
  letter-spacing: 0.01em;
}

.field-label__hint {
  font-weight: 400;
  color: #94a3b8;
}

/* ── Modern inputs ── */
:deep(.modern-input) {
  border-radius: 12px !important;
  border-color: #e2e8f0 !important;
  transition: all 0.2s ease !important;
  font-family: inherit !important;
}

:deep(.modern-input:hover) {
  border-color: #c4b5fd !important;
}

:deep(.modern-input:focus),
:deep(.modern-input.ant-input-focused) {
  border-color: #6366f1 !important;
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.1) !important;
}

:deep(.modern-select .ant-select-selector) {
  border-radius: 12px !important;
  border-color: #e2e8f0 !important;
  transition: all 0.2s ease !important;
  font-family: inherit !important;
}

:deep(.modern-select:hover .ant-select-selector) {
  border-color: #c4b5fd !important;
}

:deep(.modern-select.ant-select-focused .ant-select-selector) {
  border-color: #6366f1 !important;
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.1) !important;
}

/* ── Day count badge ── */
.day-count-badge {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 4px;
  min-height: 40px;
  border-radius: 12px;
  background: linear-gradient(135deg, #6366f1 0%, #8b5cf6 100%);
  color: #ffffff;
  box-shadow: 0 4px 16px rgba(99, 102, 241, 0.25);
}

.day-count-badge__num {
  font-size: 22px;
  font-weight: 800;
  line-height: 1;
}

.day-count-badge__unit {
  font-size: 13px;
  font-weight: 500;
  opacity: 0.9;
}

/* ── Preference chips ── */
.preference-section {
  margin-top: 22px;
}

.preference-chips {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  margin-top: 10px;
}

.preference-chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 9px 16px;
  border-radius: 12px;
  background: #f8fafc;
  border: 2px solid #e2e8f0;
  font-size: 14px;
  font-weight: 500;
  color: #64748b;
  cursor: pointer;
  transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1);
  user-select: none;
}

.preference-chip:hover {
  border-color: #c4b5fd;
  background: #faf9ff;
  color: #6366f1;
}

.preference-chip--checked {
  background: #eef2ff;
  border-color: #6366f1;
  color: #4338ca;
  font-weight: 600;
  box-shadow: 0 2px 8px rgba(99, 102, 241, 0.12);
}

.preference-chip__input {
  position: absolute;
  opacity: 0;
  width: 0;
  height: 0;
}

.preference-chip__check {
  font-size: 12px;
  opacity: 0;
  transform: scale(0);
  transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1);
}

.preference-chip--checked .preference-chip__check {
  opacity: 1;
  transform: scale(1);
  color: #6366f1;
}

/* ── Textarea ── */
:deep(.modern-textarea) {
  border-radius: 12px !important;
  border-color: #e2e8f0 !important;
  font-family: inherit !important;
  font-size: 14px !important;
  line-height: 1.7 !important;
  resize: vertical;
  transition: all 0.2s ease !important;
}

:deep(.modern-textarea:hover) {
  border-color: #c4b5fd !important;
}

:deep(.modern-textarea:focus) {
  border-color: #6366f1 !important;
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.1) !important;
}

/* ── Submit button ── */
.submit-panel {
  padding: 8px 0 0;
  text-align: center;
}

.submit-btn {
  min-width: 260px;
  border: none;
  border-radius: 16px;
  padding: 16px 36px;
  background: linear-gradient(135deg, #6366f1 0%, #7c3aed 100%);
  color: #ffffff;
  font-size: 16px;
  font-weight: 700;
  font-family: inherit;
  cursor: pointer;
  box-shadow:
    0 4px 16px rgba(99, 102, 241, 0.25),
    0 8px 32px rgba(124, 58, 237, 0.2);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  position: relative;
  overflow: hidden;
}

.submit-btn::before {
  content: "";
  position: absolute;
  inset: 0;
  background: linear-gradient(135deg, #818cf8 0%, #8b5cf6 100%);
  opacity: 0;
  transition: opacity 0.3s ease;
}

.submit-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow:
    0 8px 24px rgba(99, 102, 241, 0.3),
    0 12px 40px rgba(124, 58, 237, 0.25);
}

.submit-btn:hover:not(:disabled)::before {
  opacity: 1;
}

.submit-btn:active:not(:disabled) {
  transform: translateY(0);
  box-shadow:
    0 2px 8px rgba(99, 102, 241, 0.2),
    0 4px 16px rgba(124, 58, 237, 0.15);
}

.submit-btn:disabled {
  cursor: wait;
  opacity: 0.85;
}

.submit-btn--loading {
  background: linear-gradient(135deg, #818cf8 0%, #a78bfa 100%);
}

.submit-btn__content {
  position: relative;
  z-index: 1;
  display: inline-flex;
  align-items: center;
  gap: 10px;
}

.submit-btn__icon {
  width: 20px;
  height: 20px;
}

.submit-btn__spinner {
  width: 18px;
  height: 18px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-top-color: #ffffff;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.submit-hint {
  margin: 14px 0 0;
  color: #94a3b8;
  font-size: 13px;
  font-weight: 500;
}

/* ── Responsive ── */
@media (max-width: 900px) {
  .form-grid--5col {
    grid-template-columns: 1fr 1fr;
  }

  .form-grid--3col {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 600px) {
  .planner-card {
    padding: 20px;
  }

  .form-grid--5col {
    grid-template-columns: 1fr;
  }

  .preference-chips {
    gap: 8px;
  }

  .preference-chip {
    padding: 8px 12px;
    font-size: 13px;
  }
}
</style>
