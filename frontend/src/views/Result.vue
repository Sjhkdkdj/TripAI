<script setup lang="ts">
import { computed, ref, watch } from "vue";
import { message } from "ant-design-vue";

import AmapTripMap from "../components/AmapTripMap.vue";
import {
  editTrip,
  fetchWeatherForecast,
  getMarkdownExportUrl,
  getPdfExportUrl,
  saveTrip,
} from "../services/api";
import type { Itinerary, WeatherForecastResponse } from "../types";

const props = defineProps<{
  itinerary: Itinerary | null;
}>();

const emit = defineEmits<{
  backHome: [];
  viewHistory: [];
  updated: [itinerary: Itinerary];
}>();

const activeSection = ref("overview");
const saving = ref(false);
const exportingPdf = ref(false);
const exportingMarkdown = ref(false);
const editing = ref(false);
const editScope = ref("day_1");
const editInstruction = ref("这一天节奏更轻松一点，减少固定安排。");
const weatherLoading = ref(false);
const weatherError = ref("");
const weather = ref<WeatherForecastResponse | null>(null);

const sidebarSections = [
  { key: "overview", label: "行程概览", icon: "📋" },
  { key: "budget", label: "预算明细", icon: "💰" },
  { key: "map", label: "景点地图", icon: "🗺️" },
  { key: "weather", label: "天气信息", icon: "🌤️" },
  { key: "edit", label: "智能调整", icon: "🤖" },
  { key: "day-budget", label: "按天花费", icon: "📊" },
  { key: "points", label: "点位明细", icon: "📍" },
  { key: "days", label: "每日行程", icon: "📅" },
];

function scrollToSection(key: string) {
  activeSection.value = key;
  const el = document.getElementById(`section-${key}`);
  if (el) {
    el.scrollIntoView({ behavior: "smooth", block: "start" });
  }
}

function formatShortDate(dateText?: string | null): string {
  if (!dateText) return "待定";
  const parts = dateText.split("-");
  if (parts.length !== 3) return dateText;
  return `${parts[1]}-${parts[2]}`;
}

function formatWeatherDate(dateText?: string | null, week?: string | null): string {
  const weekdayMap: Record<string, string> = {
    "1": "周一", "2": "周二", "3": "周三", "4": "周四",
    "5": "周五", "6": "周六", "7": "周日",
  };
  const weekday = week ? weekdayMap[week] || `周${week}` : "";
  return [formatShortDate(dateText), weekday].filter(Boolean).join(" ");
}

const budgetItems = computed(() => {
  if (!props.itinerary) return [];
  const budget = props.itinerary.budget_breakdown;
  return [
    { label: "景点门票", value: budget.tickets, icon: "🎫", color: "#6366f1" },
    { label: "酒店住宿", value: budget.hotel, icon: "🏨", color: "#8b5cf6" },
    { label: "餐饮费用", value: budget.meals, icon: "🍽️", color: "#f59e0b" },
    { label: "交通费用", value: budget.transport, icon: "🚗", color: "#10b981" },
  ];
});

const dayBudgetItems = computed(() => {
  if (!props.itinerary) return [];
  return props.itinerary.days.map((day) => {
    const tickets = day.spots.reduce((sum, spot) => sum + (spot.estimated_cost ?? 0), 0);
    const meals = day.meals.reduce((sum, meal) => sum + (meal.estimated_cost ?? 0), 0);
    const transport = day.transport.reduce((sum, item) => sum + (item.estimated_cost ?? 0), 0);
    const hotel = day.hotel?.estimated_cost ?? 0;
    return {
      key: day.day_index,
      title: `第${day.day_index}天`,
      subtitle: day.theme || "未命名主题",
      tickets, meals, transport, hotel,
      total: tickets + meals + transport + hotel,
    };
  });
});

const mapPoints = computed(() => {
  if (!props.itinerary) return [];
  return props.itinerary.days.flatMap((day) =>
    day.spots.map((spot) => ({
      key: `${day.day_index}-${spot.name}`,
      dayIndex: day.day_index,
      date: day.date || "待定",
      theme: day.theme || "未命名主题",
      name: spot.name,
      address: spot.address || spot.location || "待补充",
      latitude: spot.latitude,
      longitude: spot.longitude,
      poiId: spot.poi_id,
      imageUrl: spot.image_url,
      description: spot.description || "暂无说明",
    }))
  );
});

const technicalTipKeywords = [
  "LLM", "RAG", "LangChain", "Chroma", "演示", "测试", "规则", "模型", "源码",
];

const rainWeatherKeywords = ["雨", "阵雨", "雷阵雨", "小雨", "中雨", "大雨"];
const sunnyTipKeywords = ["防晒", "太阳", "日照", "晒"];

const weatherText = computed(() => {
  if (!weather.value) return "";
  return weather.value.days
    .map((day) => `${day.day_weather || ""}${day.night_weather || ""}`)
    .join(" ");
});

const hasRainyWeather = computed(() =>
  rainWeatherKeywords.some((keyword) => weatherText.value.includes(keyword))
);

const displayTips = computed(() => {
  if (!props.itinerary) return [];
  let tips = props.itinerary.tips
    .map((tip) => tip.trim())
    .filter(Boolean)
    .filter((tip) => !technicalTipKeywords.some((keyword) => tip.includes(keyword)));

  if (hasRainyWeather.value) {
    tips = tips.filter((tip) => !sunnyTipKeywords.some((keyword) => tip.includes(keyword)));
    tips.push("🌂 天气可能有雨，建议随身带伞或轻便雨衣。");
    tips.push("👟 阴雨天路面湿滑，洱海边和古镇石板路建议穿防滑鞋。");
  }

  const uniqueTips = Array.from(new Set(tips));
  if (uniqueTips.length) return uniqueTips;
  return [
    `建议根据${props.itinerary.destination}当天实时天气准备雨具或薄外套。`,
    "古镇、生态廊道和石板路更适合慢慢走，鞋子尽量选择舒适防滑的款式。",
  ];
});

function buildVisibleItinerary(): Itinerary | null {
  if (!props.itinerary) return null;
  return { ...props.itinerary, tips: displayTips.value };
}

async function loadWeather() {
  if (!props.itinerary?.destination) {
    weather.value = null;
    return;
  }
  weatherLoading.value = true;
  weatherError.value = "";
  try {
    weather.value = await fetchWeatherForecast(props.itinerary.destination);
  } catch (error) {
    console.error(error);
    weather.value = null;
    weatherError.value = "天气信息加载失败。";
  } finally {
    weatherLoading.value = false;
  }
}

watch(() => props.itinerary?.destination, () => { void loadWeather(); }, { immediate: true });

watch(() => props.itinerary?.trip_id, () => {
  const firstDay = props.itinerary?.days[0];
  editScope.value = firstDay ? `day_${firstDay.day_index}` : "day_1";
}, { immediate: true });

async function openPdfExport() {
  const itineraryToExport = buildVisibleItinerary();
  if (!itineraryToExport) return;
  const exportWindow = window.open("about:blank", "_blank");
  exportingPdf.value = true;
  try {
    await saveTrip(itineraryToExport);
    const exportUrl = getPdfExportUrl(itineraryToExport.trip_id);
    if (exportWindow) exportWindow.location.href = exportUrl;
    else window.location.href = exportUrl;
  } catch (error) {
    console.error(error);
    exportWindow?.close();
    message.error("导出 PDF 前同步当前行程失败。");
  } finally {
    exportingPdf.value = false;
  }
}

async function openMarkdownExport() {
  const itineraryToExport = buildVisibleItinerary();
  if (!itineraryToExport) return;
  const exportWindow = window.open("about:blank", "_blank");
  exportingMarkdown.value = true;
  try {
    await saveTrip(itineraryToExport);
    const exportUrl = getMarkdownExportUrl(itineraryToExport.trip_id);
    if (exportWindow) exportWindow.location.href = exportUrl;
    else window.location.href = exportUrl;
  } catch (error) {
    console.error(error);
    exportWindow?.close();
    message.error("导出 Markdown 前同步当前行程失败。");
  } finally {
    exportingMarkdown.value = false;
  }
}

async function handleSave() {
  const itineraryToSave = buildVisibleItinerary();
  if (!itineraryToSave) return;
  saving.value = true;
  try {
    await saveTrip(itineraryToSave);
    message.success("行程已保存，可以去历史列表查看。");
  } catch (error) {
    console.error(error);
    message.error("保存行程失败。");
  } finally {
    saving.value = false;
  }
}

async function handleEdit() {
  if (!props.itinerary) return;
  const instruction = editInstruction.value.trim();
  if (!instruction) {
    message.warning("请先输入想如何调整行程。");
    return;
  }
  editing.value = true;
  try {
    const updatedItinerary = await editTrip({
      trip_id: props.itinerary.trip_id,
      current_itinerary: props.itinerary,
      user_instruction: instruction,
      edit_scope: editScope.value,
      preserve_constraints: ["保留预算结构", "保留目的地和旅行日期"],
    });
    emit("updated", updatedItinerary);
    message.success("行程已智能调整。");
  } catch (error) {
    console.error(error);
    message.error("智能调整失败，请稍后再试。");
  } finally {
    editing.value = false;
  }
}
</script>

<template>
  <!-- 有结果时 -->
  <section v-if="itinerary" class="result-page">
    <!-- 侧边栏导航 -->
    <aside class="sidebar">
      <div class="sidebar__header">
        <div class="sidebar__logo">🗺️</div>
        <div class="sidebar__title">行程导航</div>
      </div>
      <nav class="sidebar__nav">
        <button
          v-for="sec in sidebarSections"
          :key="sec.key"
          :class="['sidebar__link', { 'sidebar__link--active': activeSection === sec.key }]"
          @click="scrollToSection(sec.key)"
        >
          <span class="sidebar__link-icon">{{ sec.icon }}</span>
          <span>{{ sec.label }}</span>
        </button>
      </nav>

      <div class="sidebar__actions">
        <button class="sidebar-btn sidebar-btn--secondary" @click="$emit('backHome')">
          ← 返回规划
        </button>
        <button class="sidebar-btn sidebar-btn--primary" :disabled="saving" @click="handleSave">
          {{ saving ? "保存中..." : "💾 保存行程" }}
        </button>
        <button class="sidebar-btn sidebar-btn--ghost" @click="$emit('viewHistory')">
          📚 历史列表
        </button>
        <button class="sidebar-btn sidebar-btn--outline" :disabled="exportingPdf" @click="openPdfExport">
          {{ exportingPdf ? "准备中..." : "📄 导出 PDF" }}
        </button>
        <button class="sidebar-btn sidebar-btn--outline-green" :disabled="exportingMarkdown" @click="openMarkdownExport">
          {{ exportingMarkdown ? "准备中..." : "📝 导出 Markdown" }}
        </button>
      </div>
    </aside>

    <!-- 主内容区 -->
    <div class="result-main">
      <!-- 概览 -->
      <section id="section-overview" class="card">
        <div class="card__header">
          <span class="card__header-icon">📋</span>
          <div>
            <h2 class="card__title">{{ itinerary.destination }} · 旅行计划</h2>
            <p class="card__subtitle">
              {{ itinerary.days[0]?.date || "待定" }} — {{ itinerary.days[itinerary.days.length - 1]?.date || "待定" }}
              · 共 {{ itinerary.days.length }} 天
            </p>
          </div>
          <span class="card__badge">ID: {{ itinerary.trip_id }}</span>
        </div>
        <p class="overview-summary">{{ itinerary.summary }}</p>

        <div v-if="displayTips.length" class="tips-box">
          <div class="tips-box__title">💡 旅行提示</div>
          <ul class="tips-box__list">
            <li v-for="tip in displayTips" :key="tip">{{ tip }}</li>
          </ul>
        </div>
      </section>

      <!-- 预算明细 -->
      <section id="section-budget" class="card">
        <div class="card__header">
          <span class="card__header-icon">💰</span>
          <h2 class="card__title">预算明细</h2>
        </div>
        <div class="budget-grid">
          <div v-for="item in budgetItems" :key="item.label" class="budget-card" :style="{'--accent': item.color}">
            <span class="budget-card__icon">{{ item.icon }}</span>
            <div class="budget-card__label">{{ item.label }}</div>
            <div class="budget-card__value">¥{{ item.value.toFixed(0) }}</div>
          </div>
        </div>
        <div class="budget-total-bar">
          <span>预估总费用</span>
          <strong>¥{{ itinerary.estimated_budget.toFixed(0) }}</strong>
        </div>
      </section>

      <!-- 景点地图 -->
      <section id="section-map" class="card card--map">
        <div class="card__header">
          <span class="card__header-icon">🗺️</span>
          <h2 class="card__title">景点地图</h2>
        </div>
        <AmapTripMap :points="mapPoints" />
      </section>

      <!-- 天气 -->
      <section id="section-weather" class="card card--weather">
        <div class="card__header">
          <span class="card__header-icon">🌤️</span>
          <h2 class="card__title">天气预报</h2>
        </div>
        <div v-if="weatherLoading" class="status-text">⏳ 正在加载天气信息...</div>
        <div v-else-if="weatherError" class="status-text status-text--error">{{ weatherError }}</div>
        <div v-else-if="weather" class="weather-grid">
          <article v-for="day in weather.days" :key="`${day.date}-${day.week}`" class="weather-card">
            <div class="weather-card__date">{{ formatWeatherDate(day.date, day.week) }}</div>
            <div class="weather-card__temps">
              <span class="weather-card__high">{{ day.day_temp || "-" }}°</span>
              <span class="weather-card__sep">/</span>
              <span class="weather-card__low">{{ day.night_temp || "-" }}°</span>
            </div>
            <div class="weather-card__desc">
              <span>☀️ 白天：{{ day.day_weather || "未知" }}</span>
              <span>🌙 夜间：{{ day.night_weather || "未知" }}</span>
            </div>
          </article>
        </div>
        <div v-else class="status-text">暂无天气信息。</div>
      </section>

      <!-- 智能调整 -->
      <section id="section-edit" class="card">
        <div class="card__header">
          <span class="card__header-icon">🤖</span>
          <h2 class="card__title">智能调整行程</h2>
        </div>
        <div class="edit-panel">
          <div class="edit-panel__row">
            <label class="edit-field">
              <span class="edit-field__label">调整范围</span>
              <select v-model="editScope" class="modern-select-native">
                <option
                  v-for="day in itinerary.days"
                  :key="day.day_index"
                  :value="`day_${day.day_index}`"
                >
                  第{{ day.day_index }}天 · {{ day.theme || "未命名主题" }}
                </option>
              </select>
            </label>
            <button
              class="edit-btn"
              :class="{ 'edit-btn--loading': editing }"
              :disabled="editing"
              @click="handleEdit"
            >
              {{ editing ? "调整中..." : "✨ 智能调整" }}
            </button>
          </div>
          <textarea
            v-model="editInstruction"
            class="modern-textarea-native"
            rows="3"
            placeholder="例如：第二天轻松一点，不要安排太满；第三天想换成适合看日落的地点。"
          ></textarea>
        </div>
      </section>

      <!-- 按天花费 -->
      <section id="section-day-budget" class="card">
        <div class="card__header">
          <span class="card__header-icon">📊</span>
          <h2 class="card__title">按天花费明细</h2>
        </div>
        <div class="day-budget-grid">
          <article v-for="item in dayBudgetItems" :key="item.key" class="day-budget-card">
            <div class="day-budget-card__head">
              <strong>{{ item.title }}</strong>
              <span class="day-budget-card__theme">{{ item.subtitle }}</span>
            </div>
            <div class="day-budget-card__rows">
              <div class="day-budget-row"><span>门票</span><strong>¥{{ item.tickets.toFixed(0) }}</strong></div>
              <div class="day-budget-row"><span>餐饮</span><strong>¥{{ item.meals.toFixed(0) }}</strong></div>
              <div class="day-budget-row"><span>交通</span><strong>¥{{ item.transport.toFixed(0) }}</strong></div>
              <div class="day-budget-row"><span>住宿</span><strong>¥{{ item.hotel.toFixed(0) }}</strong></div>
              <div class="day-budget-row day-budget-row--total">
                <span>当日合计</span><strong>¥{{ item.total.toFixed(0) }}</strong>
              </div>
            </div>
          </article>
        </div>
      </section>

      <!-- 点位明细 -->
      <section id="section-points" class="card">
        <div class="card__header">
          <span class="card__header-icon">📍</span>
          <h2 class="card__title">地图点位明细</h2>
        </div>
        <div class="point-grid">
          <article v-for="point in mapPoints" :key="point.key" class="point-card">
            <div class="point-card__head">
              <span>Day {{ point.dayIndex }} · {{ point.name }}</span>
              <span class="point-card__date">{{ formatShortDate(point.date) }}</span>
            </div>
            <div class="point-card__body">
              <div
                v-if="point.imageUrl"
                class="point-card__img"
                :style="{ backgroundImage: `url(${point.imageUrl})` }"
              ></div>
              <div v-else class="point-card__img point-card__img--empty">
                <span>🏞️</span>
                <span>暂无图片</span>
              </div>
              <div class="point-card__info"><strong>主题：</strong>{{ point.theme }}</div>
              <div class="point-card__info"><strong>地址：</strong>{{ point.address }}</div>
              <p class="point-card__desc">{{ point.description }}</p>
            </div>
          </article>
        </div>
      </section>

      <!-- 每日行程 -->
      <section id="section-days" class="card">
        <div class="card__header">
          <span class="card__header-icon">📅</span>
          <h2 class="card__title">每日行程</h2>
        </div>
        <div class="day-list">
          <details
            v-for="day in itinerary.days"
            :key="day.day_index"
            class="day-card"
            :open="day.day_index === 1"
          >
            <summary class="day-card__summary">
              <div class="day-card__head-left">
                <span class="day-card__day-num">{{ day.day_index }}</span>
                <div>
                  <div class="day-card__title">第{{ day.day_index }}天</div>
                  <div class="day-card__theme">{{ day.theme || "未命名主题" }}</div>
                </div>
              </div>
              <span class="day-card__date">{{ formatShortDate(day.date) }}</span>
            </summary>
            <div class="day-card__body">
              <div class="day-card__item">
                <span class="day-card__item-icon">📍</span>
                <div>
                  <strong>主要景点</strong>
                  <p>{{ day.spots[0]?.name || "未安排" }}</p>
                </div>
              </div>
              <div class="day-card__item">
                <span class="day-card__item-icon">🏠</span>
                <div>
                  <strong>景点地址</strong>
                  <p>{{ day.spots[0]?.address || day.spots[0]?.location || "待补充" }}</p>
                </div>
              </div>
              <div class="day-card__item">
                <span class="day-card__item-icon">🍽️</span>
                <div>
                  <strong>餐饮建议</strong>
                  <p>{{ day.meals[0]?.name || "未安排" }}</p>
                </div>
              </div>
              <div class="day-card__item">
                <span class="day-card__item-icon">🛏️</span>
                <div>
                  <strong>住宿安排</strong>
                  <p>{{ day.hotel?.name || "未安排" }}</p>
                </div>
              </div>
              <div class="day-card__item">
                <span class="day-card__item-icon">🚗</span>
                <div>
                  <strong>交通信息</strong>
                  <p>
                    {{
                      day.transport[0]?.distance_km != null
                        ? `${day.transport[0].distance_km.toFixed(2)} km / ${day.transport[0].estimated_minutes ?? 0} 分钟`
                        : day.transport[0]?.duration || "待补充"
                    }}
                  </p>
                </div>
              </div>
              <div class="day-card__item">
                <span class="day-card__item-icon">📝</span>
                <div>
                  <strong>备注</strong>
                  <p>{{ day.notes[day.notes.length - 1] || "无" }}</p>
                </div>
              </div>
            </div>
          </details>
        </div>
      </section>
    </div>
  </section>

  <!-- 无结果时 -->
  <section v-else class="empty-state">
    <div class="empty-card">
      <div class="empty-card__icon">📭</div>
      <h2>还没有生成结果</h2>
      <p>先回到规划页生成一条行程，结果页就会开始展示真实数据。</p>
      <button class="empty-card__btn" @click="$emit('backHome')">← 前往规划页</button>
    </div>
  </section>
</template>

<style scoped>
/* ===== Layout ===== */
.result-page {
  display: grid;
  grid-template-columns: 220px 1fr;
  gap: 20px;
  align-items: start;
}

/* ===== Sidebar ===== */
.sidebar {
  position: sticky;
  top: 20px;
  padding: 20px;
  border-radius: 18px;
  background: #ffffff;
  box-shadow:
    0 1px 3px rgba(0, 0, 0, 0.04),
    0 4px 12px rgba(0, 0, 0, 0.04),
    0 8px 32px rgba(99, 102, 241, 0.06);
  border: 1px solid rgba(99, 102, 241, 0.06);
}

.sidebar__header {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 18px;
  padding-bottom: 14px;
  border-bottom: 1px solid #f1f5f9;
}

.sidebar__logo {
  font-size: 24px;
}

.sidebar__title {
  font-size: 15px;
  font-weight: 700;
  color: #1e293b;
}

.sidebar__nav {
  display: flex;
  flex-direction: column;
  gap: 4px;
  margin-bottom: 18px;
}

.sidebar__link {
  display: flex;
  align-items: center;
  gap: 10px;
  width: 100%;
  border: none;
  border-radius: 10px;
  padding: 10px 12px;
  background: transparent;
  color: #64748b;
  font-size: 13px;
  font-weight: 500;
  font-family: inherit;
  cursor: pointer;
  transition: all 0.2s ease;
  text-align: left;
}

.sidebar__link:hover {
  background: #f8fafc;
  color: #4338ca;
}

.sidebar__link--active {
  background: #eef2ff;
  color: #4338ca;
  font-weight: 600;
}

.sidebar__link-icon {
  font-size: 15px;
  width: 20px;
  text-align: center;
}

.sidebar__actions {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding-top: 14px;
  border-top: 1px solid #f1f5f9;
}

.sidebar-btn {
  width: 100%;
  border: none;
  border-radius: 10px;
  padding: 10px 14px;
  font-size: 13px;
  font-weight: 600;
  font-family: inherit;
  cursor: pointer;
  transition: all 0.2s ease;
  text-align: center;
}

.sidebar-btn:disabled {
  opacity: 0.6;
  cursor: wait;
}

.sidebar-btn--primary {
  background: linear-gradient(135deg, #6366f1 0%, #7c3aed 100%);
  color: #ffffff;
  box-shadow: 0 2px 8px rgba(99, 102, 241, 0.2);
}

.sidebar-btn--primary:hover:not(:disabled) {
  box-shadow: 0 4px 16px rgba(99, 102, 241, 0.3);
  transform: translateY(-1px);
}

.sidebar-btn--secondary {
  background: #f1f5f9;
  color: #475569;
}

.sidebar-btn--secondary:hover {
  background: #e2e8f0;
}

.sidebar-btn--ghost {
  background: transparent;
  color: #6366f1;
}

.sidebar-btn--ghost:hover {
  background: #eef2ff;
}

.sidebar-btn--outline {
  background: transparent;
  color: #3b82f6;
  border: 1.5px solid #bfdbfe;
}

.sidebar-btn--outline:hover:not(:disabled) {
  background: #eff6ff;
  border-color: #93c5fd;
}

.sidebar-btn--outline-green {
  background: transparent;
  color: #059669;
  border: 1.5px solid #a7f3d0;
}

.sidebar-btn--outline-green:hover:not(:disabled) {
  background: #ecfdf5;
  border-color: #6ee7b7;
}

/* ===== Main content ===== */
.result-main {
  display: flex;
  flex-direction: column;
  gap: 16px;
  min-width: 0;
}

/* ===== Card base ===== */
.card {
  padding: 24px;
  border-radius: 18px;
  background: #ffffff;
  box-shadow:
    0 1px 3px rgba(0, 0, 0, 0.04),
    0 4px 12px rgba(0, 0, 0, 0.04),
    0 8px 32px rgba(99, 102, 241, 0.06);
  border: 1px solid rgba(99, 102, 241, 0.06);
  transition: box-shadow 0.3s ease;
}

.card:hover {
  box-shadow:
    0 1px 3px rgba(0, 0, 0, 0.04),
    0 4px 16px rgba(0, 0, 0, 0.06),
    0 12px 40px rgba(99, 102, 241, 0.08);
}

.card--map {
  min-height: 380px;
}

.card--weather {
  min-height: 200px;
}

.card__header {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 20px;
  padding-bottom: 14px;
  border-bottom: 1px solid #f1f5f9;
}

.card__header-icon {
  font-size: 22px;
}

.card__title {
  margin: 0;
  font-size: 17px;
  font-weight: 700;
  color: #1e293b;
}

.card__subtitle {
  margin: 2px 0 0;
  font-size: 13px;
  color: #94a3b8;
}

.card__badge {
  margin-left: auto;
  padding: 4px 12px;
  border-radius: 999px;
  background: #f1f5f9;
  color: #64748b;
  font-size: 11px;
  font-weight: 600;
  font-family: "SF Mono", "Fira Code", monospace;
}

/* ===== Overview ===== */
.overview-summary {
  margin: 0 0 16px;
  color: #475569;
  line-height: 1.8;
  font-size: 15px;
}

.tips-box {
  padding: 18px 20px;
  border-radius: 14px;
  background: #fefce8;
  border: 1px solid #fef08a;
}

.tips-box__title {
  font-weight: 700;
  color: #854d0e;
  margin-bottom: 10px;
  font-size: 14px;
}

.tips-box__list {
  margin: 0;
  padding-left: 18px;
  display: flex;
  flex-direction: column;
  gap: 6px;
  color: #92400e;
  font-size: 14px;
  line-height: 1.7;
}

/* ===== Budget ===== */
.budget-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 12px;
  margin-bottom: 16px;
}

.budget-card {
  padding: 18px 14px;
  border-radius: 14px;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  text-align: center;
  transition: all 0.2s ease;
}

.budget-card:hover {
  border-color: var(--accent);
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.04);
  transform: translateY(-2px);
}

.budget-card__icon {
  font-size: 24px;
  margin-bottom: 6px;
}

.budget-card__label {
  font-size: 12px;
  color: #64748b;
  font-weight: 500;
  margin-bottom: 6px;
}

.budget-card__value {
  font-size: 22px;
  font-weight: 800;
  color: #1e293b;
}

.budget-total-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 20px;
  border-radius: 14px;
  background: linear-gradient(135deg, #6366f1 0%, #8b5cf6 100%);
  color: #ffffff;
  font-size: 15px;
  font-weight: 500;
}

.budget-total-bar strong {
  font-size: 30px;
  font-weight: 800;
}

/* ===== Weather ===== */
.weather-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
  gap: 10px;
}

.weather-card {
  padding: 16px;
  border-radius: 14px;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  transition: all 0.2s ease;
}

.weather-card:hover {
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.04);
  border-color: #93c5fd;
}

.weather-card__date {
  font-weight: 700;
  color: #334155;
  font-size: 14px;
  margin-bottom: 8px;
}

.weather-card__temps {
  display: flex;
  align-items: baseline;
  gap: 6px;
  margin-bottom: 10px;
}

.weather-card__high {
  font-size: 26px;
  font-weight: 800;
  color: #f59e0b;
}

.weather-card__sep {
  color: #cbd5e1;
  font-size: 16px;
}

.weather-card__low {
  font-size: 20px;
  font-weight: 700;
  color: #3b82f6;
}

.weather-card__desc {
  display: flex;
  flex-direction: column;
  gap: 4px;
  font-size: 13px;
  color: #64748b;
}

.status-text {
  color: #94a3b8;
  font-size: 14px;
  padding: 8px 0;
}

.status-text--error {
  color: #ef4444;
}

/* ===== Edit panel ===== */
.edit-panel {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.edit-panel__row {
  display: grid;
  grid-template-columns: 1fr auto;
  gap: 12px;
  align-items: end;
}

.edit-field {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.edit-field__label {
  font-size: 13px;
  font-weight: 600;
  color: #475569;
}

.modern-select-native {
  width: 100%;
  min-height: 42px;
  padding: 0 14px;
  border: 1.5px solid #e2e8f0;
  border-radius: 12px;
  background: #fafbfc;
  color: #334155;
  font-size: 13px;
  font-family: inherit;
  outline: none;
  cursor: pointer;
  transition: all 0.2s ease;
}

.modern-select-native:focus {
  border-color: #6366f1;
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.1);
}

.modern-textarea-native {
  width: 100%;
  border: 1.5px solid #e2e8f0;
  border-radius: 12px;
  padding: 12px 14px;
  background: #fafbfc;
  color: #334155;
  font-size: 14px;
  font-family: inherit;
  line-height: 1.7;
  resize: vertical;
  min-height: 88px;
  outline: none;
  transition: all 0.2s ease;
}

.modern-textarea-native:focus {
  border-color: #6366f1;
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.1);
}

.edit-btn {
  min-height: 42px;
  padding: 0 24px;
  border: none;
  border-radius: 12px;
  background: linear-gradient(135deg, #6366f1 0%, #8b5cf6 100%);
  color: #ffffff;
  font-size: 14px;
  font-weight: 700;
  font-family: inherit;
  cursor: pointer;
  white-space: nowrap;
  transition: all 0.2s ease;
  box-shadow: 0 2px 8px rgba(99, 102, 241, 0.2);
}

.edit-btn:hover:not(:disabled) {
  box-shadow: 0 4px 16px rgba(99, 102, 241, 0.3);
  transform: translateY(-1px);
}

.edit-btn:disabled {
  opacity: 0.7;
  cursor: wait;
}

/* ===== Day budget ===== */
.day-budget-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 12px;
}

.day-budget-card {
  border-radius: 14px;
  overflow: hidden;
  border: 1px solid #e2e8f0;
  background: #fafbfc;
  transition: all 0.2s ease;
}

.day-budget-card:hover {
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.04);
  border-color: #c4b5fd;
}

.day-budget-card__head {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 14px;
  background: #eef2ff;
  font-size: 14px;
  color: #3730a3;
}

.day-budget-card__theme {
  font-size: 12px;
  color: #6366f1;
  font-weight: 500;
}

.day-budget-card__rows {
  padding: 12px 14px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.day-budget-row {
  display: flex;
  justify-content: space-between;
  font-size: 13px;
  color: #475569;
}

.day-budget-row--total {
  padding-top: 8px;
  border-top: 1px solid #e2e8f0;
  color: #4338ca;
  font-weight: 700;
}

/* ===== Point cards ===== */
.point-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
  gap: 14px;
}

.point-card {
  border-radius: 14px;
  overflow: hidden;
  border: 1px solid #e2e8f0;
  background: #fafbfc;
  transition: all 0.2s ease;
}

.point-card:hover {
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.04);
  border-color: #c4b5fd;
}

.point-card__head {
  display: flex;
  justify-content: space-between;
  gap: 8px;
  padding: 12px 14px;
  background: #eef2ff;
  font-size: 13px;
  font-weight: 600;
  color: #3730a3;
}

.point-card__date {
  font-size: 12px;
  color: #6366f1;
  font-weight: 500;
}

.point-card__body {
  padding: 14px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.point-card__img {
  height: 150px;
  border-radius: 10px;
  background-size: cover;
  background-position: center;
  background-color: #f1f5f9;
}

.point-card__img--empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 6px;
  color: #94a3b8;
  font-size: 13px;
  font-weight: 500;
  background: linear-gradient(135deg, #f1f5f9, #f8fafc);
}

.point-card__info {
  font-size: 13px;
  color: #475569;
  line-height: 1.6;
}

.point-card__desc {
  margin: 4px 0 0;
  padding-top: 8px;
  border-top: 1px solid #e2e8f0;
  font-size: 13px;
  color: #64748b;
  line-height: 1.7;
}

/* ===== Day cards (accordion) ===== */
.day-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.day-card {
  border-radius: 14px;
  border: 1px solid #e2e8f0;
  background: #fafbfc;
  overflow: hidden;
  transition: all 0.2s ease;
}

.day-card:hover {
  border-color: #c4b5fd;
}

.day-card__summary {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 14px 16px;
  cursor: pointer;
  list-style: none;
  background: #fafbfc;
  transition: background 0.2s ease;
}

.day-card__summary::-webkit-details-marker {
  display: none;
}

.day-card__summary:hover {
  background: #f1f5f9;
}

.day-card__head-left {
  display: flex;
  align-items: center;
  gap: 12px;
}

.day-card__day-num {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  border-radius: 10px;
  background: linear-gradient(135deg, #6366f1, #8b5cf6);
  color: #ffffff;
  font-size: 15px;
  font-weight: 700;
}

.day-card__title {
  font-weight: 700;
  color: #1e293b;
  font-size: 15px;
}

.day-card__theme {
  font-size: 12px;
  color: #6366f1;
  font-weight: 500;
  margin-top: 2px;
}

.day-card__date {
  font-size: 12px;
  color: #94a3b8;
  font-weight: 500;
}

.day-card[open] .day-card__summary {
  background: #eef2ff;
  border-bottom: 1px solid #e2e8f0;
}

.day-card__body {
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding: 16px;
}

.day-card__item {
  display: flex;
  gap: 12px;
  color: #475569;
  font-size: 14px;
  line-height: 1.6;
}

.day-card__item-icon {
  font-size: 16px;
  flex-shrink: 0;
  margin-top: 1px;
}

.day-card__item strong {
  display: block;
  font-size: 12px;
  color: #94a3b8;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  margin-bottom: 2px;
}

.day-card__item p {
  margin: 0;
  color: #334155;
}

/* ===== Empty state ===== */
.empty-state {
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 400px;
}

.empty-card {
  text-align: center;
  padding: 48px 40px;
  border-radius: 20px;
  background: #ffffff;
  box-shadow:
    0 1px 3px rgba(0, 0, 0, 0.04),
    0 8px 32px rgba(99, 102, 241, 0.06);
  border: 1px solid rgba(99, 102, 241, 0.06);
  max-width: 460px;
}

.empty-card__icon {
  font-size: 56px;
  margin-bottom: 16px;
}

.empty-card h2 {
  margin: 0 0 8px;
  font-size: 22px;
  color: #1e293b;
}

.empty-card p {
  margin: 0 0 20px;
  color: #94a3b8;
  line-height: 1.7;
  font-size: 14px;
}

.empty-card__btn {
  border: none;
  border-radius: 12px;
  padding: 12px 28px;
  background: linear-gradient(135deg, #6366f1 0%, #8b5cf6 100%);
  color: #ffffff;
  font-size: 15px;
  font-weight: 600;
  font-family: inherit;
  cursor: pointer;
  transition: all 0.2s ease;
  box-shadow: 0 2px 8px rgba(99, 102, 241, 0.2);
}

.empty-card__btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 16px rgba(99, 102, 241, 0.3);
}

/* ===== Responsive ===== */
@media (max-width: 960px) {
  .result-page {
    grid-template-columns: 1fr;
  }

  .sidebar {
    position: static;
  }

  .sidebar__nav {
    flex-direction: row;
    flex-wrap: wrap;
    gap: 6px;
  }

  .sidebar__link {
    width: auto;
    font-size: 12px;
    padding: 6px 10px;
  }

  .sidebar__actions {
    flex-direction: row;
    flex-wrap: wrap;
  }

  .sidebar-btn {
    width: auto;
    flex: 1;
    min-width: 80px;
    font-size: 12px;
    padding: 8px 10px;
  }

  .budget-grid {
    grid-template-columns: repeat(2, 1fr);
  }

  .day-budget-grid {
    grid-template-columns: 1fr;
  }

  .edit-panel__row {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 600px) {
  .budget-grid {
    grid-template-columns: 1fr 1fr;
  }

  .weather-grid {
    grid-template-columns: 1fr 1fr;
  }
}
</style>
