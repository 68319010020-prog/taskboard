<template>
  <div class="container">
    <header class="hero">
      <div>
        <p class="eyebrow">TaskBoard</p>
        <h1>จัดการงานอย่างมืออาชีพ พร้อม UI สวยขึ้น</h1>
      </div>
      <div class="stats">
        <div class="stat-card todo">
          <strong>{{ counts.todo }}</strong>
          <span>Todo</span>
        </div>
        <div class="stat-card inprogress">
          <strong>{{ counts.inprogress }}</strong>
          <span>In Progress</span>
        </div>
        <div class="stat-card done">
          <strong>{{ counts.done }}</strong>
          <span>Done</span>
        </div>
      </div>
    </header>

    <section class="card form-card">
      <div class="card-header">
        <div>
          <p class="subtitle">เพิ่มงานใหม่</p>
          <h2>เพิ่มรายการงานง่าย ๆ</h2>
        </div>
      </div>

      <div class="form-grid">
        <input
          v-model="newTitle"
          placeholder="ชื่องาน..."
          @keyup.enter="addTask"
        />
        <textarea
          v-model="newDescription"
          placeholder="รายละเอียดเพิ่มเติม (ไม่บังคับ)"
          rows="3"
        ></textarea>
        <select v-model="newPriority">
          <option value="low">Low</option>
          <option value="medium">Medium</option>
          <option value="high">High</option>
        </select>
        <button class="primary-btn" @click="addTask">+ เพิ่มงาน</button>
      </div>
    </section>

    <section class="card filter-card">
      <div class="filter-group">
        <label>สถานะ</label>
        <select v-model="filterStatus" @change="loadTasks">
          <option value="">ทั้งหมด</option>
          <option value="todo">Todo</option>
          <option value="inprogress">In Progress</option>
          <option value="done">Done</option>
        </select>
      </div>
      <div class="filter-group">
        <label>ความสำคัญ</label>
        <select v-model="filterPriority" @change="loadTasks">
          <option value="">ทั้งหมด</option>
          <option value="low">Low</option>
          <option value="medium">Medium</option>
          <option value="high">High</option>
        </select>
      </div>
      <button class="secondary-btn" @click="resetFilters">ล้างตัวกรอง</button>
    </section>

    <div v-if="error" class="notice error">{{ error }}</div>
    <div v-if="loading" class="notice loading">กำลังโหลดรายการงาน...</div>

    <section v-else class="task-list">
      <article
        v-for="task in tasks"
        :key="task.id"
        class="task-card"
        :class="[task.priority, task.status]"
      >
        <div class="task-head">
          <div>
            <span class="status-chip">{{ statusLabel(task.status) }}</span>
            <span class="priority-chip">{{ task.priority }}</span>
            <h3>{{ task.title }}</h3>
          </div>
          <div class="task-meta">
            <small>สร้าง {{ formatDate(task.created_at) }}</small>
            <small v-if="task.updated_at">แก้ไข {{ formatDate(task.updated_at) }}</small>
          </div>
        </div>

        <p class="task-description">
          {{ task.description || 'ไม่มีรายละเอียดเพิ่มเติม' }}
        </p>

        <div v-if="editTaskId === task.id" class="edit-panel">
          <input
            v-model="editTitle"
            placeholder="ชื่องาน"
          />
          <textarea v-model="editDescription" rows="3"></textarea>
          <div class="edit-grid">
            <select v-model="editPriority">
              <option value="low">Low</option>
              <option value="medium">Medium</option>
              <option value="high">High</option>
            </select>
            <select v-model="editStatus">
              <option value="todo">Todo</option>
              <option value="inprogress">In Progress</option>
              <option value="done">Done</option>
            </select>
            <button class="primary-btn small" @click="saveTask(task)">บันทึก</button>
            <button class="secondary-btn small" @click="cancelEdit">ยกเลิก</button>
          </div>
        </div>

        <div class="task-actions">
          <div class="action-row">
            <label>สถานะ</label>
            <select :value="task.status" @change="changeStatus(task, $event.target.value)">
              <option value="todo">Todo</option>
              <option value="inprogress">In Progress</option>
              <option value="done">Done</option>
            </select>
          </div>
          <div class="action-row">
            <label>จัดการ</label>
            <div class="button-row">
              <button class="secondary-btn small" @click="startEdit(task)">แก้ไข</button>
              <button class="danger-btn small" @click="deleteTask(task.id)">ลบ</button>
            </div>
          </div>
        </div>
      </article>

      <div v-if="tasks.length === 0" class="empty-state">
        ไม่มีงานในขณะนี้ ลองเพิ่มงานหรือปรับตัวกรองดู
      </div>
    </section>
  </div>
</template>

<script setup>
import { computed, ref, onMounted } from 'vue'

const API = import.meta.env.VITE_API_URL || 'http://localhost:3002'

const tasks = ref([])
const loading = ref(false)
const error = ref('')

const newTitle = ref('')
const newDescription = ref('')
const newPriority = ref('medium')

const filterStatus = ref('')
const filterPriority = ref('')

const editTaskId = ref(null)
const editTitle = ref('')
const editDescription = ref('')
const editPriority = ref('medium')
const editStatus = ref('todo')

const counts = computed(() => ({
  todo: tasks.value.filter(task => task.status === 'todo').length,
  inprogress: tasks.value.filter(task => task.status === 'inprogress').length,
  done: tasks.value.filter(task => task.status === 'done').length,
}))

function buildQuery() {
  const params = new URLSearchParams()
  if (filterStatus.value) params.set('status', filterStatus.value)
  if (filterPriority.value) params.set('priority', filterPriority.value)
  return params.toString() ? `?${params.toString()}` : ''
}

function statusLabel(status) {
  return status === 'inprogress' ? 'In Progress' : status.charAt(0).toUpperCase() + status.slice(1)
}

function formatDate(value) {
  if (!value) return '-'
  return new Date(value).toLocaleString('th-TH', {
    day: '2-digit',
    month: 'short',
    year: 'numeric',
    hour: '2-digit',
    minute: '2-digit',
    hour12: false,
  })
}

async function loadTasks() {
  loading.value = true
  error.value = ''

  try {
    const res = await fetch(`${API}/api/tasks${buildQuery()}`)
    if (!res.ok) throw new Error('ไม่สามารถโหลดข้อมูลงานได้')
    tasks.value = await res.json()
  } catch (err) {
    error.value = err.message || 'เกิดข้อผิดพลาดระหว่างโหลดงาน'
  } finally {
    loading.value = false
  }
}

async function addTask() {
  if (!newTitle.value.trim()) {
    error.value = 'กรุณากรอกชื่องานก่อน'
    return
  }

  loading.value = true
  error.value = ''

  try {
    const res = await fetch(`${API}/api/tasks`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        title: newTitle.value.trim(),
        description: newDescription.value.trim(),
        priority: newPriority.value,
      }),
    })
    const data = await res.json()
    if (!res.ok) throw new Error(data.error || 'เพิ่มงานไม่สำเร็จ')
    newTitle.value = ''
    newDescription.value = ''
    newPriority.value = 'medium'
    await loadTasks()
  } catch (err) {
    error.value = err.message || 'เกิดข้อผิดพลาดขณะเพิ่มงาน'
  } finally {
    loading.value = false
  }
}

function startEdit(task) {
  editTaskId.value = task.id
  editTitle.value = task.title || ''
  editDescription.value = task.description || ''
  editPriority.value = task.priority
  editStatus.value = task.status
}

function cancelEdit() {
  editTaskId.value = null
  editDescription.value = ''
}

async function saveTask(task) {
  loading.value = true
  error.value = ''

  try {
    const res = await fetch(`${API}/api/tasks/${task.id}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        title: editTitle.value.trim() || task.title,
        description: editDescription.value.trim(),
        status: editStatus.value,
        priority: editPriority.value,
      }),
    })
    const data = await res.json()
    if (!res.ok) throw new Error(data.error || 'อัปเดตงานไม่สำเร็จ')
    editTaskId.value = null
    await loadTasks()
  } catch (err) {
    error.value = err.message || 'เกิดข้อผิดพลาดขณะบันทึกงาน'
  } finally {
    loading.value = false
  }
}

async function changeStatus(task, status) {
  loading.value = true
  error.value = ''

  try {
    const res = await fetch(`${API}/api/tasks/${task.id}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        title: task.title,
        description: task.description || '',
        priority: task.priority,
        status,
      }),
    })
    const data = await res.json()
    if (!res.ok) throw new Error(data.error || 'เปลี่ยนสถานะไม่สำเร็จ')
    await loadTasks()
  } catch (err) {
    error.value = err.message || 'เกิดข้อผิดพลาดขณะเปลี่ยนสถานะ'
  } finally {
    loading.value = false
  }
}

async function deleteTask(id) {
  if (!confirm('ยืนยันลบงานนี้?')) return

  loading.value = true
  error.value = ''

  try {
    const res = await fetch(`${API}/api/tasks/${id}`, { method: 'DELETE' })
    const data = await res.json()
    if (!res.ok) throw new Error(data.error || 'ลบงานไม่สำเร็จ')
    await loadTasks()
  } catch (err) {
    error.value = err.message || 'เกิดข้อผิดพลาดขณะลบงาน'
  } finally {
    loading.value = false
  }
}

function resetFilters() {
  filterStatus.value = ''
  filterPriority.value = ''
  loadTasks()
}

onMounted(loadTasks)
</script>

<style scoped>
.container {
  max-width: 960px;
  margin: 2.5rem auto;
  padding: 0 1.2rem 3rem;
  font-family: 'Sarabun', system-ui, sans-serif;
  color: #1f2937;
}

.hero {
  display: grid;
  gap: 1.25rem;
  grid-template-columns: minmax(0, 1fr) auto;
  align-items: center;
  margin-bottom: 1.5rem;
}

.eyebrow {
  margin: 0 0 0.5rem;
  font-size: 0.85rem;
  font-weight: 800;
  color: #2563eb;
  letter-spacing: 0.18em;
  text-transform: uppercase;
}

h1 {
  margin: 0;
  font-size: clamp(2rem, 2.5vw, 2.5rem);
  line-height: 1.05;
}

.stats {
  display: flex;
  flex-wrap: wrap;
  gap: 0.85rem;
}

.stat-card {
  min-width: 115px;
  border-radius: 18px;
  padding: 1rem 1.1rem;
  color: #fff;
  display: grid;
  gap: 0.35rem;
}

.stat-card strong {
  font-size: 1.35rem;
}

.stat-card.todo { background: linear-gradient(135deg, #2563eb, #3b82f6); }
.stat-card.inprogress { background: linear-gradient(135deg, #f59e0b, #fbbf24); }
.stat-card.done { background: linear-gradient(135deg, #16a34a, #4ade80); }

.card {
  background: #ffffffee;
  border: 1px solid #e5e7eb;
  border-radius: 24px;
  box-shadow: 0 24px 60px -40px rgba(15, 23, 42, 0.35);
  padding: 1.4rem;
  margin-bottom: 1rem;
}

.card-header {
  border-bottom: 1px solid #e5e7eb;
  padding-bottom: 1rem;
  margin-bottom: 1rem;
}

.subtitle {
  margin: 0 0 0.25rem;
  font-size: 0.9rem;
  color: #475569;
  font-weight: 700;
}

h2 {
  margin: 0;
  font-size: 1.4rem;
}

.form-grid,
.filter-card,
.edit-grid {
  display: grid;
  gap: 0.95rem;
}

.form-grid input,
.form-grid textarea,
.form-grid select,
.task-actions select,
.edit-panel textarea {
  width: 100%;
  border-radius: 16px;
  border: 1px solid #d1d5db;
  padding: 1rem 1rem;
  font-size: 0.96rem;
  color: #1f2937;
  background: #f8fafc;
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.form-grid input:focus,
.form-grid textarea:focus,
.form-grid select:focus,
.task-actions select:focus,
.edit-panel textarea:focus {
  outline: none;
  border-color: #2563eb;
  box-shadow: 0 0 0 4px rgba(37, 99, 235, 0.12);
}

.form-grid textarea,
.edit-panel textarea {
  resize: vertical;
  min-height: 110px;
}

.primary-btn,
.secondary-btn,
.danger-btn {
  border: none;
  border-radius: 16px;
  font-weight: 700;
  cursor: pointer;
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.primary-btn {
  background: linear-gradient(135deg, #2563eb, #3b82f6);
  color: #fff;
  padding: 1rem 1.1rem;
}

.secondary-btn {
  background: #f8fafc;
  color: #334155;
  border: 1px solid #cbd5e1;
  padding: 1rem 1.1rem;
}

.danger-btn {
  background: #fee2e2;
  color: #991b1b;
  padding: 0.95rem 1rem;
}

.primary-btn:hover,
.secondary-btn:hover,
.danger-btn:hover {
  transform: translateY(-1px);
}

.filter-card {
  grid-template-columns: repeat(auto-fit, minmax(170px, 1fr));
  align-items: end;
}

.filter-group {
  display: grid;
  gap: 0.55rem;
}

.filter-group label {
  font-size: 0.88rem;
  font-weight: 700;
  color: #475569;
}

.notice {
  border-radius: 20px;
  padding: 1rem 1.2rem;
  margin-bottom: 1rem;
  font-weight: 600;
}

.notice.loading {
  background: #eff6ff;
  color: #1d4ed8;
}

.notice.error {
  background: #fee2e2;
  color: #991b1b;
}

.task-list {
  display: grid;
  gap: 1rem;
}

.task-card {
  border-left: 6px solid transparent;
  background: #fff;
  padding: 1.25rem;
}

.task-card.low { border-color: #10b981; }
.task-card.medium { border-color: #f59e0b; }
.task-card.high { border-color: #ef4444; }

.task-card.todo { background: #f8fafc; }
.task-card.inprogress { background: #fff7ed; }
.task-card.done { background: #ecfdf5; }

.task-head {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
  flex-wrap: wrap;
  align-items: flex-start;
  margin-bottom: 1rem;
}

.status-chip,
.priority-chip {
  display: inline-flex;
  align-items: center;
  padding: 0.4rem 0.85rem;
  border-radius: 999px;
  font-size: 0.8rem;
  font-weight: 700;
  text-transform: uppercase;
  margin-right: 0.55rem;
}

.status-chip { background: rgba(37, 99, 235, 0.12); color: #1d4ed8; }
.priority-chip { background: rgba(16, 185, 129, 0.12); color: #047857; }

.task-meta {
  display: grid;
  gap: 0.25rem;
  text-align: right;
  color: #475569;
  font-size: 0.85rem;
}

.task-description {
  margin: 0 0 1rem;
  color: #334155;
  line-height: 1.75;
}

.edit-panel {
  display: grid;
  gap: 0.9rem;
  margin-bottom: 1rem;
}

.edit-grid {
  grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
}

.task-actions {
  display: grid;
  gap: 1rem;
}

.action-row {
  display: grid;
  gap: 0.5rem;
}

.action-row label {
  font-size: 0.85rem;
  font-weight: 700;
  color: #475569;
}

.button-row {
  display: flex;
  flex-wrap: wrap;
  gap: 0.85rem;
}

.small {
  min-width: 130px;
  padding: 0.9rem 1rem;
}

.empty-state {
  border: 1px dashed #cbd5e1;
  padding: 1.8rem;
  border-radius: 22px;
  text-align: center;
  color: #475569;
  background: #f8fafc;
}

@media (max-width: 760px) {
  .hero {
    grid-template-columns: 1fr;
  }

  .filter-card,
  .edit-grid {
    grid-template-columns: 1fr;
  }
}
</style>