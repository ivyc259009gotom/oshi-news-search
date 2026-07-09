<script setup>
import { ref, onMounted, computed } from 'vue'

// 検索キーワード
const keyword = ref('')

// 言語選択
const searchTarget = ref('ja')

// 検索対象
const searchTargets = [
  {
    label: '日本語',
    value: 'ja',
    lang: 'ja',
    country: '',
  },
  {
    label: '英語',
    value: 'en',
    lang: 'en',
    country: '',
  },
]

// 記事一覧
const articles = ref([])

// 検索履歴
const searchHistory = ref([])

// 検索履歴をブラウザに保存する
const saveSearchHistory = () => {
  localStorage.setItem('searchHistory', JSON.stringify(searchHistory.value))
}

// お気に入り記事のURLを保存する配列
const favorites = ref([])

// お気に入りのみ表示するかどうか
const showFavoritesOnly = ref(false)

// お気に入りをブラウザに保存する
const saveFavorites = () => {
  localStorage.setItem('favorites', JSON.stringify(favorites.value))
}

// 読み込み中かどうか
const isLoading = ref(false)

// エラーメッセージ
const errorMessage = ref('')

// 検索したかどうか
const hasSearched = ref(false)

// GNews APIキー
const apiKey = import.meta.env.VITE_GNEWS_API_KEY

// サムネイルがない場合の代替画像
const noImageUrl = 'https://placehold.co/300x180?text=No+Image'

// GNews APIのURLを作る
const createGNewsUrl = (searchWord, target, useLang = true) => {
  let url = `https://gnews.io/api/v4/search?q=${encodeURIComponent(searchWord)}&max=10&sortby=publishedAt&in=title,description,content&apikey=${apiKey}`

  if (useLang && target.lang) {
    url += `&lang=${target.lang}`
  }

  if (target.country) {
    url += `&country=${target.country}`
  }

  return url
}

// ページ表示時に保存済みの検索履歴・お気に入りを読み込む
onMounted(() => {
  const savedHistory = localStorage.getItem('searchHistory')

  if (savedHistory) {
    searchHistory.value = JSON.parse(savedHistory)
  }

  const savedFavorites = localStorage.getItem('favorites')

  if (savedFavorites) {
    favorites.value = JSON.parse(savedFavorites)
  }
})

// 検索ボタンを押したときの処理
const searchNews = async () => {
  // キーワードが空の場合は検索しない
  if (keyword.value.trim() === '') {
    alert('検索キーワードを入力してください')
    return
  }

  // 入力されたキーワードの前後の空白を取り除く
  const searchWord = keyword.value.trim()

  // 検索開始時の状態
  isLoading.value = true
  errorMessage.value = ''
  articles.value = []
  hasSearched.value = true
  showFavoritesOnly.value = false

  // 検索履歴に同じキーワードがある場合は一度削除する
  searchHistory.value = searchHistory.value.filter((item) => item !== searchWord)

  // 検索履歴の先頭に追加する
  searchHistory.value.unshift(searchWord)

  // 検索履歴は最大5件までにする
  searchHistory.value = searchHistory.value.slice(0, 5)

  // 検索履歴をブラウザに保存する
  saveSearchHistory()

  try {
    // 選択中の検索対象を取得する
    const selectedTarget = searchTargets.find((target) => target.value === searchTarget.value)

    // GNews APIのURLを作る
    const url = createGNewsUrl(searchWord, selectedTarget, true)

    // APIにアクセスする
    const response = await fetch(url)

    // 通信に失敗した場合
    if (!response.ok) {
      throw new Error('API通信に失敗しました')
    }

    // JSON形式に変換する
    const data = await response.json()

    // 取得した記事をarticlesに入れる
    articles.value = data.articles || []
  } catch (error) {
    errorMessage.value = 'ニュースの取得に失敗しました。時間をおいて再度お試しください。'
    console.error(error)
  } finally {
    isLoading.value = false
  }
}

// 検索履歴をクリックしたときの処理
const searchFromHistory = (historyKeyword) => {
  keyword.value = historyKeyword
  searchNews()
}

// 検索条件と検索結果をリセットする
const resetSearch = () => {
  keyword.value = ''
  searchTarget.value = 'ja'
  articles.value = []
  errorMessage.value = ''
  hasSearched.value = false
  showFavoritesOnly.value = false
}

// 検索履歴をすべて削除する
const clearSearchHistory = () => {
  searchHistory.value = []
  localStorage.removeItem('searchHistory')
}

// お気に入りボタンを押したときの処理
const toggleFavorite = (articleUrl) => {
  if (favorites.value.includes(articleUrl)) {
    favorites.value = favorites.value.filter((url) => url !== articleUrl)
  } else {
    favorites.value.push(articleUrl)
  }

  // お気に入りの状態をブラウザに保存する
  saveFavorites()
}

// お気に入り済みかどうかを判定する
const isFavorite = (articleUrl) => {
  return favorites.value.includes(articleUrl)
}

// 画面に表示する記事一覧
const displayedArticles = computed(() => {
  if (showFavoritesOnly.value) {
    return articles.value.filter((article) => favorites.value.includes(article.url))
  }

  return articles.value
})

// 公開日を見やすい形に変換する
const formatDate = (dateString) => {
  if (!dateString) {
    return '日付不明'
  }

  const date = new Date(dateString)

  return date.toLocaleDateString('ja-JP', {
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
  })
}
</script>

<template>
  <div class="app">
    <h1>推しニュース検索</h1>

    <p class="app-description">日本語・英語のニュースから、推しに関する最新記事を検索できます。</p>

    <div class="search-area">
      <input v-model="keyword" type="text" placeholder="俳優名・作品名・アーティスト名を入力" />

      <select v-model="searchTarget">
        <option v-for="target in searchTargets" :key="target.value" :value="target.value">
          {{ target.label }}
        </option>
      </select>

      <button @click="searchNews" :disabled="isLoading">
        {{ isLoading ? '検索中...' : '検索' }}
      </button>

      <button class="reset-button" @click="resetSearch">リセット</button>
    </div>

    <div v-if="searchHistory.length > 0" class="history-area">
      <div class="history-header">
        <p>検索履歴</p>

        <button class="clear-history-button" @click="clearSearchHistory">履歴を削除</button>
      </div>

      <button
        v-for="historyKeyword in searchHistory"
        :key="historyKeyword"
        class="history-button"
        @click="searchFromHistory(historyKeyword)"
      >
        {{ historyKeyword }}
      </button>
    </div>

    <p v-if="isLoading" class="loading-message">読み込み中...</p>

    <p v-if="errorMessage" class="error-message">
      {{ errorMessage }}
    </p>

    <p
      v-if="hasSearched && !isLoading && !errorMessage && articles.length === 0"
      class="no-result-message"
    >
      検索結果がありません。
    </p>

    <p
      v-if="hasSearched && !isLoading && !errorMessage && articles.length > 0"
      class="result-count"
    >
      検索結果：{{ articles.length }}件
    </p>

    <div
      v-if="hasSearched && !isLoading && !errorMessage && articles.length > 0"
      class="filter-area"
    >
      <button :class="{ active: !showFavoritesOnly }" @click="showFavoritesOnly = false">
        すべての記事
      </button>

      <button :class="{ active: showFavoritesOnly }" @click="showFavoritesOnly = true">
        お気に入りのみ
      </button>
    </div>

    <p
      v-if="
        hasSearched &&
        !isLoading &&
        !errorMessage &&
        showFavoritesOnly &&
        displayedArticles.length === 0
      "
      class="no-result-message"
    >
      お気に入りに登録した記事はありません。
    </p>

    <div class="article-list">
      <div v-for="article in displayedArticles" :key="article.url" class="article-card">
        <img :src="article.image || noImageUrl" alt="記事画像" class="article-image" />

        <div class="article-content">
          <h2>{{ article.title }}</h2>

          <p class="source">
            {{ article.source.name }}
          </p>

          <p class="date">
            {{ formatDate(article.publishedAt) }}
          </p>

          <p class="description">
            {{ article.description }}
          </p>

          <a :href="article.url" target="_blank" rel="noopener noreferrer"> 記事を読む </a>
          <button class="favorite-button" @click="toggleFavorite(article.url)">
            {{ isFavorite(article.url) ? '♥ お気に入り済み' : '♡ お気に入り' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.app {
  max-width: 900px;
  margin: 0 auto;
  padding: 32px;
  font-family: sans-serif;
}

h1 {
  text-align: center;
  color: #3f51b5;
}

.search-area {
  display: flex;
  gap: 8px;
  margin: 24px 0;
}

input {
  flex: 1;
  padding: 10px;
  font-size: 16px;
}

select {
  padding: 10px;
  font-size: 16px;
}

button {
  padding: 10px 20px;
  font-size: 16px;
  background-color: #3f51b5;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

button:disabled {
  background-color: #aaa;
  cursor: not-allowed;
}

.article-list {
  display: grid;
  gap: 16px;
}

.article-card {
  display: flex;
  gap: 16px;
  padding: 16px;
  border: 1px solid #ddd;
  border-radius: 8px;
  background-color: white;
}

.article-image {
  width: 180px;
  height: 110px;
  object-fit: cover;
  border-radius: 6px;
}

.article-content h2 {
  margin: 0 0 8px;
  font-size: 20px;
}

.source,
.date {
  color: #666;
  font-size: 14px;
}

.description {
  margin: 8px 0;
  display: -webkit-box;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

a {
  color: #3f51b5;
  font-weight: bold;
}

.history-area {
  margin: 16px 0 24px;
}

.history-area p {
  margin-bottom: 8px;
  font-weight: bold;
}

.history-button {
  margin: 0 8px 8px 0;
  padding: 6px 12px;
  background-color: #eef0ff;
  color: #3f51b5;
  border: 1px solid #c5cae9;
  border-radius: 20px;
  cursor: pointer;
  font-size: 14px;
}

.favorite-button {
  margin-left: 12px;
  padding: 6px 12px;
  background-color: #fff0f5;
  color: #d81b60;
  border: 1px solid #f8bbd0;
  border-radius: 20px;
  cursor: pointer;
  font-size: 14px;
}

.loading-message {
  text-align: center;
  color: #3f51b5;
  font-weight: bold;
  margin: 24px 0;
}

.error-message {
  padding: 12px;
  background-color: #ffebee;
  color: #c62828;
  border: 1px solid #ef9a9a;
  border-radius: 6px;
  margin: 24px 0;
}

.no-result-message {
  text-align: center;
  color: #666;
  margin: 24px 0;
}

.result-count {
  margin: 16px 0;
  color: #555;
  font-weight: bold;
}

.history-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.clear-history-button {
  padding: 6px 10px;
  background-color: #f5f5f5;
  color: #555;
  border: 1px solid #ccc;
  border-radius: 4px;
  cursor: pointer;
  font-size: 13px;
}

.filter-area {
  display: flex;
  gap: 8px;
  margin: 12px 0 20px;
}

.filter-area button {
  padding: 8px 14px;
  background-color: #f5f5f5;
  color: #555;
  border: 1px solid #ccc;
  border-radius: 20px;
  cursor: pointer;
  font-size: 14px;
}

.filter-area button.active {
  background-color: #3f51b5;
  color: white;
  border-color: #3f51b5;
}

.reset-button {
  background-color: #757575;
}

.reset-button:hover {
  background-color: #616161;
}

.app-description {
  text-align: center;
  color: #666;
  margin-bottom: 24px;
}
</style>
