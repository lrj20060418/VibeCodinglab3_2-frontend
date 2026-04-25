<script setup>
import { computed, onMounted, reactive, ref } from 'vue'
import { createPlan, getPlan, listPlans, updatePlan } from '../api/plans'

const LAST_OPEN_PLAN_ID_KEY = 'lab3.lastOpenPlanId'

const plans = ref([])
const selectedPlanId = ref(null)

const listLoading = ref(false)
const planLoading = ref(false)
const saving = ref(false)

const listError = ref('')
const planError = ref('')
const saveError = ref('')
const saveSuccess = ref(false)

const form = reactive({
  title: '',
  date: '',
  budget: '',
  people_count: '',
  preferences: '',
})

const isEditingExisting = computed(() => Boolean(selectedPlanId.value))

function todayISO() {
  const d = new Date()
  const yyyy = d.getFullYear()
  const mm = String(d.getMonth() + 1).padStart(2, '0')
  const dd = String(d.getDate()).padStart(2, '0')
  return `${yyyy}-${mm}-${dd}`
}

function resetForm() {
  form.title = ''
  form.date = todayISO()
  form.budget = ''
  form.people_count = ''
  form.preferences = ''
  planError.value = ''
  saveError.value = ''
  saveSuccess.value = false
}

function normalizePayload() {
  const payload = {
    title: form.title?.trim() || null,
    date: form.date,
    budget: form.budget === '' ? null : Number(form.budget),
    people_count: form.people_count === '' ? null : Number(form.people_count),
    preferences: form.preferences?.trim() || null,
  }

  if (payload.title === null) delete payload.title
  if (payload.budget === null) delete payload.budget
  if (payload.people_count === null) delete payload.people_count
  if (payload.preferences === null) delete payload.preferences

  return payload
}

async function refreshPlans() {
  listLoading.value = true
  listError.value = ''
  try {
    plans.value = await listPlans()
  } catch (e) {
    listError.value = e?.message || '加载规划列表失败'
    plans.value = []
  } finally {
    listLoading.value = false
  }
}

async function openPlan(planId) {
  selectedPlanId.value = planId
  localStorage.setItem(LAST_OPEN_PLAN_ID_KEY, planId)

  planLoading.value = true
  planError.value = ''
  saveError.value = ''
  saveSuccess.value = false
  try {
    const p = await getPlan(planId)
    form.title = p.title || ''
    form.date = p.date || todayISO()
    form.budget = p.budget ?? ''
    form.people_count = p.people_count ?? ''
    form.preferences = p.preferences || ''
  } catch (e) {
    planError.value = e?.message || '加载规划失败'
  } finally {
    planLoading.value = false
  }
}

function newPlan() {
  selectedPlanId.value = null
  localStorage.removeItem(LAST_OPEN_PLAN_ID_KEY)
  resetForm()
}

async function savePlan() {
  saveSuccess.value = false
  saveError.value = ''
  planError.value = ''

  if (!form.date) {
    saveError.value = '请先填写日期'
    return
  }

  const payload = normalizePayload()

  saving.value = true
  try {
    let saved
    if (isEditingExisting.value) {
      saved = await updatePlan(selectedPlanId.value, payload)
    } else {
      saved = await createPlan(payload)
      selectedPlanId.value = saved.id
      localStorage.setItem(LAST_OPEN_PLAN_ID_KEY, saved.id)
    }

    saveSuccess.value = true
    await refreshPlans()
  } catch (e) {
    saveError.value = e?.message || '保存失败'
  } finally {
    saving.value = false
  }
}

const emptyList = computed(() => !listLoading.value && !listError.value && plans.value.length === 0)
const nextStepText = computed(() => {
  if (!selectedPlanId.value) return '下一步：保存规划后，再进入 V4（地点管理：地图选点加入规划）。'
  return '下一步：进入 V4（地点管理：地图选点加入规划）。'
})

onMounted(async () => {
  resetForm()
  await refreshPlans()

  const lastId = localStorage.getItem(LAST_OPEN_PLAN_ID_KEY)
  if (lastId) {
    await openPlan(lastId)
  }
})
</script>

<template>
  <div class="app-shell">
    <header class="topbar">
      <div class="brand">
        <div class="title">智能出行规划器</div>
        <div class="subtitle">V3：规划表单 + 打开/保存</div>
      </div>
      <div class="top-actions">
        <button class="btn" type="button" @click="newPlan">新建规划</button>
        <button class="btn primary" type="button" :disabled="saving || planLoading" @click="savePlan">
          {{ saving ? '保存中…' : '保存' }}
        </button>
      </div>
    </header>

    <main class="main">
      <aside class="sidebar">
        <div class="panel-title">我的规划</div>

        <div v-if="listLoading" class="state">加载中…</div>
        <div v-else-if="listError" class="state error">
          <div>加载失败：{{ listError }}</div>
          <button class="btn small" type="button" @click="refreshPlans">重试</button>
        </div>
        <div v-else-if="emptyList" class="state">
          <div>还没有任何规划。</div>
          <div class="muted">点击“新建规划”开始。</div>
        </div>

        <ul v-else class="plan-list">
          <li v-for="p in plans" :key="p.id">
            <button
              class="plan-item"
              type="button"
              :aria-current="p.id === selectedPlanId ? 'true' : 'false'"
              :data-active="p.id === selectedPlanId ? 'true' : 'false'"
              @click="openPlan(p.id)"
            >
              <div class="plan-name">
                {{ p.title || '未命名规划' }}
              </div>
              <div class="plan-meta">
                <span>{{ p.date }}</span>
                <span v-if="p.budget != null">¥{{ p.budget }}</span>
                <span v-if="p.people_count != null">{{ p.people_count }} 人</span>
              </div>
            </button>
          </li>
        </ul>
      </aside>

      <section class="content">
        <div class="card">
          <div class="card-header">
            <div>
              <div class="card-title">
                {{ isEditingExisting ? '编辑规划' : '新建规划' }}
              </div>
              <div class="card-subtitle">
                {{ nextStepText }}
              </div>
            </div>

            <div class="badge" v-if="selectedPlanId">
              已打开：{{ selectedPlanId.slice(0, 8) }}
            </div>
          </div>

          <div v-if="planLoading" class="state">加载中…</div>
          <div v-else-if="planError" class="state error">
            <div>加载失败：{{ planError }}</div>
            <button v-if="selectedPlanId" class="btn small" type="button" @click="openPlan(selectedPlanId)">
              重试
            </button>
          </div>

          <form v-else class="form" @submit.prevent="savePlan">
            <div class="grid">
              <label class="field">
                <span class="label">日期（必填）</span>
                <input class="input" type="date" v-model="form.date" required />
              </label>

              <label class="field">
                <span class="label">预算（元）</span>
                <input class="input" type="number" min="0" step="1" inputmode="numeric" v-model="form.budget" />
              </label>

              <label class="field">
                <span class="label">人数</span>
                <input class="input" type="number" min="1" step="1" inputmode="numeric" v-model="form.people_count" />
              </label>

              <label class="field">
                <span class="label">标题（可选）</span>
                <input class="input" type="text" maxlength="120" v-model="form.title" placeholder="例如：周末出游" />
              </label>
            </div>

            <label class="field">
              <span class="label">偏好</span>
              <textarea class="textarea" rows="5" maxlength="2000" v-model="form.preferences" placeholder="例如：喜欢自然景点、尽量少走路、想吃本地小吃…"></textarea>
            </label>

            <div class="form-actions">
              <button class="btn primary" type="submit" :disabled="saving">
                {{ saving ? '保存中…' : '保存' }}
              </button>
              <button class="btn" type="button" :disabled="saving || planLoading" @click="refreshPlans">
                刷新列表
              </button>
            </div>

            <div v-if="saveSuccess" class="notice success">
              已保存。刷新页面后会自动打开你最后编辑的规划。
            </div>
            <div v-else-if="saveError" class="notice error">
              保存失败：{{ saveError }}
            </div>
          </form>
        </div>
      </section>
    </main>
  </div>
</template>

<style scoped>
.app-shell {
  min-height: 100svh;
  display: flex;
  flex-direction: column;
}

.topbar {
  position: sticky;
  top: 0;
  z-index: 10;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 16px 20px;
  border-bottom: 1px solid var(--border);
  background: color-mix(in srgb, var(--bg) 92%, transparent);
  backdrop-filter: blur(10px);
}

.brand .title {
  font-family: var(--heading);
  color: var(--text-h);
  font-size: 18px;
  letter-spacing: -0.2px;
}
.brand .subtitle {
  margin-top: 2px;
  font-size: 13px;
  color: var(--text);
}

.top-actions {
  display: flex;
  gap: 10px;
}

.main {
  width: 1126px;
  max-width: 100%;
  margin: 0 auto;
  flex: 1 1 auto;
  display: grid;
  grid-template-columns: 320px 1fr;
  border-inline: 1px solid var(--border);
  box-sizing: border-box;
}

.sidebar {
  border-right: 1px solid var(--border);
  padding: 18px;
  text-align: left;
}

.content {
  padding: 18px;
}

.panel-title {
  font-family: var(--heading);
  color: var(--text-h);
  font-size: 14px;
  letter-spacing: 0.2px;
  margin-bottom: 12px;
}

.plan-list {
  list-style: none;
  padding: 0;
  margin: 0;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.plan-item {
  width: 100%;
  border-radius: 10px;
  border: 1px solid var(--border);
  background: transparent;
  color: inherit;
  padding: 12px;
  text-align: left;
  cursor: pointer;
  transition: box-shadow 0.2s, border-color 0.2s;
}

.plan-item[data-active='true'] {
  border-color: var(--accent-border);
  box-shadow: var(--shadow);
}

.plan-name {
  color: var(--text-h);
  font-weight: 500;
  margin-bottom: 6px;
}

.plan-meta {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  font-size: 12px;
  color: var(--text);
}

.card {
  border: 1px solid var(--border);
  border-radius: 12px;
  padding: 16px;
  text-align: left;
}

.card-header {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 12px;
  padding-bottom: 12px;
  border-bottom: 1px solid var(--border);
  margin-bottom: 12px;
}

.card-title {
  font-family: var(--heading);
  color: var(--text-h);
  font-size: 18px;
  margin-bottom: 4px;
}
.card-subtitle {
  font-size: 13px;
  color: var(--text);
}

.badge {
  font-family: var(--mono);
  font-size: 12px;
  padding: 6px 10px;
  border-radius: 999px;
  border: 1px solid var(--border);
  background: var(--social-bg);
  color: var(--text-h);
  white-space: nowrap;
}

.form {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.label {
  font-size: 12px;
  color: var(--text);
}

.input,
.textarea {
  border: 1px solid var(--border);
  border-radius: 10px;
  padding: 10px 12px;
  font: inherit;
  color: var(--text-h);
  background: transparent;
}

.textarea {
  resize: vertical;
  min-height: 110px;
}

.btn {
  border-radius: 10px;
  border: 1px solid var(--border);
  background: transparent;
  color: var(--text-h);
  padding: 10px 12px;
  cursor: pointer;
  transition: box-shadow 0.2s, border-color 0.2s, background 0.2s;
}

.btn:hover {
  box-shadow: var(--shadow);
}

.btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.btn.primary {
  border-color: var(--accent-border);
  background: var(--accent-bg);
  color: var(--accent);
}

.btn.small {
  padding: 8px 10px;
  font-size: 13px;
}

.state {
  border: 1px dashed var(--border);
  border-radius: 12px;
  padding: 12px;
  color: var(--text);
}

.state.error {
  border-style: solid;
  border-color: color-mix(in srgb, #ef4444 45%, var(--border));
}

.notice {
  border-radius: 12px;
  padding: 10px 12px;
  font-size: 13px;
}

.notice.success {
  border: 1px solid color-mix(in srgb, #22c55e 35%, var(--border));
  background: color-mix(in srgb, #22c55e 10%, transparent);
  color: var(--text-h);
}

.notice.error {
  border: 1px solid color-mix(in srgb, #ef4444 35%, var(--border));
  background: color-mix(in srgb, #ef4444 10%, transparent);
  color: var(--text-h);
}

.form-actions {
  display: flex;
  gap: 10px;
  align-items: center;
  justify-content: flex-start;
}

.muted {
  margin-top: 6px;
  font-size: 12px;
  color: var(--text);
}

@media (max-width: 980px) {
  .main {
    grid-template-columns: 1fr;
  }
  .sidebar {
    border-right: none;
    border-bottom: 1px solid var(--border);
  }
  .grid {
    grid-template-columns: 1fr;
  }
}
</style>

