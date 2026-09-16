<script setup lang="ts">
import { computed, onMounted, ref, watch } from 'vue'
import MarkdownIt from 'markdown-it'
import questionsData from '../files/teletype-questions.json'

interface QuestionItem {
  id: number
  questionNumber: string
  questionText: string
  tags: string[]
  answerText: string
  answerImage: string | null
  answerImageDesc: string | null
  answerVideo: string | null
}

const questions = questionsData as QuestionItem[]

const markdown = new MarkdownIt({
  html: false,
  linkify: true,
  typographer: true,
})

markdown.renderer.rules.link_open = (tokens, index, options, _env, self) => {
  tokens[index].attrSet('target', '_blank')
  tokens[index].attrSet('rel', 'noopener noreferrer')

  return self.renderToken(tokens, index, options)
}

const allTags = [...new Set(questions.flatMap(({ tags }) => tags))]
  .sort((left, right) => left.localeCompare(right, 'ru'))

const selectedTags = ref(new Set(allTags))
const storageReady = ref(false)
const selectedTagsStorageKey = 'nisio-questions-selected-tags'

onMounted(() => {
  try {
    const storedTags = localStorage.getItem(selectedTagsStorageKey)

    if (storedTags !== null) {
      const parsedTags: unknown = JSON.parse(storedTags)

      if (Array.isArray(parsedTags)) {
        selectedTags.value = new Set(
          parsedTags.filter((tag): tag is string => (
            typeof tag === 'string' && allTags.includes(tag)
          )),
        )
      }
    }
  } catch {
    // Ignore unavailable storage and malformed saved values.
  } finally {
    storageReady.value = true
  }
})

watch(selectedTags, (tags) => {
  if (!storageReady.value) return

  try {
    localStorage.setItem(selectedTagsStorageKey, JSON.stringify([...tags]))
  } catch {
    // Filtering still works when storage is unavailable.
  }
})

const filteredQuestions = computed(() => questions.filter(({ tags }) => {
  return tags.length === 0 || tags.some(tag => selectedTags.value.has(tag))
}))

const allTagsSelected = computed(() => selectedTags.value.size === allTags.length)
const noTagsSelected = computed(() => selectedTags.value.size === 0)

const toggleTag = (tag: string) => {
  const nextSelectedTags = new Set(selectedTags.value)

  if (nextSelectedTags.has(tag)) {
    nextSelectedTags.delete(tag)
  } else {
    nextSelectedTags.add(tag)
  }

  selectedTags.value = nextSelectedTags
}

const selectAllTags = () => {
  selectedTags.value = new Set(allTags)
}

const clearAllTags = () => {
  selectedTags.value = new Set()
}

const wrapComments = (html: string) => {
  return html.replace(
    /\[([\s\S]*?)\]/g,
    '<span class="my-comment">[$1]</span>',
  )
}

const renderMarkdown = (text: string) => wrapComments(markdown.render(text))
const renderInlineMarkdown = (text: string) => wrapComments(markdown.renderInline(text))

const imageAlt = (item: QuestionItem) => {
  return (item.answerImageDesc || item.questionText)
    .replace(/[\[\]*_`]/g, '')
    .trim()
}

const youtubeEmbedUrl = (source: string) => {
  try {
    const url = new URL(source)
    let videoId = ''

    if (url.hostname === 'youtu.be') {
      videoId = url.pathname.slice(1).split('/')[0]
    } else if (url.hostname === 'youtube.com' || url.hostname === 'www.youtube.com') {
      if (url.pathname === '/watch') {
        videoId = url.searchParams.get('v') || ''
      } else if (url.pathname.startsWith('/embed/')) {
        videoId = url.pathname.split('/')[2] || ''
      }
    }

    return /^[\w-]{6,}$/.test(videoId)
      ? `https://www.youtube-nocookie.com/embed/${videoId}`
      : null
  } catch {
    return null
  }
}

const isDirectVideo = (source: string) => {
  try {
    return /\.(mp4|webm|ogg)$/i.test(new URL(source).pathname)
  } catch {
    return false
  }
}
</script>

<template>
  <section class="questions-and-answers" aria-labelledby="questions-filter-title">
    <div class="filters">
      <div class="filters-heading">
        <h2 id="questions-filter-title">Темы</h2>
        <div class="filter-actions">
          <button
            type="button"
            class="filter-action filter-action-clear"
            :disabled="noTagsSelected"
            @click="clearAllTags"
          >
            Скрыть все
          </button>
          <button
            type="button"
            class="filter-action filter-action-select"
            :disabled="allTagsSelected"
            @click="selectAllTags"
          >
            Выбрать все
          </button>
        </div>
      </div>

      <div class="tag-list" aria-label="Фильтры по темам">
        <button
          v-for="tag in allTags"
          :key="tag"
          type="button"
          class="tag-button"
          :class="{ selected: selectedTags.has(tag) }"
          :aria-pressed="selectedTags.has(tag)"
          @click="toggleTag(tag)"
        >
          {{ tag }}
        </button>
      </div>

      <p class="result-count" aria-live="polite">
        Показано вопросов: {{ filteredQuestions.length }} из {{ questions.length }}
      </p>
    </div>

    <div v-if="filteredQuestions.length" class="question-list">
      <article
        v-for="item in filteredQuestions"
        :key="item.id"
        class="question-item"
        :aria-labelledby="`question-${item.id}`"
      >
        <div
          :id="`question-${item.id}`"
          class="question-heading"
          role="heading"
          aria-level="2"
        >
          <span class="question-number">{{ item.questionNumber }}.</span>
          <span
            class="question-text markdown-content"
            v-html="renderInlineMarkdown(item.questionText)"
          />
        </div>

        <div class="answer">
          <div
            class="answer-text markdown-content"
            v-html="renderMarkdown(item.answerText)"
          />

          <figure v-if="item.answerImage" class="answer-image">
            <img
              :src="item.answerImage"
              :alt="imageAlt(item)"
              loading="lazy"
              decoding="async"
            >
            <figcaption
              v-if="item.answerImageDesc"
              v-html="renderInlineMarkdown(item.answerImageDesc)"
            />
          </figure>

          <div v-if="item.answerVideo" class="answer-video">
            <iframe
              v-if="youtubeEmbedUrl(item.answerVideo)"
              :src="youtubeEmbedUrl(item.answerVideo)!"
              :title="`Видео к вопросу ${item.questionNumber}`"
              loading="lazy"
              allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
              referrerpolicy="strict-origin-when-cross-origin"
              allowfullscreen
            />
            <video
              v-else-if="isDirectVideo(item.answerVideo)"
              :src="item.answerVideo"
              controls
              playsinline
              preload="metadata"
            />
            <a v-else :href="item.answerVideo" target="_blank" rel="noopener noreferrer">
              Посмотреть видео
            </a>
          </div>
        </div>
      </article>
    </div>

    <div v-else class="empty-state" role="status">
      Выберите хотя бы одну тему, чтобы увидеть вопросы.
    </div>
  </section>
</template>

<style scoped>
.questions-and-answers {
  --qa-orange: #ff9666;
  --qa-mint: #0fd1bd;
  --qa-orange-soft: color-mix(in srgb, var(--qa-orange) 3.6%, transparent);
  --qa-mint-soft: color-mix(in srgb, var(--qa-mint) 3.6%, transparent);
  --qa-gradient: linear-gradient(120deg, var(--qa-orange), var(--qa-mint));
  --qa-answer-bg: color-mix(in srgb, var(--vp-c-bg-soft) 30%, var(--vp-c-bg));

  margin-top: 32px;
}

:global(.dark) .questions-and-answers {
  --qa-orange: #ffa477;
  --qa-mint: #3ad8c7;
}

.filters {
  padding: 20px;
  border: 1px solid color-mix(in srgb, var(--qa-mint) 32%, var(--vp-c-divider));
  border-radius: 12px;
  background:
    radial-gradient(circle at 0 0, var(--qa-mint-soft), transparent 42%),
    radial-gradient(circle at 100% 100%, var(--qa-orange-soft), transparent 46%),
    var(--vp-c-bg-soft);
}

.filters-heading {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  margin-bottom: 16px;
}

.filters-heading h2 {
  margin: 0;
  padding: 0;
  border: 0;
  font-size: 20px;
}

.filter-action,
.tag-button {
  border: 1px solid var(--vp-c-divider);
  font: inherit;
  cursor: pointer;
  transition: border-color 0.2s, background-color 0.2s, color 0.2s;
}

.filter-actions {
  display: flex;
  flex: none;
  gap: 8px;
}

.filter-action {
  flex: none;
  padding: 6px 10px;
  border-radius: 8px;
  color: var(--qa-mint);
  background: var(--vp-c-bg);
  font-size: 14px;
  font-weight: 600;
}

.filter-action-clear {
  color: var(--qa-orange);
}

.filter-action-select {
  color: var(--qa-mint);
}

.filter-action:disabled {
  color: var(--vp-c-text-3);
  cursor: default;
}

.filter-action:not(:disabled):hover,
.filter-action:not(:disabled):focus-visible {
  border-color: currentColor;
  background: var(--qa-mint-soft);
}

.filter-action-clear:not(:disabled):hover,
.filter-action-clear:not(:disabled):focus-visible {
  background: var(--qa-orange-soft);
}

.tag-list {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.tag-button {
  padding: 7px 11px;
  border-radius: 999px;
  color: var(--vp-c-text-2);
  background: var(--vp-c-bg);
  font-size: 14px;
  line-height: 1.25;
}

.tag-button:hover,
.tag-button:focus-visible {
  border-color: var(--qa-mint);
}

.tag-button.selected {
  border-color: #20aa9b;
  color: #fff;
  background: #20aa9b;
}

.result-count {
  margin: 14px 0 0;
  color: var(--vp-c-text-2);
  font-size: 13px;
}

.question-list {
  display: grid;
  gap: 20px;
  margin-top: 24px;
}

.question-item {
  position: relative;
  overflow: hidden;
  border: 1px solid color-mix(in srgb, var(--qa-mint) 18%, var(--vp-c-divider));
  border-radius: 12px;
  background: var(--vp-c-bg);
}

.question-item::before {
  position: absolute;
  top: 0;
  right: 0;
  left: 0;
  height: 3px;
  background: var(--qa-gradient);
  content: '';
}

.question-number {
  margin-right: 0.15em;
  color: var(--qa-orange);
  font-size: 20px;
  font-weight: 700;
  letter-spacing: 0.06em;
}

.question-heading {
  margin: 0;
  padding: 20px;
  font-size: 17px;
  line-height: 1.45;
}

.question-text {
  font-size: inherit;
}

.answer {
  padding: 18px 20px 20px;
  border-top: 1px solid color-mix(in srgb, var(--qa-mint) 20%, var(--vp-c-divider));
  background:
    linear-gradient(135deg, var(--qa-mint-soft), transparent 46%),
    var(--qa-answer-bg);
}

.answer-text {
  margin-top: 0;
}

.markdown-content :deep(p) {
  margin: 0;
}

.markdown-content :deep(p + p) {
  margin-top: 12px;
}

.markdown-content :deep(.my-comment),
.answer-image figcaption :deep(.my-comment) {
  opacity: 0.72;
}

.markdown-content :deep(a),
.answer-image figcaption :deep(a),
.answer-video a {
  color: var(--qa-orange);
  text-decoration-color: color-mix(in srgb, var(--qa-orange) 60%, transparent);
  text-underline-offset: 2px;
  transition: color 0.2s, text-decoration-color 0.2s;
}

.markdown-content :deep(a:hover),
.markdown-content :deep(a:focus-visible),
.answer-image figcaption :deep(a:hover),
.answer-image figcaption :deep(a:focus-visible),
.answer-video a:hover,
.answer-video a:focus-visible {
  color: color-mix(in srgb, var(--qa-orange) 68%, transparent);
  text-decoration-color: currentColor;
}

.answer-image {
  margin: 18px 0 0;
}

.answer-image img {
  display: block;
  width: 100%;
  max-height: 720px;
  border-radius: 8px;
  object-fit: contain;
  background: var(--vp-c-bg);
}

.answer-image figcaption {
  margin-top: 8px;
  color: var(--vp-c-text-2);
  font-size: 13px;
  text-align: center;
}

.answer-video {
  margin-top: 18px;
}

.answer-video iframe,
.answer-video video {
  display: block;
  width: 100%;
  aspect-ratio: 16 / 9;
  border: 0;
  border-radius: 8px;
  background: #000;
}

.empty-state {
  margin-top: 24px;
  padding: 32px 20px;
  border: 1px dashed color-mix(in srgb, var(--qa-mint) 55%, var(--vp-c-divider));
  border-radius: 12px;
  color: var(--vp-c-text-2);
  background: linear-gradient(135deg, var(--qa-orange-soft), var(--qa-mint-soft));
  text-align: center;
}

@media (max-width: 640px) {
  .questions-and-answers {
    margin-top: 24px;
  }

  .filters {
    padding: 16px;
  }

  .filters-heading {
    align-items: center;
    gap: 8px;
  }

  .filters-heading h2 {
    font-size: 18px;
  }

  .filter-actions {
    gap: 4px;
  }

  .filter-action {
    padding: 5px 8px;
    font-size: 13px;
  }

  .tag-button {
    padding: 7px 10px;
    font-size: 13px;
  }

  .question-list {
    gap: 16px;
  }

  .question-heading {
    padding: 16px;
  }

  .answer {
    padding: 16px;
  }
}
</style>
