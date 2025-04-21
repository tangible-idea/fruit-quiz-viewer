<script setup>
import { ref, onMounted, computed } from 'vue'
import { useRoute } from 'vue-router'
import { supabase } from '../lib/supabaseClient'

const route = useRoute()
const roomId = computed(() => route.params.roomId)
const users = ref([])
const loading = ref(true)
const error = ref(null)

const fetchUserData = async () => {
  try {
    loading.value = true
    error.value = null
    
    const { data, error: supabaseError } = await supabase
      .from('quiz_score')
      .select('*')
      .eq('room_id', roomId.value)
      
    if (supabaseError) throw supabaseError
    
    users.value = data || []
    console.log('Fetched users:', users.value)
  } catch (err) {
    console.error('Error fetching data:', err)
    error.value = '데이터를 불러오는 중 오류가 발생했습니다.'
  } finally {
    loading.value = false
  }
}

const parseFruitEmojis = (fruitText) => {
  if (!fruitText) return { heartCount: 0, fruits: [] }
  
  // Extract heart count
  const heartMatch = fruitText.match(/💛x(\d+)/)
  const heartCount = heartMatch ? parseInt(heartMatch[1]) : 0
  
  // Remove the heart count part from the text
  const textWithoutHearts = fruitText.replace(/💛x\d+\s*/, '')
  
  // Extract all emoji characters
  const emojiRegex = /\p{Emoji_Presentation}|\p{Extended_Pictographic}/gu
  const fruits = textWithoutHearts.match(emojiRegex) || []
  
  return { heartCount, fruits }
}

const getFruitCounts = (user) => {
  const { fruits } = parseFruitEmojis(user.items)
  if (!fruits.length) return []
  
  // Count occurrences of each fruit
  const counts = {}
  fruits.forEach(fruit => {
    counts[fruit] = (counts[fruit] || 0) + 1
  })
  
  // Convert to array and sort by count
  return Object.entries(counts)
    .map(([fruit, count]) => ({ fruit, count }))
    .sort((a, b) => b.count - a.count)
}

const getTotalFruits = (user) => {
  const { fruits } = parseFruitEmojis(user.items)
  return fruits.length
}

const getFruitColors = (fruit) => {
  const colorMap = {
    '🍎': '#e74c3c', // 빨간 사과
    '🍏': '#2ecc71', // 초록 사과
    '🍊': '#f39c12', // 오렌지
    '🍋': '#f1c40f', // 레몬
    '🍌': '#f1c40f', // 바나나
    '🍉': '#e74c3c', // 수박
    '🍇': '#9b59b6', // 포도
    '🍓': '#e74c3c', // 딸기
    '🍒': '#c0392b', // 체리
    '🍑': '#e67e22', // 복숭아
    '🍍': '#f39c12', // 파인애플
    '🥝': '#27ae60', // 키위
    '🥭': '#e67e22', // 망고
    '🥥': '#95a5a6', // 코코넛
    '🌽': '#f1c40f', // 옥수수
    '🍅': '#e74c3c', // 토마토
    '🍈': '#1abc9c', // 멜론
    '🥑': '#27ae60', // 아보카도
    '🌷': '#e84393', // 튤립
    '🍐': '#2ecc71'  // 배
  }
  
  return colorMap[fruit] || '#3498db' // 기본색상
}

onMounted(() => {
  fetchUserData()
})
</script>

<template>
  <div class="farm-view">
    <div class="farm-header">
      <h1>🌳 농장 {{ roomId }} 과일 나무 🌳</h1>
    </div>
    
    <div v-if="loading" class="loading">
      <div class="loader"></div>
      <p>나무에서 과일을 수확하는 중...</p>
    </div>
    
    <div v-else-if="error" class="error">
      {{ error }}
    </div>
    
    <div v-else-if="users.length === 0" class="empty">
      <p>이 농장에는 아직 나무가 심어지지 않았어요.</p>
    </div>
    
    <div v-else class="farm-container">
      <div v-for="user in users" :key="user.id" class="tree-container">
        <div class="tree-top">
          <div class="user-name">{{ user.user_name || '이름 없음' }}</div>
          <div v-if="user.score" class="score-tag">{{ user.score }}점</div>
        </div>
        
        <div class="tree">
          <div class="tree-trunk"></div>
          <div class="tree-crown">
            <div v-for="(fruitItem, fruitIndex) in getFruitCounts(user)" 
                 :key="fruitIndex" 
                 class="fruit-item"
                 :style="{
                   '--fruit-color': getFruitColors(fruitItem.fruit),
                   '--fruit-size': `${Math.min(1 + (fruitItem.count * 0.15), 2.5)}rem`,
                   '--fruit-left': `${5 + (fruitIndex * 18) % 75}%`,
                   '--fruit-top': `${10 + ((fruitIndex * 23) % 65)}%`,
                 }">
              <div class="fruit-emoji">{{ fruitItem.fruit }}</div>
              <div class="fruit-count">{{ fruitItem.count }}</div>
            </div>
          </div>
        </div>
        
        <div class="tree-info">
          <span>총 {{ getTotalFruits(user) }}개의 과일</span>
        </div>
      </div>
    </div>
    
    <div class="back-button">
      <router-link to="/">← 농장으로 돌아가기</router-link>
    </div>
  </div>
</template>

<style scoped>
.farm-view {
  max-width: 100%;
  margin: 0 auto;
  padding: 1rem;
  background-color: #e8f5e9;
  background-image: linear-gradient(#c8e6c9 1px, transparent 1px),
                    linear-gradient(90deg, #c8e6c9 1px, transparent 1px);
  background-size: 20px 20px;
  min-height: 100vh;
}

.farm-header {
  text-align: center;
  margin-bottom: 1.5rem;
  background-color: rgba(255, 255, 255, 0.7);
  padding: 1rem;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
}

h1 {
  font-size: 1.5rem;
  color: #2e7d32;
  margin: 0;
}

.loading, .error, .empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 200px;
  text-align: center;
  background-color: rgba(255, 255, 255, 0.7);
  padding: 2rem;
  border-radius: 8px;
  margin: 1rem 0;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
}

.loader {
  width: 40px;
  height: 40px;
  border: 4px solid #4caf50;
  border-radius: 50%;
  border-top-color: transparent;
  animation: spin 1s linear infinite;
  margin-bottom: 1rem;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.error {
  color: #c62828;
}

.farm-container {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(160px, 1fr));
  gap: 1rem;
  justify-content: center;
}

@media (min-width: 640px) {
  .farm-container {
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 1.5rem;
  }
}

.tree-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  background-color: rgba(255, 255, 255, 0.7);
  border-radius: 12px;
  padding: 1rem;
  box-shadow: 0 3px 10px rgba(0, 0, 0, 0.08);
  transition: transform 0.2s;
}

.tree-container:hover {
  transform: translateY(-5px);
}

.tree-top {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  margin-bottom: 0.5rem;
  position: relative;
}

.user-name {
  font-weight: 700;
  font-size: 0.9rem;
  color: #1b5e20;
  text-align: center;
  max-width: 100%;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.score-tag {
  position: absolute;
  right: -8px;
  top: -8px;
  background-color: #f57f17;
  color: white;
  font-size: 0.75rem;
  font-weight: bold;
  border-radius: 50%;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.tree {
  position: relative;
  width: 100%;
  height: 180px;
  margin-bottom: 0.5rem;
}

.tree-trunk {
  position: absolute;
  bottom: 0;
  left: 50%;
  transform: translateX(-50%);
  width: 20px;
  height: 50px;
  background-color: #795548;
  border-radius: 0 0 5px 5px;
}

.tree-crown {
  position: absolute;
  bottom: 45px;
  left: 50%;
  transform: translateX(-50%);
  width: 120px;
  height: 135px;
  background-color: #81c784;
  border-radius: 70% 70% 60% 60%;
  z-index: 0;
}

.fruit-item {
  position: absolute;
  width: var(--fruit-size);
  height: var(--fruit-size);
  left: var(--fruit-left);
  top: var(--fruit-top);
  border-radius: 50%;
  background-color: var(--fruit-color);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  transition: transform 0.2s;
}

.fruit-item:hover {
  transform: scale(1.2);
  z-index: 10;
}

.fruit-emoji {
  font-size: calc(var(--fruit-size) * 0.8);
  line-height: 1;
  z-index: 2;
}

.fruit-count {
  position: absolute;
  right: -5px;
  bottom: -5px;
  background-color: white;
  color: #333;
  font-size: 0.65rem;
  font-weight: bold;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
  z-index: 3;
}

.tree-info {
  font-size: 0.8rem;
  color: #558b2f;
  text-align: center;
  font-weight: 600;
}

.back-button {
  margin-top: 2rem;
  text-align: center;
}

.back-button a {
  display: inline-block;
  padding: 0.5rem 1.2rem;
  background-color: #43a047;
  color: white;
  text-decoration: none;
  border-radius: 50px;
  font-weight: 600;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
  transition: background-color 0.2s, transform 0.2s;
}

.back-button a:hover {
  background-color: #2e7d32;
  transform: translateY(-2px);
}
</style>
