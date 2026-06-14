<template>
  <div class="story-list">
    <div class="container">
      <section class="hero card">
        <div class="hero-left">
          <h1>开启你的微小说之旅 ✍️</h1>
          <p class="hero-desc">
            每篇故事最多允许 <strong>{{ config?.maxParticipants || 10 }}</strong> 位作者参与接龙，
            总字数上限 <strong>{{ config?.maxCharsPerStory || 5000 }}</strong> 字。
            邀请你的朋友们一起创作有趣的故事吧！
          </p>
          <button class="btn-primary" @click="showCreate = true">
            + 创建新故事
          </button>
        </div>
        <div class="hero-stats">
          <div class="stat-item">
            <span class="stat-num">{{ stories.length }}</span>
            <span class="stat-label">故事总数</span>
          </div>
          <div class="stat-item">
            <span class="stat-num">{{ ongoingCount }}</span>
            <span class="stat-label">进行中</span>
          </div>
          <div class="stat-item">
            <span class="stat-num">{{ lockedCount }}</span>
            <span class="stat-label">已完结</span>
          </div>
        </div>
      </section>

      <section class="list-section">
        <div class="section-header">
          <div class="section-title-row">
            <h2>📚 故事广场</h2>
            <div v-if="activeTag" class="active-tag-wrap">
              <span class="active-tag-label">当前标签：</span>
              <span class="tag tag-info active-tag">{{ activeTag }}</span>
              <button class="clear-tag-btn" @click="clearTag">×</button>
            </div>
          </div>
        </div>

        <div v-if="loading" class="loading card">正在加载故事列表</div>

        <div v-else-if="stories.length === 0" class="empty card">
          <div class="empty-icon">📝</div>
          <p v-if="activeTag">标签「{{ activeTag }}」下还没有故事</p>
          <p v-else>还没有故事，点击上方按钮创建第一个吧！</p>
          <button v-if="activeTag" class="btn-secondary" style="margin-top: 16px" @click="clearTag">
            查看全部故事
          </button>
        </div>

        <div v-else class="story-grid">
          <div
            v-for="s in stories"
            :key="s.id"
            class="story-card"
          >
            <router-link :to="`/story/${s.id}`" class="story-card-link">
              <div class="story-card-header">
                <h3 class="story-title">{{ s.title }}</h3>
                <span
                  :class="['tag', s.locked ? 'tag-success' : 'tag-warning']"
                >{{ s.locked ? '已完结' : '接龙中' }}</span>
              </div>
              <div v-if="s.tags && s.tags.length > 0" class="story-tags">
                <span
                  v-for="t in s.tags"
                  :key="t"
                  class="tag tag-info story-tag"
                  @click.stop="filterByTag(t)"
                >#{{ t }}</span>
              </div>
              <div class="story-card-body">
              <div class="story-meta">
                <span class="meta-item">
                  <span class="meta-icon">👥</span>
                  <span>{{ s.participantCount }}/{{ config?.maxParticipants || 10 }} 人</span>
                </span>
                <span class="meta-item">
                  <span class="meta-icon">✏️</span>
                  <span>{{ s.entryCount }} 段</span>
                </span>
                <span class="meta-item">
                  <span class="meta-icon">📝</span>
                  <span>{{ s.totalChars }}/{{ config?.maxCharsPerStory || 5000 }} 字</span>
                </span>
              </div>
                <div class="progress-wrap">
                  <div
                    class="progress-bar"
                    :style="{
                      width: Math.min(
                        Math.max(
                          (s.participantCount / (config?.maxParticipants || 10)) * 100,
                          (s.totalChars / (config?.maxCharsPerStory || 5000)) * 100
                        ),
                        100
                      ) + '%'
                    }"
                  ></div>
                </div>
              </div>
              <div class="story-card-footer">
                <span class="time">更新于 {{ formatTime(s.updatedAt) }}</span>
              </div>
            </router-link>
          </div>
        </div>
      </section>
    </div>

    <div v-if="showCreate" class="modal-mask" @click.self="showCreate = false">
      <div class="modal card">
        <div class="modal-header">
          <h3>✨ 创建新故事</h3>
          <button class="close-btn" @click="showCreate = false">×</button>
        </div>
        <div class="modal-body">
          <div class="form-group">
            <label>故事标题</label>
            <input v-model="form.title" placeholder="请输入故事标题..." maxlength="50" />
          </div>
          <div class="form-row">
            <div class="form-group">
              <label>你的笔名</label>
              <input v-model="form.author" placeholder="请输入笔名..." maxlength="20" />
            </div>
          </div>
          <div class="form-group">
            <label>故事标签 <span class="hint-inline">（最多5个，回车或逗号分隔）</span></label>
            <div class="tag-input-wrap">
              <div class="tag-list">
                <span
                  v-for="(t, idx) in form.tags"
                  :key="idx"
                  class="tag tag-info input-tag"
                >
                  #{{ t }}
                  <button class="tag-remove" @click="removeTag(idx)">×</button>
                </span>
              </div>
              <input
                v-model="tagInput"
                class="tag-input"
                placeholder="输入标签后回车..."
                maxlength="10"
                @keydown.enter.prevent="addTag"
                @keydown.,="addTag"
                @keydown.，="addTag"
              />
            </div>
            <div class="hint">{{ form.tags.length }}/5 个标签，单个标签不超过10字</div>
          </div>
          <div class="form-group">
            <label>开篇内容</label>
            <textarea
              v-model="form.content"
              placeholder="写下故事的第一段吧..."
              rows="6"
              :maxlength="config?.maxCharsPerStory || 5000"
            ></textarea>
            <div class="hint">
              {{ form.content.length }} / {{ config?.maxCharsPerStory || 5000 }} 字
            </div>
          </div>
          <div v-if="submitError" class="error-text">{{ submitError }}</div>
        </div>
        <div class="modal-footer">
          <button class="btn-secondary" @click="showCreate = false">取消</button>
          <button class="btn-primary" :disabled="submitting" @click="handleCreate">
            {{ submitting ? '创建中...' : '开始创作' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import api from '../api.js'
import { formatTime } from '../utils.js'

const router = useRouter()
const route = useRoute()
const stories = ref([])
const config = ref(null)
const loading = ref(false)
const showCreate = ref(false)
const submitting = ref(false)
const submitError = ref('')
const tagInput = ref('')
const activeTag = ref('')
const form = ref({
  title: '',
  author: '',
  content: '',
  tags: []
})

const ongoingCount = computed(() => stories.value.filter(s => !s.locked).length)
const lockedCount = computed(() => stories.value.filter(s => s.locked).length)

function addTag() {
  const t = tagInput.value.trim().replace(/^#/, '')
  if (!t) return
  if (form.value.tags.includes(t)) {
    tagInput.value = ''
    return
  }
  if (form.value.tags.length >= 5) {
    submitError.value = '最多只能添加5个标签'
    tagInput.value = ''
    return
  }
  form.value.tags.push(t)
  tagInput.value = ''
  submitError.value = ''
}

function removeTag(idx) {
  form.value.tags.splice(idx, 1)
}

function filterByTag(tag) {
  activeTag.value = tag
  router.push({ query: { tag } })
  loadData()
}

function clearTag() {
  activeTag.value = ''
  router.push({ query: {} })
  loadData()
}

async function loadData() {
  loading.value = true
  try {
    const [list, cfg] = await Promise.all([api.getStories(activeTag.value), api.getConfig()])
    stories.value = list
    config.value = cfg
  } catch (e) {
    console.error(e)
  } finally {
    loading.value = false
  }
}

async function handleCreate() {
  submitError.value = ''
  if (!form.value.title.trim()) {
    submitError.value = '请填写故事标题'
    return
  }
  if (!form.value.author.trim()) {
    submitError.value = '请填写你的笔名'
    return
  }
  if (!form.value.content.trim()) {
    submitError.value = '请填写开篇内容'
    return
  }
  submitting.value = true
  try {
    const story = await api.createStory({
      title: form.value.title.trim(),
      author: form.value.author.trim(),
      content: form.value.content.trim(),
      tags: form.value.tags
    })
    showCreate.value = false
    form.value = { title: '', author: '', content: '', tags: [] }
    router.push(`/story/${story.id}`)
  } catch (e) {
    submitError.value = e.message
  } finally {
    submitting.value = false
  }
}

onMounted(() => {
  const tagFromQuery = route.query.tag
  if (tagFromQuery) {
    activeTag.value = tagFromQuery
  }
  loadData()
})

watch(() => route.query.tag, (newTag) => {
  if (newTag !== activeTag.value) {
    activeTag.value = newTag || ''
    loadData()
  }
})
</script>

<style scoped>
.hero {
  display: flex;
  gap: 40px;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 32px;
  background: linear-gradient(135deg, #eef2ff 0%, #fdf2f8 100%);
  border-color: #e0e7ff;
}

.hero-left {
  flex: 1;
}

.hero h1 {
  font-size: 28px;
  margin-bottom: 12px;
  color: var(--text);
}

.hero-desc {
  color: var(--text-muted);
  margin-bottom: 20px;
  font-size: 15px;
  max-width: 500px;
}

.hero-desc strong {
  color: var(--primary);
}

.hero-stats {
  display: flex;
  gap: 24px;
  flex-shrink: 0;
}

.stat-item {
  text-align: center;
  padding: 16px 24px;
  background: var(--surface);
  border-radius: var(--radius);
  box-shadow: var(--shadow-sm);
  min-width: 90px;
}

.stat-num {
  display: block;
  font-size: 28px;
  font-weight: 700;
  background: linear-gradient(135deg, var(--primary), var(--accent));
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.stat-label {
  font-size: 12px;
  color: var(--text-muted);
}

.section-header {
  margin-bottom: 16px;
}

.section-title-row {
  display: flex;
  align-items: center;
  gap: 16px;
  flex-wrap: wrap;
}

.section-header h2 {
  font-size: 20px;
}

.active-tag-wrap {
  display: flex;
  align-items: center;
  gap: 8px;
}

.active-tag-label {
  font-size: 14px;
  color: var(--text-muted);
}

.active-tag {
  font-size: 13px;
}

.clear-tag-btn {
  background: none;
  border: 1px solid var(--border);
  border-radius: 50%;
  width: 22px;
  height: 22px;
  font-size: 14px;
  line-height: 1;
  color: var(--text-muted);
  padding: 0;
  display: flex;
  align-items: center;
  justify-content: center;
}

.clear-tag-btn:hover {
  background: var(--surface-alt);
  color: var(--text);
}

.story-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 16px;
}

.story-card {
  display: block;
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 20px;
  color: var(--text);
  transition: all 0.25s;
  cursor: pointer;
}

.story-card-link {
  display: block;
  color: inherit;
}

.story-card-link:hover {
  color: inherit;
}

.story-card:hover {
  transform: translateY(-3px);
  box-shadow: var(--shadow-lg);
  border-color: var(--primary-light);
}

.story-card-header {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 12px;
  margin-bottom: 14px;
}

.story-title {
  font-size: 17px;
  font-weight: 600;
  line-height: 1.4;
}

.story-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  margin-bottom: 12px;
}

.story-tag {
  cursor: pointer;
  transition: all 0.2s;
}

.story-tag:hover {
  background: rgba(99, 102, 241, 0.2);
  transform: translateY(-1px);
}

.hint-inline {
  font-weight: 400;
  color: var(--text-muted);
  font-size: 12px;
}

.tag-input-wrap {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 8px;
  padding: 8px;
  border: 1px solid var(--border);
  border-radius: var(--radius-sm);
  background: var(--surface);
  min-height: 44px;
  transition: border-color 0.2s;
}

.tag-input-wrap:focus-within {
  border-color: var(--primary);
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.1);
}

.tag-list {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.input-tag {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 12px;
  padding: 4px 10px;
}

.tag-remove {
  background: none;
  border: none;
  color: var(--primary);
  font-size: 14px;
  cursor: pointer;
  padding: 0;
  line-height: 1;
  opacity: 0.7;
  width: auto;
  height: auto;
}

.tag-remove:hover {
  opacity: 1;
}

.tag-input {
  border: none;
  outline: none;
  padding: 4px;
  font-size: 13px;
  min-width: 100px;
  flex: 1;
  background: transparent;
  box-shadow: none;
  width: auto;
}

.tag-input:focus {
  border: none;
  box-shadow: none;
}

.story-meta {
  display: flex;
  flex-wrap: wrap;
  gap: 8px 16px;
  font-size: 13px;
  color: var(--text-muted);
  margin-bottom: 12px;
}

.meta-item {
  display: flex;
  align-items: center;
  gap: 4px;
}

.progress-wrap {
  height: 6px;
  background: var(--surface-alt);
  border-radius: 3px;
  overflow: hidden;
  margin-bottom: 12px;
}

.progress-bar {
  height: 100%;
  background: linear-gradient(90deg, var(--primary), var(--accent));
  border-radius: 3px;
  transition: width 0.3s;
}

.story-card-footer {
  padding-top: 12px;
  border-top: 1px solid var(--border);
}

.time {
  font-size: 12px;
  color: var(--text-light);
}

.modal-mask {
  position: fixed;
  inset: 0;
  background: rgba(15, 23, 42, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  padding: 20px;
}

.modal {
  width: 100%;
  max-width: 520px;
  max-height: 90vh;
  overflow-y: auto;
  animation: slideUp 0.25s ease;
}

@keyframes slideUp {
  from {
    transform: translateY(20px);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding-bottom: 16px;
  border-bottom: 1px solid var(--border);
  margin-bottom: 20px;
}

.modal-header h3 {
  font-size: 18px;
}

.close-btn {
  background: none;
  font-size: 28px;
  color: var(--text-muted);
  width: 32px;
  height: 32px;
  padding: 0;
  line-height: 1;
  border-radius: 50%;
}

.close-btn:hover {
  background: var(--surface-alt);
  color: var(--text);
}

.modal-footer {
  display: flex;
  justify-content: flex-end;
  gap: 12px;
  padding-top: 16px;
  border-top: 1px solid var(--border);
  margin-top: 8px;
}

@media (max-width: 640px) {
  .hero {
    flex-direction: column;
    gap: 24px;
  }
  .hero-stats {
    width: 100%;
    justify-content: space-around;
  }
}
</style>
