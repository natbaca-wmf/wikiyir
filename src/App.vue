<script setup>
import { ref, watch } from 'vue'
import { CdxLabel, CdxTextInput, CdxButton, CdxIcon, CdxSelect } from '@wikimedia/codex'
import { cdxIconArrowNext, cdxIconHeart } from '@wikimedia/codex-icons'
import { Carousel, Slide, Pagination, Navigation } from 'vue3-carousel'
import 'vue3-carousel/dist/carousel.css'

import { initMap, fetchData, TopArticlesAPI } from '@/services'
import {
  Header,
  ArticleSlide,
  ArticleHistoryChart,
  MostEditedArticles,
  BytesAdded,
  DonateToday,
  StickyFooter
} from '@/components'

const project = ref('en.wikipedia')
const year = ref('2024')
const articleData = ref(null)
const category = ref('')
const currentArticleIndex = ref(0)
const currentArticleHistoryData = ref(null)
const currentArticleTitle = ref('')

const bytesAdded = ref(0)
const numNewEditors = ref(0)
const numEditors = ref(0)
const numEdits = ref(0)

const mostEditedArticles = ref([])

const yearItems = [
  { label: '2024', value: '2024' },
  { label: '2023', value: '2023' },
  { label: '2022', value: '2022' },
  { label: '2021', value: '2021' },
  { label: '2020', value: '2020' }
]

const categoryItems = [
  { label: 'All', value: '' },
  { label: 'Passings', value: '0' },
  { label: 'Movies', value: '1' },
  { label: 'Science', value: '2' }
]

const toReadable = (num) => {
  return num.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ',')
}

const updateFromData = (data) => {
  bytesAdded.value = toReadable(data.bytes)
  numEdits.value = toReadable(data.numEdits)
  numEditors.value = toReadable(data.numEditors)
  numNewEditors.value = toReadable(data.numNewEditors)
  mostEditedArticles.value = data.mostEditedArticles || []
  console.log('Updated mostEditedArticles:', mostEditedArticles.value)
}

async function fetchArticles() {
  const data = await fetchData({ project: project.value, limit: 10, year: year.value })
  updateFromData(data)
  articleData.value = data
  window.articleData = articleData.value
}

/**
 * If a search already got carried out, changing dropdown will update the query.
 */
async function updateIfNewYearSelected(newYear) {
  if (articleData.value) {
    const data = await fetchData({ project: project.value, limit: 10, year: newYear })
    updateFromData(data)
    articleData.value = data
  }
}

const getSlideData = ref(() => {
  if (category.value) {
    return articleData.value.categorizedYearlyTopArticles[category.value]
  } else {
    return articleData.value.yearlyTopArticles
  }
})

async function getArticleHistoryData(currentArticleIndex) {
  try {
    const currentArticle = articleData.value.yearlyTopArticles[currentArticleIndex]
    currentArticleHistoryData.value = await TopArticlesAPI.getArticleHistory(
      currentArticle,
      project.value,
      year.value
    )
    currentArticleTitle.value = currentArticle.article.replace(/_/g, ' ')
    console.log('Article history data:', currentArticleHistoryData.value)
  } catch (error) {
    console.error('Error fetching article history data:', error)
    currentArticleHistoryData.value = null
    currentArticleTitle.value = ''
  }
}

watch(project, (newProject) => {
  initMap(newProject, year.value)
})

watch(year, (newYear) => {
  initMap(project.value, newYear)
})

watch(currentArticleIndex, async (newIndex) => {
  getArticleHistoryData(newIndex)
})

watch(articleData, async () => {
  getArticleHistoryData(currentArticleIndex.value)
  initMap(project.value, year.value)
})

watch(currentArticleHistoryData, (newData) => {
  console.log('Current article history data updated:', newData)
})
</script>

<template>
  <div id="app">
    <Header :hasArticleData="!!articleData" />
    <main class="main-content">
      <section class="input-wrapper wrapper">
        <div>
          <cdx-label input-id="wiki-input"> Year</cdx-label>
          <cdx-select
            @update:selected="updateIfNewYearSelected"
            v-model:selected="year"
            :menu-items="yearItems"
            default-label="Choose an option"
            class="year-select"
          />
        </div>
        <div>
          <cdx-label input-id="wiki-input"> Wiki:</cdx-label>
          <cdx-text-input
            required
            :disabled="articleData !== null"
            pattern="[^\.]*\.(wikivoyage|wikinews|wikiversity|wikibooks|wikiquote|wiktionary|wikifunctions|wikisource|wikipedia|mediawiki|wikidata|wikimedia)"
            type="text"
            v-model="project"
            id="wiki-input"
            class="wiki-input"
          ></cdx-text-input>
        </div>
        <div v-if="false">
          <cdx-label input-id="filter-input"> Categories </cdx-label>
          <cdx-select
            v-model:selected="category"
            :menu-items="categoryItems"
            default-label="Filter"
            id="filter-select"
          />
        </div>
        <div>
          <cdx-button class="submit-btn" @click="fetchArticles" action="progressive" weight="primary">
            <cdx-icon class="nextIcon" :icon="cdxIconArrowNext"></cdx-icon>
            <span>Get Year in Review</span>
          </cdx-button>
        </div>
      </section>
      <section class="top-articles" v-if="articleData">
        <div class="wrapper">
          <h2 class="top-article-heading">Most visited articles in {{ year }}</h2>
          <carousel :items-to-show="1" v-model="currentArticleIndex">
            <slide v-for="(slide, i) in getSlideData()" :key="i">
              <article-slide
                :data="slide"
                :index="i + 1"
                :isCurrent="i === currentArticleIndex"
              ></article-slide>
            </slide>
            <template #addons>
              <navigation />
              <pagination />
            </template>
          </carousel>
        </div>
      </section>
      <section class="timeline-chart wrapper" v-if="currentArticleHistoryData">
        <div v-if="currentArticleHistoryData === null">Loading chart data...</div>
        <article-history-chart
          v-else
          :dataset="currentArticleHistoryData"
          :title="currentArticleTitle"
        ></article-history-chart>
      </section>
      <section :class="{ wrapper: true, 'map-hidden': !articleData }">
        <div id="map" class="map" ref="map"></div>
      </section>
      <section v-if="articleData" class="editor-stats wrapper">
        <div>
          <dl>
            <dt>{{ numEditors }}</dt>
            <dd>editors this year</dd>
          </dl>
          <dl>
            <dt>{{ numNewEditors }}</dt>
            <dd>new editors</dd>
          </dl>
          <dl>
            <dt>{{ numEdits }}</dt>
            <dd>edits</dd>
          </dl>
          <dl>
            <dt>{{ bytesAdded }}</dt>
            <dd>bytes added</dd>
          </dl>
        </div>
        <img src="./assets/community.svg" width="300" />
      </section>
      <section v-if="articleData" class="edit-stats wrapper">
        <div>
          <MostEditedArticles :data="mostEditedArticles" />
        </div>
      </section>
      <section v-if="articleData" class="bytes-stats wrapper">
        <BytesAdded :data="bytesAdded" />
      </section>
      <section v-if="articleData" class="donate-today wrapper">
        <DonateToday />
      </section>
    </main>

    <StickyFooter />
  </div>
</template>

<style lang="css" src="@/App.vue.css"></style>

<style>
#app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

/* Ensure main content takes up available space */
.main-content {
  flex: 1 0 auto;
  padding-bottom: 60px; /* Adjust this value based on your footer height */
}

/* You may need to adjust other global styles */
body {
  margin: 0;
  padding: 0;
}
</style>