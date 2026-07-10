<script setup>
import { computed, reactive, ref, watch } from 'vue'

const STORAGE_KEY = 'tolist.tasks.v1'

const priorityOptions = [
  { label: '高', value: 'high' },
  { label: '普通', value: 'normal' },
  { label: '低', value: 'low' },
]

const filterOptions = [
  { label: '全部', value: 'all' },
  { label: '未完成', value: 'active' },
  { label: '已完成', value: 'completed' },
]

const sortOptions = [
  { label: '创建时间', value: 'createdAt' },
  { label: '截止日期', value: 'dueDate' },
  { label: '优先级', value: 'priority' },
]

const newTask = reactive({
  title: '',
  description: '',
  dueDate: '',
  priority: 'normal',
})

const editForm = reactive({
  title: '',
  description: '',
  dueDate: '',
  priority: 'normal',
})

const tasks = ref(loadTasks())
const activeFilter = ref('all')
const searchText = ref('')
const sortType = ref('createdAt')
const editingId = ref('')
const errorMessage = ref('')

const stats = computed(() => {
  const total = tasks.value.length
  const completed = tasks.value.filter((task) => task.completed).length
  const active = total - completed
  const completionRate = total ? Math.round((completed / total) * 100) : 0

  return { total, completed, active, completionRate }
})

const filteredTasks = computed(() => {
  const keyword = searchText.value.trim().toLowerCase()

  return tasks.value
    .filter((task) => {
      if (activeFilter.value === 'active') return !task.completed
      if (activeFilter.value === 'completed') return task.completed
      return true
    })
    .filter((task) => {
      if (!keyword) return true

      return (
        task.title.toLowerCase().includes(keyword) ||
        task.description.toLowerCase().includes(keyword)
      )
    })
    .slice()
    .sort((firstTask, secondTask) => {
      if (sortType.value === 'priority') {
        const priorityRank = { high: 3, normal: 2, low: 1 }
        return priorityRank[secondTask.priority] - priorityRank[firstTask.priority]
      }

      if (sortType.value === 'dueDate') {
        if (!firstTask.dueDate && !secondTask.dueDate) return 0
        if (!firstTask.dueDate) return 1
        if (!secondTask.dueDate) return -1
        return firstTask.dueDate.localeCompare(secondTask.dueDate)
      }

      return new Date(secondTask.createdAt) - new Date(firstTask.createdAt)
    })
})

const emptyMessage = computed(() => {
  if (tasks.value.length === 0) return '还没有任务，先添加一个待办事项吧。'
  return '没有找到匹配的任务。'
})

watch(
  tasks,
  (value) => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(value))
  },
  { deep: true },
)

function loadTasks() {
  try {
    const savedTasks = localStorage.getItem(STORAGE_KEY)
    return savedTasks ? JSON.parse(savedTasks) : []
  } catch {
    return []
  }
}

function createTask() {
  const title = newTask.title.trim()
  if (!title) {
    errorMessage.value = '任务标题不能为空。'
    return
  }

  const now = new Date().toISOString()

  tasks.value.push({
    id: crypto.randomUUID(),
    title,
    description: newTask.description.trim(),
    completed: false,
    priority: newTask.priority,
    dueDate: newTask.dueDate,
    createdAt: now,
    updatedAt: now,
    completedAt: '',
  })

  newTask.title = ''
  newTask.description = ''
  newTask.dueDate = ''
  newTask.priority = 'normal'
  errorMessage.value = ''
}

function startEdit(task) {
  editingId.value = task.id
  editForm.title = task.title
  editForm.description = task.description
  editForm.dueDate = task.dueDate
  editForm.priority = task.priority
  errorMessage.value = ''
}

function cancelEdit() {
  editingId.value = ''
  errorMessage.value = ''
}

function saveEdit(taskId) {
  const title = editForm.title.trim()
  if (!title) {
    errorMessage.value = '任务标题不能为空。'
    return
  }

  const task = tasks.value.find((item) => item.id === taskId)
  if (!task) return

  task.title = title
  task.description = editForm.description.trim()
  task.dueDate = editForm.dueDate
  task.priority = editForm.priority
  task.updatedAt = new Date().toISOString()

  editingId.value = ''
  errorMessage.value = ''
}

function toggleTask(task) {
  const completed = !task.completed
  task.completed = completed
  task.completedAt = completed ? new Date().toISOString() : ''
  task.updatedAt = new Date().toISOString()
}

function deleteTask(taskId) {
  if (!window.confirm('确定要删除这条任务吗？删除后不可恢复。')) return
  tasks.value = tasks.value.filter((task) => task.id !== taskId)
}

function clearCompleted() {
  if (stats.value.completed === 0) return
  if (!window.confirm('确定要清除所有已完成任务吗？删除后不可恢复。')) return
  tasks.value = tasks.value.filter((task) => !task.completed)
}

function formatDateTime(value) {
  return new Intl.DateTimeFormat('zh-CN', {
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
  }).format(new Date(value))
}

function priorityLabel(value) {
  return priorityOptions.find((option) => option.value === value)?.label || '普通'
}
</script>

<template>
  <main class="app-shell">
    <section class="hero">
      <div>
        <p class="eyebrow">TodoList V1</p>
        <h1>任务管理</h1>
        <p class="hero-copy">记录、筛选和完成今天要处理的事情，数据只保存在当前浏览器。</p>
      </div>

      <div class="stats-grid" aria-label="任务统计">
        <div class="stat-item">
          <span>{{ stats.total }}</span>
          <small>总任务</small>
        </div>
        <div class="stat-item">
          <span>{{ stats.active }}</span>
          <small>未完成</small>
        </div>
        <div class="stat-item">
          <span>{{ stats.completed }}</span>
          <small>已完成</small>
        </div>
        <div class="stat-item">
          <span>{{ stats.completionRate }}%</span>
          <small>完成率</small>
        </div>
      </div>
    </section>

    <section class="task-composer" aria-labelledby="composer-title">
      <div class="section-heading">
        <h2 id="composer-title">新增任务</h2>
        <p v-if="errorMessage" class="error-message">{{ errorMessage }}</p>
      </div>

      <form class="composer-form" @submit.prevent="createTask">
        <label class="field field-title">
          <span>任务标题</span>
          <input
            v-model="newTask.title"
            type="text"
            placeholder="例如：完成任务管理页面"
            @keydown.enter.prevent="createTask"
          />
        </label>

        <label class="field">
          <span>优先级</span>
          <select v-model="newTask.priority">
            <option
              v-for="option in priorityOptions"
              :key="option.value"
              :value="option.value"
            >
              {{ option.label }}
            </option>
          </select>
        </label>

        <label class="field">
          <span>截止日期</span>
          <input v-model="newTask.dueDate" type="date" />
        </label>

        <label class="field field-description">
          <span>任务描述</span>
          <textarea
            v-model="newTask.description"
            rows="3"
            placeholder="补充任务细节，可不填"
          ></textarea>
        </label>

        <button class="primary-button" type="submit">新增任务</button>
      </form>
    </section>

    <section class="toolbar" aria-label="任务工具">
      <label class="search-field">
        <span>搜索</span>
        <input v-model="searchText" type="search" placeholder="搜索标题或描述" />
      </label>

      <div class="segmented-control" aria-label="任务筛选">
        <button
          v-for="option in filterOptions"
          :key="option.value"
          :class="{ active: activeFilter === option.value }"
          type="button"
          @click="activeFilter = option.value"
        >
          {{ option.label }}
        </button>
      </div>

      <label class="sort-field">
        <span>排序</span>
        <select v-model="sortType">
          <option v-for="option in sortOptions" :key="option.value" :value="option.value">
            {{ option.label }}
          </option>
        </select>
      </label>

      <button
        class="ghost-button"
        type="button"
        :disabled="stats.completed === 0"
        @click="clearCompleted"
      >
        清除已完成
      </button>
    </section>

    <section class="task-list" aria-label="任务列表">
      <article
        v-for="task in filteredTasks"
        :key="task.id"
        class="task-item"
        :class="{ completed: task.completed }"
      >
        <template v-if="editingId === task.id">
          <div class="edit-grid">
            <label class="field field-title">
              <span>任务标题</span>
              <input v-model="editForm.title" type="text" />
            </label>

            <label class="field">
              <span>优先级</span>
              <select v-model="editForm.priority">
                <option
                  v-for="option in priorityOptions"
                  :key="option.value"
                  :value="option.value"
                >
                  {{ option.label }}
                </option>
              </select>
            </label>

            <label class="field">
              <span>截止日期</span>
              <input v-model="editForm.dueDate" type="date" />
            </label>

            <label class="field field-description">
              <span>任务描述</span>
              <textarea v-model="editForm.description" rows="3"></textarea>
            </label>
          </div>

          <div class="task-actions">
            <button class="primary-button small" type="button" @click="saveEdit(task.id)">
              保存
            </button>
            <button class="ghost-button small" type="button" @click="cancelEdit">取消</button>
          </div>
        </template>

        <template v-else>
          <div class="task-content">
            <label class="checkbox-label">
              <input
                type="checkbox"
                :checked="task.completed"
                @change="toggleTask(task)"
              />
              <span class="task-title">{{ task.title }}</span>
            </label>

            <p v-if="task.description" class="task-description">{{ task.description }}</p>

            <div class="task-meta">
              <span :class="['priority-badge', task.priority]">
                {{ priorityLabel(task.priority) }}优先级
              </span>
              <span>创建：{{ formatDateTime(task.createdAt) }}</span>
              <span v-if="task.dueDate">截止：{{ task.dueDate }}</span>
            </div>
          </div>

          <div class="task-actions">
            <button class="ghost-button small" type="button" @click="startEdit(task)">
              编辑
            </button>
            <button class="danger-button small" type="button" @click="deleteTask(task.id)">
              删除
            </button>
          </div>
        </template>
      </article>

      <div v-if="filteredTasks.length === 0" class="empty-state">
        {{ emptyMessage }}
      </div>
    </section>
  </main>
</template>
