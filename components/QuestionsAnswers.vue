<script setup lang="ts">
import { computed, nextTick, onMounted, onUnmounted, ref, watch } from 'vue'
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

const tagPriority = [
  'Личное',
  'Писательство',
  'Про иллюстрации',
  'Monogatari Series',
  'Zaregoto Series',
  'Katanagatari',
  'Forgetful Detective',
  'Medaka Box',
]

const allTags = [...new Set(questions.flatMap(({ tags }) => tags))]
  .sort((left, right) => {
    const leftPriority = tagPriority.indexOf(left)
    const rightPriority = tagPriority.indexOf(right)

    if (leftPriority !== -1 || rightPriority !== -1) {
      if (leftPriority === -1) return 1
      if (rightPriority === -1) return -1
      return leftPriority - rightPriority
    }

    return left.localeCompare(right, 'ru')
  })

const selectedTags = ref(new Set(allTags))
const storageReady = ref(false)
const selectedTagsStorageKey = 'nisio-questions-selected-tags'
const openedImage = ref<{ src: string; alt: string } | null>(null)
const referencedQuestion = ref<QuestionItem | null>(null)
const showDesktopBackToTop = ref(false)
const filtersElement = ref<HTMLElement | null>(null)
const questionViewerElement = ref<HTMLElement | null>(null)
const imageViewerElement = ref<HTMLElement | null>(null)
let bodyOverflowBeforeViewer = ''
let isScrollLocked = false
let referencedQuestionTrigger: HTMLElement | null = null
let imageViewerTrigger: HTMLElement | null = null
let pageLayoutElement: HTMLElement | null = null
let pageLayoutWasInert = false

const focusableSelector = [
  'a[href]',
  'button:not([disabled])',
  'iframe',
  'video[controls]',
  '[tabindex]:not([tabindex="-1"])',
].join(',')

const focusFirstElement = async (container: { value: HTMLElement | null }) => {
  await nextTick()
  container.value
    ?.querySelector<HTMLElement>(focusableSelector)
    ?.focus({ preventScroll: true })
}

const restoreFocus = async (element: HTMLElement | null) => {
  await nextTick()

  if (element?.isConnected) {
    element.focus({ preventScroll: true })
  }
}

const updateDesktopBackToTop = () => {
  showDesktopBackToTop.value = window.scrollY > window.innerHeight * 3
}

const scrollToTop = () => {
  const reduceMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches
  filtersElement.value?.scrollIntoView({
    behavior: reduceMotion ? 'auto' : 'smooth',
    block: 'start',
  })
}

const handleMobileBackToTop = (event: MouseEvent) => {
  if (!window.matchMedia('(max-width: 959px)').matches) return

  const target = event.target as Element | null
  const button = target?.closest<HTMLButtonElement>('.VPLocalNavOutlineDropdown button')

  if (!button || button.textContent?.trim() !== 'Наверх') return

  event.preventDefault()
  event.stopImmediatePropagation()
  scrollToTop()
}

const closeImage = () => {
  const trigger = imageViewerTrigger
  openedImage.value = null
  imageViewerTrigger = null
  void restoreFocus(trigger)
}

const closeReferencedQuestion = () => {
  const trigger = referencedQuestionTrigger
  referencedQuestion.value = null
  referencedQuestionTrigger = null
  void restoreFocus(trigger)
}

const handleKeydown = (event: KeyboardEvent) => {
  if (event.key === 'Escape') {
    if (!openedImage.value && !referencedQuestion.value) return

    event.preventDefault()

    if (openedImage.value) {
      closeImage()
    } else {
      closeReferencedQuestion()
    }

    return
  }

  if (event.key !== 'Tab') return

  const activeViewer = imageViewerElement.value || questionViewerElement.value

  if (!activeViewer) return

  const focusableElements = [...activeViewer.querySelectorAll<HTMLElement>(focusableSelector)]

  if (focusableElements.length === 0) {
    event.preventDefault()
    return
  }

  const firstElement = focusableElements[0]
  const lastElement = focusableElements[focusableElements.length - 1]
  const activeElement = document.activeElement

  if (!activeViewer.contains(activeElement)) {
    event.preventDefault()
    firstElement.focus()
  } else if (event.shiftKey && activeElement === firstElement) {
    event.preventDefault()
    lastElement.focus()
  } else if (!event.shiftKey && activeElement === lastElement) {
    event.preventDefault()
    firstElement.focus()
  }
}

onMounted(() => {
  window.addEventListener('keydown', handleKeydown)
  window.addEventListener('scroll', updateDesktopBackToTop, { passive: true })
  document.addEventListener('click', handleMobileBackToTop, true)
  updateDesktopBackToTop()

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

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown)
  window.removeEventListener('scroll', updateDesktopBackToTop)
  document.removeEventListener('click', handleMobileBackToTop, true)

  if (isScrollLocked) {
    document.body.style.overflow = bodyOverflowBeforeViewer
  }

  if (pageLayoutElement && !pageLayoutWasInert) {
    pageLayoutElement.removeAttribute('inert')
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

watch(() => Boolean(openedImage.value || referencedQuestion.value), (overlayIsOpen) => {
  if (overlayIsOpen && !isScrollLocked) {
    bodyOverflowBeforeViewer = document.body.style.overflow
    document.body.style.overflow = 'hidden'
    isScrollLocked = true
    pageLayoutElement = document.querySelector<HTMLElement>('.Layout')
    pageLayoutWasInert = pageLayoutElement?.hasAttribute('inert') || false
    pageLayoutElement?.setAttribute('inert', '')
  } else if (!overlayIsOpen && isScrollLocked) {
    document.body.style.overflow = bodyOverflowBeforeViewer
    isScrollLocked = false

    if (pageLayoutElement && !pageLayoutWasInert) {
      pageLayoutElement.removeAttribute('inert')
    }

    pageLayoutElement = null
    pageLayoutWasInert = false
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

const resetTagRepulsion = (tagList: HTMLElement) => {
  tagList.querySelectorAll<HTMLElement>('.tag-button').forEach((tagButton) => {
    tagButton.style.removeProperty('--tag-shift-x')
    tagButton.style.removeProperty('--tag-shift-y')
  })
}

const repelTags = (event: PointerEvent) => {
  const tagList = event.currentTarget as HTMLElement
  const repulsionRadius = 88
  const maximumShift = 8

  tagList.querySelectorAll<HTMLElement>('.tag-button').forEach((tagButton) => {
    const bounds = tagButton.getBoundingClientRect()
    const distanceX = bounds.left + bounds.width / 2 - event.clientX
    const distanceY = bounds.top + bounds.height / 2 - event.clientY
    const distance = Math.hypot(distanceX, distanceY)

    if (distance === 0 || distance >= repulsionRadius) {
      tagButton.style.removeProperty('--tag-shift-x')
      tagButton.style.removeProperty('--tag-shift-y')
      return
    }

    const shift = maximumShift * (1 - distance / repulsionRadius)
    tagButton.style.setProperty('--tag-shift-x', `${distanceX / distance * shift}px`)
    tagButton.style.setProperty('--tag-shift-y', `${distanceY / distance * shift}px`)
  })
}

const stopTagRepulsion = (event: PointerEvent) => {
  resetTagRepulsion(event.currentTarget as HTMLElement)
}

const illuminateQuestionNumber = (event: PointerEvent) => {
  if (event.pointerType !== 'mouse') return

  const number = event.currentTarget as HTMLElement
  const bounds = number.getBoundingClientRect()

  number.style.setProperty('--question-glow-x', `${event.clientX - bounds.left}px`)
  number.style.setProperty('--question-glow-y', `${event.clientY - bounds.top}px`)
  number.style.setProperty('--question-glow-opacity', '0.52')
}

const dimQuestionNumber = (event: PointerEvent) => {
  const number = event.currentTarget as HTMLElement
  number.style.setProperty('--question-glow-opacity', '0')
}

const wrapComments = (html: string) => {
  return html.replace(
    /\[([\s\S]*?)\]/g,
    '<span class="my-comment">[$1]</span>',
  )
}

const questionsByNumber = new Map(
  questions.map(item => [item.questionNumber.replace(/\D/g, '').padStart(3, '0'), item]),
)

const wrapQuestionReferences = (html: string) => {
  return html.replace(
    /(см\.\s*)(вопрос\s+(\d{3}))/giu,
    (match, prefix: string, label: string, questionNumber: string) => {
      if (!questionsByNumber.has(questionNumber)) return match

      return `${prefix}<button type="button" class="question-reference" data-question-number="${questionNumber}">${label}</button>`
    },
  )
}

const processRenderedHtml = (html: string, withQuestionReferences = true) => {
  const htmlWithComments = wrapComments(html)

  return withQuestionReferences
    ? wrapQuestionReferences(htmlWithComments)
    : htmlWithComments
}

const renderQuestion170Answer = (text: string) => {
  const renderedParts: string[] = []
  let plainTextStart = 0
  let commentStart = -1
  let bracketDepth = 0
  const renderWithPoemLineBreaks = (part: string) => {
    const textWithHardLineBreaks = part
      .split('\n\n')
      .map((paragraph) => {
        const lines = paragraph.split('\n')
        const paragraphWithLineBreaks = lines.join('  \n')

        return lines.length === 3
          ? `*${paragraphWithLineBreaks}*`
          : paragraphWithLineBreaks
      })
      .join('\n\n')

    return markdown.render(textWithHardLineBreaks)
  }

  for (let index = 0; index < text.length; index += 1) {
    if (text[index] === '[') {
      if (bracketDepth === 0) commentStart = index
      bracketDepth += 1
      continue
    }

    if (text[index] !== ']' || bracketDepth === 0) continue

    bracketDepth -= 1

    if (bracketDepth !== 0 || commentStart === -1) continue

    const commentText = text.slice(commentStart + 1, index)
    const isMarkdownLink = text[index + 1] === '('

    if (commentText.includes('\n') && !isMarkdownLink) {
      const renderedComment = renderWithPoemLineBreaks(commentText)
        .replace(/^<p>/, '<p>&#91;')
        .replace(/<\/p>\n?$/, '&#93;</p>')

      renderedParts.push(renderWithPoemLineBreaks(text.slice(plainTextStart, commentStart)))
      renderedParts.push(
        `<div class="my-comment my-comment-block">${renderedComment}</div>`,
      )
      plainTextStart = index + 1
    }

    commentStart = -1
  }

  if (plainTextStart === 0) return renderWithPoemLineBreaks(text)

  renderedParts.push(renderWithPoemLineBreaks(text.slice(plainTextStart)))
  return renderedParts.join('')
}

const renderMarkdown = (text: string, withQuestionReferences = true) => {
  return processRenderedHtml(markdown.render(text), withQuestionReferences)
}

const renderAnswerMarkdown = (item: QuestionItem, withQuestionReferences = true) => {
  if (item.questionNumber !== '170') {
    return renderMarkdown(item.answerText, withQuestionReferences)
  }

  return processRenderedHtml(
    renderQuestion170Answer(item.answerText),
    withQuestionReferences,
  )
}

const renderInlineMarkdown = (text: string, withQuestionReferences = true) => {
  return processRenderedHtml(markdown.renderInline(text), withQuestionReferences)
}

const handleContentClick = (event: MouseEvent) => {
  const target = event.target as Element | null
  const reference = target?.closest<HTMLButtonElement>('.question-reference')
  const questionNumber = reference?.dataset.questionNumber

  if (!questionNumber) return

  referencedQuestionTrigger = reference || null
  referencedQuestion.value = questionsByNumber.get(questionNumber) || null

  if (referencedQuestion.value) {
    void focusFirstElement(questionViewerElement)
  }
}

const imageAlt = (item: QuestionItem) => {
  return (item.answerImageDesc || item.questionText)
    .replace(/[\[\]*_`]/g, '')
    .trim()
}

const openImage = (item: QuestionItem, event: MouseEvent) => {
  if (!item.answerImage) return

  imageViewerTrigger = event.currentTarget as HTMLElement
  openedImage.value = {
    src: item.answerImage,
    alt: imageAlt(item),
  }
  void focusFirstElement(imageViewerElement)
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
  <section
    class="questions-and-answers"
    aria-labelledby="questions-filter-title"
    @click="handleContentClick"
  >
    <div ref="filtersElement" class="filters">
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

      <div
        class="tag-list"
        aria-label="Фильтры по темам"
        @pointermove="repelTags"
        @pointerleave="stopTagRepulsion"
        @pointerup="stopTagRepulsion"
        @pointercancel="stopTagRepulsion"
      >
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
        Показано вопросов:
        <span class="result-count-value">{{ filteredQuestions.length }}</span>
        из {{ questions.length }}
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
          <span
            class="question-number"
            :data-number="`${item.questionNumber}.`"
            @pointermove="illuminateQuestionNumber"
            @pointerleave="dimQuestionNumber"
          >{{ item.questionNumber }}.</span>
          <span
            class="question-text markdown-content"
            v-html="renderInlineMarkdown(item.questionText)"
          />
        </div>

        <div class="answer">
          <div
            class="answer-text markdown-content"
            v-html="renderAnswerMarkdown(item)"
          />

          <figure v-if="item.answerImage" class="answer-image">
            <button
              type="button"
              class="answer-image-button"
              :aria-label="`Открыть изображение: ${imageAlt(item)}`"
              @click="openImage(item, $event)"
            >
              <img
                :src="item.answerImage"
                :alt="imageAlt(item)"
                loading="lazy"
                decoding="async"
              >
            </button>
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
      Выберите хотя бы одну тему, чтобы увидеть вопросы и ответы.
    </div>
  </section>

  <Transition name="back-to-top">
    <button
      v-if="showDesktopBackToTop && !referencedQuestion && !openedImage"
      type="button"
      class="desktop-back-to-top"
      aria-label="Вернуться наверх"
      @click="scrollToTop"
    >
      <span aria-hidden="true">↑</span>
      Наверх
    </button>
  </Transition>

  <Teleport to="body">
    <div
      v-if="referencedQuestion"
      ref="questionViewerElement"
      class="question-viewer"
      role="dialog"
      :aria-modal="openedImage ? undefined : 'true'"
      :aria-hidden="openedImage ? 'true' : undefined"
      :inert="Boolean(openedImage)"
      :aria-labelledby="`referenced-question-${referencedQuestion.id}`"
      @click.self="closeReferencedQuestion"
    >
      <article class="question-viewer-card">
        <button
          type="button"
          class="image-viewer-close"
          aria-label="Закрыть вопрос"
          @click="closeReferencedQuestion"
        />

        <div
          :id="`referenced-question-${referencedQuestion.id}`"
          class="question-heading"
          role="heading"
          aria-level="2"
        >
          <span
            class="question-number"
            :data-number="`${referencedQuestion.questionNumber}.`"
            @pointermove="illuminateQuestionNumber"
            @pointerleave="dimQuestionNumber"
          >{{ referencedQuestion.questionNumber }}.</span>
          <span
            class="question-text markdown-content"
            v-html="renderInlineMarkdown(referencedQuestion.questionText, false)"
          />
        </div>

        <div class="answer">
          <div
            class="answer-text markdown-content"
            v-html="renderAnswerMarkdown(referencedQuestion, false)"
          />

          <figure v-if="referencedQuestion.answerImage" class="answer-image">
            <button
              type="button"
              class="answer-image-button"
              :aria-label="`Открыть изображение: ${imageAlt(referencedQuestion)}`"
              @click="openImage(referencedQuestion, $event)"
            >
              <img
                :src="referencedQuestion.answerImage"
                :alt="imageAlt(referencedQuestion)"
                loading="lazy"
                decoding="async"
              >
            </button>
            <figcaption
              v-if="referencedQuestion.answerImageDesc"
              v-html="renderInlineMarkdown(referencedQuestion.answerImageDesc, false)"
            />
          </figure>

          <div v-if="referencedQuestion.answerVideo" class="answer-video">
            <iframe
              v-if="youtubeEmbedUrl(referencedQuestion.answerVideo)"
              :src="youtubeEmbedUrl(referencedQuestion.answerVideo)!"
              :title="`Видео к вопросу ${referencedQuestion.questionNumber}`"
              loading="lazy"
              allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
              referrerpolicy="strict-origin-when-cross-origin"
              allowfullscreen
            />
            <video
              v-else-if="isDirectVideo(referencedQuestion.answerVideo)"
              :src="referencedQuestion.answerVideo"
              controls
              playsinline
              preload="metadata"
            />
            <a
              v-else
              :href="referencedQuestion.answerVideo"
              target="_blank"
              rel="noopener noreferrer"
            >
              Посмотреть видео
            </a>
          </div>
        </div>
      </article>
    </div>

    <div
      v-if="openedImage"
      ref="imageViewerElement"
      class="image-viewer"
      role="dialog"
      aria-modal="true"
      aria-label="Просмотр изображения"
      @click.self="closeImage"
    >
      <button
        type="button"
        class="image-viewer-close"
        aria-label="Закрыть изображение"
        @click="closeImage"
      />
      <img
        class="image-viewer-image"
        :src="openedImage.src"
        :alt="openedImage.alt"
        @click="closeImage"
        @touchend.prevent="closeImage"
      >
    </div>
  </Teleport>
</template>

<style scoped>
.questions-and-answers,
.desktop-back-to-top,
.question-viewer {
  --qa-orange: #ff9666;
  --qa-mint: #0fd1bd;
  --qa-orange-soft: color-mix(in srgb, var(--qa-orange) 3.6%, transparent);
  --qa-mint-soft: color-mix(in srgb, var(--qa-mint) 3.6%, transparent);
  --qa-gradient: linear-gradient(120deg, var(--qa-orange), var(--qa-mint));
  --qa-answer-bg: color-mix(in srgb, var(--vp-c-bg-soft) 30%, var(--vp-c-bg));
}

.questions-and-answers {
  margin-top: 32px;
}

:global(.dark) .questions-and-answers,
:global(.dark) .desktop-back-to-top,
:global(.dark) .question-viewer {
  --qa-orange: #ffa477;
  --qa-mint: #3ad8c7;
}

.filters {
  scroll-margin-top: calc(var(--vp-nav-height) + 16px);
  padding: 20px 20px 12px;
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
  gap: 7px;
  justify-content: center;
}

.tag-button {
  --tag-shift-x: 0px;
  --tag-shift-y: 0px;

  padding: 7px 11px;
  border-radius: 8px;
  color: var(--vp-c-text-2);
  background: var(--vp-c-bg);
  font-size: 14px;
  line-height: 1.25;
  transform: translate(var(--tag-shift-x), var(--tag-shift-y));
  transition:
    transform 0.14s ease-out,
    border-color 0.2s,
    background-color 0.2s,
    color 0.2s;
  will-change: transform;
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
  display: flex;
  align-items: baseline;
  justify-content: center;
  gap: 0.35em;
  margin: 16px 0 0;
  padding-top: 12px;
  border-top: 1px solid var(--vp-c-divider);
  color: var(--vp-c-text-2);
  font-size: 13px;
}

.result-count-value {
  color: var(--vp-c-text-1);
  font-size: 2em;
  font-weight: 700;
  line-height: 1;
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
  --question-glow-x: 50%;
  --question-glow-y: 50%;
  --question-glow-opacity: 0;

  position: relative;
  display: inline-block;
  margin-right: 0.15em;
  color: var(--qa-orange);
  font-size: 20px;
  font-weight: 700;
  letter-spacing: 0.06em;
}

.question-number::after {
  position: absolute;
  inset: 0;
  background:
    linear-gradient(
      108deg,
      transparent 12%,
      rgb(255 255 255 / 76%) 48%,
      transparent 82%
    ),
    repeating-linear-gradient(
      132deg,
      rgb(255 255 255 / 18%) 0 1px,
      transparent 1px 4px
    ),
    linear-gradient(
      118deg,
      #ff69a6 0%,
      #ffd35f 19%,
      #72efca 38%,
      #65bfff 58%,
      #bd8cff 78%,
      #ff79b7 100%
    );
  background-blend-mode: screen, overlay, normal;
  background-position:
    calc(var(--question-glow-x) - 10px) 0,
    0 0,
    0% 50%;
  background-repeat: no-repeat, repeat, no-repeat;
  background-size: 20px 100%, auto, 220% 220%;
  background-clip: text;
  -webkit-background-clip: text;
  color: transparent;
  content: attr(data-number);
  opacity: var(--question-glow-opacity);
  pointer-events: none;
  transition: opacity 0.16s ease-out;
  -webkit-text-fill-color: transparent;
  animation: question-number-hologram 2.4s ease-in-out infinite alternate;
  animation-play-state: paused;
}

.question-number:hover::after {
  animation-play-state: running;
}

@keyframes question-number-hologram {
  from {
    background-position:
      calc(var(--question-glow-x) - 10px) 0,
      0 0,
      0% 50%;
  }

  to {
    background-position:
      calc(var(--question-glow-x) - 10px) 0,
      5px -3px,
      100% 50%;
  }
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

.markdown-content :deep(.my-comment-block) {
  margin-top: 12px;
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

.markdown-content :deep(.question-reference) {
  display: inline;
  padding: 0;
  border: 0;
  color: var(--qa-orange);
  background: transparent;
  font: inherit;
  text-decoration: underline;
  text-decoration-color: color-mix(in srgb, var(--qa-orange) 60%, transparent);
  text-underline-offset: 2px;
  cursor: pointer;
  transition: color 0.2s, text-decoration-color 0.2s;
}

.markdown-content :deep(.question-reference:hover),
.markdown-content :deep(.question-reference:focus-visible) {
  color: color-mix(in srgb, var(--qa-orange) 68%, transparent);
  text-decoration-color: currentColor;
}

.answer-image {
  margin: 18px 0 0;
}

.answer-image-button {
  display: block;
  padding: 0;
  width: 100%;
  border: 0;
  border-radius: 8px;
  background: transparent;
  cursor: zoom-in;
}

.answer-image img {
  display: block;
  width: 100%;
  border-radius: 8px;
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

.question-viewer {
  position: fixed;
  inset: 0;
  z-index: 900;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 56px 24px 24px;
  background: rgb(0 0 0 / 72%);
}

.question-viewer-card {
  position: relative;
  overflow: auto;
  width: min(100%, 688px);
  max-height: calc(100vh - 80px);
  border: 1px solid color-mix(in srgb, var(--qa-mint) 24%, var(--vp-c-divider));
  border-radius: 12px;
  background: var(--vp-c-bg);
  box-shadow: 0 20px 60px rgb(0 0 0 / 35%);
}

.question-viewer-card::before {
  position: absolute;
  top: 0;
  right: 0;
  left: 0;
  height: 3px;
  background: var(--qa-gradient);
  content: '';
}

.question-viewer-card .question-heading {
  padding-right: 64px;
}

.desktop-back-to-top {
  position: fixed;
  right: 24px;
  bottom: 24px;
  z-index: 50;
  display: none;
  align-items: center;
  gap: 7px;
  padding: 9px 13px;
  border: 1px solid color-mix(in srgb, var(--qa-mint) 55%, var(--vp-c-divider));
  border-radius: 8px;
  color: var(--vp-c-text-1);
  background: color-mix(in srgb, var(--vp-c-bg) 92%, transparent);
  box-shadow: 0 6px 24px rgb(0 0 0 / 14%);
  font: inherit;
  font-size: 14px;
  cursor: pointer;
  backdrop-filter: blur(8px);
  transition: border-color 0.2s, background-color 0.2s, transform 0.2s;
}

.desktop-back-to-top:hover,
.desktop-back-to-top:focus-visible {
  border-color: var(--qa-mint);
  background: var(--vp-c-bg);
  transform: translateY(-2px);
}

.back-to-top-enter-active,
.back-to-top-leave-active {
  transition: opacity 0.2s, transform 0.2s;
}

.back-to-top-enter-from,
.back-to-top-leave-to {
  opacity: 0;
  transform: translateY(8px);
}

.image-viewer {
  position: fixed;
  inset: 0;
  z-index: 1000;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 56px 24px 24px;
  background: rgb(0 0 0 / 88%);
}

.image-viewer-image {
  display: block;
  max-width: min(100%, 1400px);
  max-height: calc(100vh - 80px);
  object-fit: contain;
  cursor: pointer;
}

.image-viewer-close {
  position: fixed;
  top: 14px;
  right: 14px;
  padding: 0;
  width: 40px;
  height: 40px;
  border: 1px solid rgb(255 255 255 / 35%);
  border-radius: 8px;
  color: #fff;
  background: rgb(0 0 0 / 45%);
  font: inherit;
  cursor: pointer;
}

.image-viewer-close::before,
.image-viewer-close::after {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 18px;
  height: 2px;
  border-radius: 1px;
  background: currentColor;
  content: '';
}

.image-viewer-close::before {
  transform: translate(-50%, -50%) rotate(45deg);
}

.image-viewer-close::after {
  transform: translate(-50%, -50%) rotate(-45deg);
}

.image-viewer-close:hover,
.image-viewer-close:focus-visible {
  background: rgb(255 255 255 / 16%);
}

@media (max-width: 640px) {
  .questions-and-answers {
    margin-top: 24px;
  }

  .filters {
    padding: 16px 16px 10px;
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
    -webkit-user-select: none;
    user-select: none;
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

@media (min-width: 960px) {
  .desktop-back-to-top {
    display: flex;
  }
}

@media (prefers-reduced-motion: reduce) {
  .tag-button {
    transform: none;
    transition: border-color 0.2s, background-color 0.2s, color 0.2s;
  }

  .desktop-back-to-top,
  .back-to-top-enter-active,
  .back-to-top-leave-active {
    transition: none;
  }

  .question-number::after {
    display: none;
  }
}
</style>
