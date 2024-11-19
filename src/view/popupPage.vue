<template>
  <div class="fail" v-if="isChromeExtension">
      This extension not support on extension page
  </div>
  <form class="setting" v-if="SettingPageOpened && !isChromeExtension" @submit.prevent="saveSetting">
    <div style="text-align: center; margin-bottom: 10px;">
      <img src="@/assets/logo.png" width="50%">
    </div>
    <div class="warning">
      To use ITLIDR, you'll need to obtain a Gemini API key. You can find your key in <a href="https://aistudio.google.com/" target="_blank" style="color: rgb(217 119 6);">AI Studio.</a>
    </div>
    <div class="fail" v-if="geminiKeyFail">
      Gemini API key is invalid please check your key
    </div>
    <div>
      <h3>Gemini key</h3>
      <input type="password" class="setting-input" v-model="geminiKeyOnForm" required>
      <button class="save-button" type="submit">{{ saveSettingText }}</button>
    </div>
  </form>
  <div id="main_app" v-if="loaded && !SettingPageOpened && !isChromeExtension">
    <div class="not-support" v-if="!generateSummaryLoading && !summaryContent && geminiKey">
      This website is not currently supported. Please try a different page or retry<br>
      <button class="reload-btn" @click="reload"><img src="@/assets/rotate-right-regular.svg"
          style="display:inline; height:15px; margin-right:5px;vertical-align: bottom;">Retry</button>
    </div>
    <div class="lang-selection-area" v-if="!generateSummaryLoading && !!summaryContent">
      <select data-placeholder="Choose a Language..." v-model="userPreferdLanguage">
        <option value="Original">Original langauge</option>
        <option v-for="(value, index) in langList" :key="index" :value="index">{{ value.text }}</option>
      </select>
      <img src="@/assets/gear-light.svg" height="17" style="margin-left:10px; vertical-align: bottom; cursor: pointer;" @click="openSettingPage">
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
      <h3 v-if="questions?.length > 0" class="qaheader">
        <img src="@/assets/circle-question-solid.svg" height="16" style="vertical-align: bottom;"> Q&A
      </h3>
      <div class="answer-loading" v-if="answerLoading">
        <img src="@/assets/loading.gif" width="32">
      </div>
      <div class="answer" v-if="answer !== ''">
        <vue-markdown :source="answer" />
      </div>
      <ul class="question-list">
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
import { loadModule } from 'cld3-asm';
import VueMarkdown from 'vue-markdown-render'
import languageList from '@/assets/lang.json'
import { GoogleGenerativeAI, HarmBlockThreshold, HarmCategory } from '@google/generative-ai';
import extract from 'extract-json-from-string';
let cldFactory
let model = null
const isChromeExtension = ref(false)
const activeTab = ref(null)
const contentObject = ref(null)
const url = ref(null)
const summarizerModel = ref(null)
const summarizerModelStatus = ref(null)
const summaryContent = ref(null)
const topic = ref(null)
const generateSummaryLoading = ref(true)
const translationModel = ref(null)
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
const geminiKey = ref(null)
const geminiKeyFail = ref(false)
const geminiKeyOnForm = ref(null)
const saveSettingText = ref('Save setting')
const loaded = ref(false)
const SettingPageOpened = ref(false)
const openSettingPage = () => {
  SettingPageOpened.value = true
}

const getModel = (apiKey) => {
  const safetySettings = [
    {
      category: HarmCategory.HARM_CATEGORY_SEXUALLY_EXPLICIT,
      threshold: HarmBlockThreshold.BLOCK_NONE,
    },
    {
      category: HarmCategory.HARM_CATEGORY_HATE_SPEECH,
      threshold: HarmBlockThreshold.BLOCK_NONE,
    },
    {
      category: HarmCategory.HARM_CATEGORY_HARASSMENT,
      threshold: HarmBlockThreshold.BLOCK_NONE,
    },
    {
      category: HarmCategory.HARM_CATEGORY_DANGEROUS_CONTENT,
      threshold: HarmBlockThreshold.BLOCK_NONE,
    }
  ];

  const genAI = new GoogleGenerativeAI(apiKey);
  return genAI.getGenerativeModel({
    safetySettings: safetySettings,
    temperature: 0,
    model: "gemini-1.5-flash-8b",
    systemInstruction: "Act like a journalist",
  });
};

const verifyGeminiApiKey = async (apiKey) => {
  const API_VERSION = 'v1';
  const apiUrl = `https://generativelanguage.googleapis.com/${API_VERSION}/models?key=${apiKey}`;
  const response = await fetch(apiUrl, {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
    muteHttpExceptions: true,
  });

  if (!response.ok) {
    return false
  }

  return true;
};

const translateText = async (origialText) => {
  if (await ableToTranslate()) {
    const translator = await window.translation.createTranslator({
      sourceLanguage: detectedLanguage.value || 'en',
      targetLanguage: userPreferdLanguage.value || 'en',
    });
    const translation = await translator.translate(origialText);
    return translation.replaceAll(' ### ', '\n### ').replaceAll(' ## ', '\n## ').replaceAll(' # ', '\n# ').replaceAll(' * ', ' \n* ').replaceAll(' - ', ' \n- ')
  } else {
    try {
      const prompt = `Please translate this text from ${langList.value[detectedLanguage.value].label} to ${langList.value[userPreferdLanguage.value].label} language. This is full article ${origialText}. Please response in JSON format example response {"result": "translated text"}`;
      const result = await model.generateContent(prompt);

      const geminiResponse = await result.response;
      const text = geminiResponse.text();
      let jsonObject = extract(text);
      let translateResult = jsonObject[0]
      if (!translateResult) {
        return origialText
      }
      return translateResult.result
    } catch (error) {
      console.error(error)
      return origialText
    }
  }
}

const ableToTranslate = async () => {
  if(!window.translation) {
    return false
  }

  if(!window.translation.canTranslate) {
    return false
  }

  const canTranslate = await window.translation.canTranslate({
    sourceLanguage: detectedLanguage.value || 'en',
    targetLanguage: userPreferdLanguage.value || 'en',
  })

  return canTranslate !== 'no'
}

const saveSetting = async () => {
  saveSettingText.value = 'Saving...'
  const geminiVerifyStatus = await verifyGeminiApiKey(geminiKeyOnForm.value)
  if (geminiVerifyStatus) {
    await chrome.storage.local.set({ geminiKey: geminiKeyOnForm.value })
    reload()
  } else {
    geminiKeyFail.value = true
  }
}

onMounted(async () => {

  chrome.tabs.onActivated.addListener(async (activeInfo) => {
    await checkChromeExtension(activeInfo)
  });

  document.body.addEventListener('click', function(e) {
    if (e.target.tagName === 'A' && e.target.target === '_blank') {
      e.preventDefault();
      chrome.tabs.create({ url: e.target.href, active: true });
      return false;
    }
  });

  const geminiKeyFromStorage = await chrome.storage.local.get("geminiKey");
  geminiKey.value = geminiKeyFromStorage.geminiKey

  if (!geminiKey.value) {
    generateSummaryLoading.value = false
    SettingPageOpened.value = true
    return
  }else {
    geminiKeyOnForm.value = geminiKey.value
  }

  const data = await chrome.storage.local.get("userPreferdLanguage");
  model = getModel(geminiKey.value);

  // Prepare code until translation model is able to run on extension
  // translationModel.value = await loadLanguageDetectionModel()

  //Use CLD because langauge detection is still not work on extension
  cldFactory = await loadModule()
  translationModel.value = await cldFactory.create()

  if (data.userPreferdLanguage) {
    userPreferdLanguage.value = data.userPreferdLanguage
  }

  try {
    canSummarize.value = await window.ai.summarizer?.capabilities();
  } catch (error) {
    console.log(error)
  }

  try {
    canPrompt.value = await window.ai.languageModel.capabilities();
  } catch (error) {
    console.log(error)
  }

  loaded.value = true

  loadSummary()
  firstLoad.value = false
  
  chrome.tabs.onActivated.addListener(async () => {
    loadSummary()
  });

  chrome.tabs.onUpdated.addListener((tabId, changeInfo) => {
    if (changeInfo.status === 'complete') {
      loadSummary(tabId)
    }
  })
})

watch(userPreferdLanguage, async () => {
  await chrome.storage.local.set({ userPreferdLanguage: userPreferdLanguage.value })
  loadSummary()
})

const reload = async () => {
  window.location.reload();
}

const loadContent = async () => {
  const results = await chrome.tabs.query({ active: true, currentWindow: true }).then(function (tabs) {
    
    activeTab.value = tabs[0];
    var activeTabId = activeTab.value.id;

    return chrome.scripting.executeScript({
      target: { tabId: activeTabId },
      injectImmediately: true,
      func: DOMtoString,
      args: ['html']
    });
  })
  url.value = activeTab.value.url
  const article = await extractFromHtml(results[0].result, activeTab.value.url)
  return article
}

// const loadLanguageDetectionModel = async () => {
//   const canDetect = await window.translation.canDetect();
//   let detector;

//   if (canDetect !== 'no') {
//     if (canDetect === 'readily') {
//       detector = await window.translation.createDetector();
//     } else {
//       detector = await window.translation.createDetector();
//       await detector.ready;
//     }
//     return detector;
//   } else {
//     return false
//   }
// }

// const detectLanguage = async (text) => {
//   const langResult = await translationModel.value.detect(contentBody);
//   return langResult
// }

const detectLanguage = async (text) => {
  const { language } = await translationModel.value.findLanguage(text)
  return language
}

const loadSummaryModel = async () => {
  let summarizer;
  if (canSummarize.value && canSummarize.value.available !== 'no') {
    if (canSummarize.value.available === 'readily') {
      summarizer = await window.ai.summarizer.create();
    } else {
      summarizer = await window.ai.summarizer.create();
      summarizer.addEventListener('downloadprogress', () => {
        summarizerModelStatus.value = 'loading';
      });
      await summarizer.ready;
    }
    summarizerModelStatus.value = 'ready';
    return summarizer;
  } else {
    summarizerModelStatus.value = 'not-ready';
    return false
  }
}

const loadQuestion = async () => {
  try {
    const prompt = `Please suggest me a short 5 question in ${langList.value[detectedLanguage.value].label} language of the content that will be a good prompt for Gemini in JSON exampale output
      Example Output:
      [
          "Question 1",
          "Question 2",
          "Question 3",
          "Question 4",
          "Question 5"
      ]
      . This is full article ${contentBody.value}`;
    const result = await model.generateContent(prompt);
    const geminiResponse = await result.response;
    const text = geminiResponse.text();
    let jsonObject = extract(text);
    let questions = jsonObject[0]

    return questions
  } catch (error) {
    console.error(error)
    return []
  }
}

const resetResult = () => {
  geminiKeyFail.value = false
  summaryContent.value = null
  contentObject.value = null
  topic.value = null
  answer.value = ''
  questions.value = []
}

const checkChromeExtension = async (activeInfo) => {
  const tab = await chrome.tabs.get(activeInfo.tabId);
  if(!tab?.url) {
    isChromeExtension.value = true
  } else {
    isChromeExtension.value = false
  }
}

const loadSummary = async () => {
  resetResult()
  contentObject.value = await loadContent()

  let localTopic = null
  let summary = null

  document.startViewTransition(async () => {
    generateSummaryLoading.value = true
  })

  if (contentObject.value) {
    localTopic = contentObject.value.title

    contentBody.value = contentObject.value.content.replace(/(<([^>]+)>)/gi, '')
    if (contentBody.value) {

      const lang = await detectLanguage(contentBody.value)
      detectedLanguage.value = lang

      if (lang === 'en' && canSummarize.value !== false) {

        if (summarizerModel.value) {
          summarizerModel.value.destroy();
          summarizerModel.value = null
        }

        summarizerModel.value = await loadSummaryModel()

        try {
          summary = await summarizerModel.value.summarize(contentBody.value)

        } catch (e) {
          resetResult()
        }
      } else {
        try {
          const plainContent = contentBody.value.replace(/(<([^>]+)>)/gi, "");
          const prompt = `This is an article or news from website. Please summary to 1 paragraph. The result MUST be in ${langList.value[detectedLanguage.value].label} language of the article. This is full article ${plainContent}. Please response in JSON Output Example {"summary":".........."}`;

          const result = await model.generateContent(prompt);

          const geminiResponse = await result.response;
          const text = geminiResponse.text();

          let jsonObject = extract(text);
          let summaryObject = jsonObject[0]

          summary = summaryObject.summary

        } catch (e) {
          resetResult()
        }
      }

      if (userPreferdLanguage.value !== 'Original' && userPreferdLanguage.value !== detectedLanguage.value) {
        summary = await translateText(summary)
        localTopic = await translateText(localTopic)
      }

      topic.value = localTopic

      document.startViewTransition(() => {
        summaryContent.value = summary
      });

      document.startViewTransition(() => {
        generateSummaryLoading.value = false
      });

      let questionResponse = await loadQuestion()

      if (userPreferdLanguage.value !== 'Original' && userPreferdLanguage.value !== detectedLanguage.value) {
        const questionResponsePromise = []
        questionResponse.forEach(q => {
          questionResponsePromise.push(translateText(q))
        })
        questionResponse = await Promise.all(questionResponsePromise)
      }

      document.startViewTransition(() => {
        questions.value = questionResponse
      });
    }
  }

}

const askQuestion = async (question) => {
  answer.value = ''
  let answerFromReq = null

  document.startViewTransition(() => {
    answerLoading.value = true
  });

  const lang = await detectLanguage(summaryContent.value)
  if (lang === 'en' && canPrompt.value !== false) {
    if (canPrompt.value.available !== "no") {
      const session = await window.ai.languageModel.create();
      const prompt = `Based on this is an article "${summaryContent.value}" "${question}". Response must less than 140 charecter`
      const result = await session.prompt(prompt);
      answerFromReq = result
    }
  } else {
    const prompt = `Based on this is an article or news ${contentBody.value} please answer this question "${question}"`;
    const result = await model.generateContent(prompt);
    const geminiResponse = await result.response;
    const text = geminiResponse.text();
    answerFromReq = text
  }

  if (userPreferdLanguage.value !== 'Original' && userPreferdLanguage.value !== detectedLanguage.value) {
    answerFromReq = await translateText(answerFromReq)
  }

  document.startViewTransition(() => {
    answer.value = answerFromReq
  });

  document.startViewTransition(() => {
    answerLoading.value = false
  });
}

const DOMtoString = (selector) => {
  if (selector) {
    selector = document.querySelector(selector);
    if (!selector) return "ERROR: querySelector failed to find node"
  } else {
    selector = document.documentElement;
  }
  return selector.outerHTML;
}

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
footer{
  position: fixed;
  bottom: 0;
  padding: 10px;
  color:#cbd5e18d;
}
footer > a {
  color:#cbd5e18d;
}
</style>
