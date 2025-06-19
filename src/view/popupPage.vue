<template>
  <div class="fail" v-if="isChromeExtension">
    This extension not support on extension page
  </div>
  <div id="main_app" v-if="loaded && !isChromeExtension">
    <div class="not-support" v-if="!generateSummaryLoading && !summaryContent">
      This website is not currently supported. Please try a different page or retry<br>
      <button class="reload-btn" @click="reload"><img src="@/assets/rotate-right-regular.svg"
          style="display:inline; height:15px; margin-right:5px;vertical-align: bottom;">Retry</button>
    </div>
    <div class="lang-selection-area" v-if="!generateSummaryLoading && !!summaryContent">
      <select data-placeholder="Choose a Language..." v-model="userPreferdLanguage">
        <option value="Original">Original langauge</option>
        <option v-for="(value, index) in langList" :key="index" :value="index">{{ value.text }}</option>
      </select>
      <img src="@/assets/gear-light.svg" height="17" style="margin-left:10px; vertical-align: bottom; cursor: pointer;"
        @click="openSettingPage">
    </div>
    <div class="loading" v-if="generateSummaryLoading">
      <img src="@/assets/loading.gif" width="64">
    </div>
    <div v-if="!generateSummaryLoading && !!summaryContent">
      <h1>
        {{ topic }}
      </h1>
      <div>
        <vue-markdown :source="summaryContent" />
      </div>
      <h3 v-if="questions?.length > 0 || questionsLoading" class="qaheader">
        <img src="@/assets/circle-question-solid.svg" height="16" style="vertical-align: bottom;"> Q&A
      </h3>
      <div class="answer-loading" v-if="answerLoading">
        <img src="@/assets/loading.gif" width="32">
      </div>
      <div class="answer" v-if="answer !== ''">
        <vue-markdown :source="answer" />
      </div>
      <div class="questions-loading" v-if="questionsLoading">
        <img src="@/assets/loading.gif" width="32">
        <p>Generating questions...</p>
      </div>
      <ul class="question-list" v-else>
        <li v-for="question of questions" :key="question" @click="askQuestion(question)" class="question-item">
          {{ question }}
        </li>
      </ul>
    </div>
  </div>
  <footer>
    Powered by <a href="https://thangman22.com" target="_blank">@thangman22</a>
  </footer>
</template>

<script setup>
import { onMounted, ref, watch, toRef } from 'vue'
import { extractFromHtml } from '@extractus/article-extractor'
import VueMarkdown from 'vue-markdown-render'
import languageList from '@/assets/lang.json'

// --- State Variables ---
const isChromeExtension = ref(false)
const activeTab = ref(null)
const contentObject = ref(null)
const url = ref(null)
const summarizerModel = ref(null)
const summarizerModelStatus = ref(null)
const summaryContent = ref(null)
const topic = ref(null)
const generateSummaryLoading = ref(true)
const questionsLoading = ref(false)
const questions = ref([])
const firstLoad = ref(true)
const contentBody = ref(null)
const answer = ref('')
const answerLoading = ref(false)
const canSummarize = ref(false)
const canPrompt = ref(false)
const detectedLanguage = ref(false)
const userPreferdLanguage = ref('Original')
const langList = toRef(languageList)
const loaded = ref(false)

// --- Utility: Translation ---
async function translateText(originalText, fromLanguage, toLanguage) {
  try {
    if (!originalText) return ''
    if (toLanguage === 'Original') toLanguage = detectedLanguage.value
    if (fromLanguage === toLanguage) return originalText
    if (await isTranslationAvailable(fromLanguage, toLanguage)) {
      const translator = await window.Translator.create({
        sourceLanguage: fromLanguage || 'en',
        targetLanguage: toLanguage || 'en',
      })
      const translation = await translator.translate(originalText)
      console.log(`[translateText] Translated from ${fromLanguage} to ${toLanguage}:`, translation)
      return translation
        .replaceAll(' ### ', '\n### ')
        .replaceAll(' ## ', '\n## ')
        .replaceAll(' # ', '\n# ')
        .replaceAll(' * ', ' \n* ')
        .replaceAll(' - ', ' \n- ')
    }
    return originalText
  } catch (err) {
    console.error('[translateText] Error:', err)
    return originalText
  }
}

async function isTranslationAvailable(fromLanguage, toLanguage) {
  const canTranslate = await window.Translator.availability({
    sourceLanguage: fromLanguage || 'en',
    targetLanguage: toLanguage || 'en',
  })
  return canTranslate === 'available'
}

// --- Utility: Language Detection ---
async function detectLanguage(text) {
  try {
    if (!text) throw new Error('No text provided for language detection')
    if (!('LanguageDetector' in self)) return 'en'
    const availability = await window.LanguageDetector.availability()
    if (availability === 'unavailable') return 'en'
    const detector = await window.LanguageDetector.create()
    const results = await detector.detect(text)
    const detected = results[0]?.detectedLanguage || 'en'
    console.log(`[detectLanguage] Detected language: ${detected}`)
    return detected
  } catch (err) {
    console.error('[detectLanguage] Error:', err)
    return 'en'
  }
}

// --- Utility: Chrome Extension Check ---
async function checkIfChromeExtension(activeInfo) {
  const tab = await chrome.tabs.get(activeInfo.tabId)
  isChromeExtension.value = !tab?.url
}

// --- Utility: DOM Extraction ---
function DOMtoString(selector) {
  const el = selector ? document.querySelector(selector) : document.documentElement
  return el ? el.outerHTML : 'ERROR: querySelector failed to find node'
}

// --- Content Loading ---
async function loadContentFromActiveTab() {
  try {
    const tabs = await chrome.tabs.query({ active: true, currentWindow: true })
    if (!tabs || !tabs[0]) throw new Error('No active tab found')
    activeTab.value = tabs[0]
    const results = await chrome.scripting.executeScript({
      target: { tabId: activeTab.value.id },
      injectImmediately: true,
      func: DOMtoString,
      args: ['html'],
    })
    url.value = activeTab.value.url
    if (!results || !results[0] || !results[0].result) throw new Error('No DOM result')
    return await extractFromHtml(results[0].result, activeTab.value.url)
  } catch (err) {
    console.error('[loadContentFromActiveTab] Error:', err)
    return null
  }
}

// --- Summarizer Model ---
async function loadSummaryModel() {
  try {
    if (canSummarize.value && canSummarize.value.available !== 'no') {
      let summarizer
      if (canSummarize.value.available === 'available') {
        summarizer = await window.Summarizer.create({
          type: 'key-points',
          format: 'markdown',
          length: 'long',
        })
      } else {
        summarizer = await window.Summarizer.create({
          monitor(m) {
            m.addEventListener('downloadprogress', (e) => {
              console.log(`Downloaded summarizer ${e.loaded * 100}%`)
            })
          },
        })
        await summarizer.ready
      }
      summarizerModelStatus.value = 'ready'
      return summarizer
    } else {
      summarizerModelStatus.value = 'not-ready'
      return false
    }
  } catch (err) {
    console.error('[loadSummaryModel] Error:', err)
    summarizerModelStatus.value = 'not-ready'
    return false
  }
}

// --- Question Generation ---
async function generateQuestions() {
  try {
    let session
    if (canPrompt.value.available === 'readily') {
      session = await window.LanguageModel.create()
    } else {
      session = await window.LanguageModel.create({
        monitor(m) {
          m.addEventListener('downloadprogress', (e) => {
            console.log(`Downloaded language model ${e.loaded * 100}%`)
          })
        },
      })
    }
    const prompt = `Please suggest me a short 5 question in ${langList.value[detectedLanguage.value].label} language of the content that will be a good prompt for Gemini in JSON exampale output\n      Example Output:\n      [\n          "Question 1",\n          "Question 2",\n          "Question 3",\n          "Question 4",\n          "Question 5"\n      ]\n      . This is full article ${contentBody.value}`
    const result = await session.prompt(prompt, {
      responseConstraint: {
        type: 'array',
        items: {
          type: 'string',
          description: 'A question about the article content',
        },
        minItems: 5,
        maxItems: 5,
        description: 'Array of 5 questions about the article content',
      },
    })
    const parsed = JSON.parse(result)
    console.log('[generateQuestions] Questions generated:', parsed)
    return parsed
  } catch (error) {
    console.error('[generateQuestions] Error:', error)
    return []
  }
}

// --- State Reset ---
function resetResult() {
  summaryContent.value = null
  contentObject.value = null
  topic.value = null
  answer.value = ''
  questions.value = []
}

// --- Main Summary Loading ---
async function loadSummary() {
  resetResult()
  let errorMessage = ''
  try {
    contentObject.value = await loadContentFromActiveTab()
    if (!contentObject.value) throw new Error('No content object')
    console.log('[loadSummary] Content object:', contentObject.value)
    let localTopic = contentObject.value.title
    contentBody.value = (contentObject.value.content || '').replace(/(<([^>]+)>)/gi, '').slice(0, 20000)
    if (!contentBody.value) throw new Error('No content body')
    const lang = await detectLanguage(contentBody.value)
    detectedLanguage.value = lang
    if (canSummarize.value !== false) {
      summarizerModel.value = await loadSummaryModel()
      if (!summarizerModel.value) throw new Error('Summarizer model not available')
      let summary
      try {
        summary = await summarizerModel.value.summarize(contentBody.value)
        console.log('[loadSummary] Summary generated:', summary)
      } catch (e) {
        console.error('[loadSummary] Summarization error:', e)
        summary = contentBody.value
      }
      setTimeout(async () => {
        try {
          const detectedSummaryLanguage = await detectLanguage(summary)
          if (userPreferdLanguage.value !== detectedSummaryLanguage) {
            summary = await translateText(summary, detectedSummaryLanguage, userPreferdLanguage.value)
          }
          document.startViewTransition(() => {
            summaryContent.value = summary
          })
        } catch (err) {
          console.error('[loadSummary] Error in summary translation:', err)
          document.startViewTransition(() => {
            summaryContent.value = 'An error occurred while translating the summary.'
          })
        }
      }, 0)
      setTimeout(async () => {
        try {
          const detectedTopicLanguage = await detectLanguage(localTopic)
          if (userPreferdLanguage.value !== detectedTopicLanguage) {
            localTopic = await translateText(localTopic, detectedTopicLanguage, userPreferdLanguage.value)
          }
          topic.value = localTopic
        } catch (err) {
          console.error('[loadSummary] Error in topic translation:', err)
          topic.value = 'An error occurred while translating the topic.'
        }
      }, 0)
      setTimeout(async () => {
        try {
          questionsLoading.value = true
          let questionResponse = await generateQuestions()
          if (userPreferdLanguage.value !== 'Original' && userPreferdLanguage.value !== detectedLanguage.value) {
            const questionResponsePromise = questionResponse.map(q => translateText(q))
            questionResponse = await Promise.all(questionResponsePromise)
          }
          console.log('[loadSummary] Questions after translation (if any):', questionResponse)
          document.startViewTransition(() => {
            questions.value = questionResponse
          })
        } catch (err) {
          console.error('[loadSummary] Error in question generation:', err)
          document.startViewTransition(() => {
            questions.value = ['An error occurred while generating questions.']
          })
        } finally {
          questionsLoading.value = false
        }
      }, 0)
    }
  } catch (err) {
    console.error('[loadSummary] Error:', err)
    errorMessage = 'An error occurred while loading the summary. Please try again.'
    document.startViewTransition(() => {
      summaryContent.value = errorMessage
    })
  } finally {
    document.startViewTransition(() => {
      generateSummaryLoading.value = false
    })
  }
}

// --- Q&A: Ask Question ---
async function askQuestion(question) {
  answer.value = ''
  let answerFromReq = null
  document.startViewTransition(() => {
    answerLoading.value = true
  })
  try {
    const lang = await detectLanguage(summaryContent.value)
    if (lang === 'en' && canPrompt.value !== false) {
      let session
      if (canPrompt.value.available === 'readily') {
        session = await window.LanguageModel.create()
      } else {
        session = await window.LanguageModel.create({
          monitor(m) {
            m.addEventListener('downloadprogress', (e) => {
              console.log(`Downloaded language model ${e.loaded * 100}%`)
            })
          },
        })
      }
      const prompt = `Based on this is an article "${summaryContent.value}"  please answer this question "${question}". Response must less than 140 charecter and in ${langList.value[detectedLanguage.value].label} language`
      const result = await session.prompt(prompt)
      answerFromReq = result
      console.log('[askQuestion] Answer generated:', answerFromReq)
    }
    if (userPreferdLanguage.value !== 'Original' && userPreferdLanguage.value !== detectedLanguage.value) {
      answerFromReq = await translateText(answerFromReq)
    }
    document.startViewTransition(() => {
      answer.value = answerFromReq
    })
  } catch (err) {
    console.error('[askQuestion] Error:', err)
    document.startViewTransition(() => {
      answer.value = 'An error occurred while generating the answer.'
    })
  } finally {
    document.startViewTransition(() => {
      answerLoading.value = false
    })
  }
}

// --- Reload Handler ---
function reload() {
  window.location.reload()
}

// --- Lifecycle: onMounted ---
onMounted(async () => {
  chrome.tabs.onActivated.addListener(async (activeInfo) => {
    await checkIfChromeExtension(activeInfo)
  })
  document.body.addEventListener('click', function (e) {
    if (e.target.tagName === 'A' && e.target.target === '_blank') {
      e.preventDefault()
      chrome.tabs.create({ url: e.target.href, active: true })
      return false
    }
  })
  const data = await chrome.storage.local.get('userPreferdLanguage')
  if (data.userPreferdLanguage) {
    userPreferdLanguage.value = data.userPreferdLanguage
  }
  if ('Summarizer' in self) {
    canSummarize.value = await window.Summarizer.availability()
  } else {
    canSummarize.value = false
  }
  const { available } = await window.LanguageModel.availability()
  if (available !== 'no') {
    canPrompt.value = { available: available }
  } else {
    canPrompt.value = { available: 'no' }
  }
  loaded.value = true
  loadSummary()
  firstLoad.value = false
  chrome.tabs.onActivated.addListener(async () => {
    loadSummary()
  })
  chrome.tabs.onUpdated.addListener((tabId, changeInfo) => {
    if (changeInfo.status === 'complete') {
      loadSummary(tabId)
    }
  })
})

// --- Watchers ---
watch(userPreferdLanguage, async () => {
  await chrome.storage.local.set({ userPreferdLanguage: userPreferdLanguage.value })
  loadSummary()
})
</script>

<style>
#main_app {
  color: #FAF9F6;
  padding: 10px
}

.loading {
  view-transition-name: loading;
  text-align: center;
}

.question-list {
  view-transition-name: question-list;
  list-style-type: none;
  padding: 0;
}

.question-item {
  color: #cbd5e1;
  padding: 8px;
  margin-bottom: 10px;
  cursor: pointer;
  border-bottom: #cbd5e1 1px dotted;
}

h1 {
  color: #0ea5e9;
}

.save-button {
  margin-top: 15px;
  background: #0ea5e9;
  color: #FFF;
  font-weight: bold;
  border: 0;
  padding: 10px 10px;
  width: calc(100%);
  border-radius: 6px;
  box-sizing: border-box;
}

.qaheader {
  view-transition-name: qaheader;
  border-top: #FFF solid 1.5px;
  padding-top: 15px;
  margin-bottom: 15px;
}

.answer {
  view-transition-name: answer;
  padding: 10px;
  background: #cbd5e126;
  color: #cbd5e1;
  border-radius: 10px;
}

.setting {
  color: #FFF;
  padding: 10px;
}

.setting-input {
  background: #cbd5e126;
  color: #cbd5e1;
  border: 0;
  padding: 10px 10px;
  width: calc(100%);
  border-radius: 6px;
  box-sizing: border-box;
}

.answer-loading {
  view-transition-name: answer-loading;
  text-align: center;
}

.not-support {
  text-align: center;
}

.reload-btn {
  background: transparent;
  color: #0ea5e9;
  border: 1px solid #0ea5e9;
  padding: 10px 15px;
  border-radius: 5px;
  cursor: pointer;
  margin-top: 10px;
}

.lang-selection-area {
  text-align: right;
}

.lang-selection-area>select {
  border: 0;
  background: transparent;
  color: #cbd5e1;
}

.warning {
  background: rgb(254 243 199);
  color: rgb(217 119 6);
  padding: 10px;
  border-radius: 5px;
  border: 1px solid rgb(217 119 6);
}

.fail {
  margin-top: 10px;
  background: rgb(254 226 226);
  color: rgb(153 27 27);
  padding: 10px;
  border-radius: 5px;
  border: 1px solid rgb(153 27 27);
}

footer {
  position: fixed;
  bottom: 0;
  padding: 10px;
  color: #cbd5e18d;
}

footer>a {
  color: #cbd5e18d;
}

.questions-loading {
  text-align: center;
  margin: 20px 0;
  color: #cbd5e1;
}

.questions-loading img {
  margin-bottom: 10px;
}

.questions-loading p {
  margin: 0;
  font-size: 14px;
}
</style>
