<template>
  <div class="fridge-view">
    <div class="room-header">
      <h1>🧊 {{ roomTitle }} 🧊</h1>
    </div>
    
    <div v-if="loading" class="loading">
      <div class="loader"></div>
      <p>냉장고 정보를 가져오는 중...</p>
    </div>
    
    <div v-else-if="error" class="error">
      {{ error }}
    </div>
    
    <div v-else-if="users.length === 0" class="empty">
      <p>이 방에는 아직 냉장고가 없어요.</p>
    </div>
    
    <div v-else class="fridges-container" ref="horizontalScrollContainer">
      <div v-for="user in users" :key="user.id" class="fridge-container">
        <div class="user-info">
          <div class="user-name">{{ user.user_name || '이름 없음' }}</div>
          <div v-if="user.score !== undefined" class="score-tag">{{ getUserRank(user) }}위</div>
        </div>
        
        <div 
          class="fridge" 
          :class="{ 'fridge-open': openFridges[user.id] }" 
          @click="toggleFridge(user.id)">
          <div class="fridge-door">
            <div class="fridge-handle"></div>
            <div class="fridge-info">
              <span>{{ getTotalFruits(user) }}개 보관 중</span>
            </div>
          </div>
          
          <div class="fridge-interior">
            <div class="fridge-light"></div>
            <div v-for="(fruitItem, fruitIndex) in getFruitCounts(user)" 
                 :key="fruitIndex" 
                 class="fruit-item"
                 :class="{ 
                   'fruit-active': openFridges[user.id],
                   'fruit-animate': activeFruitAnimation && activeFruitAnimation.userId === user.id 
                 }"
                 :style="{
                   '--fruit-color': getFruitColors(fruitItem.fruit),
                   '--fruit-size': `${Math.min(25 + (fruitItem.count * 2), 40)}px`,
                   ...getFruitPosition(user.id, fruitIndex)
                 }">
              <span class="fruit-emoji">{{ fruitItem.fruit }}</span>
              <span class="fruit-count">{{ fruitItem.count }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, onBeforeUnmount } from 'vue'
import { useRoute } from 'vue-router'
import { supabase } from '../lib/supabaseClient'

const route = useRoute()
const roomId = computed(() => route.params.roomId)
const users = ref([])
const loading = ref(true)
const error = ref(null)
const horizontalScrollContainer = ref(null)

// 냉장고 애니메이션 관련 상태
const openFridges = ref({}) // 냉장고 열림/닫힘 상태 저장
const activeFruitAnimation = ref(null)
const fruitPositions = ref({})

// 방 제목을 저장하는 변수
const roomTitle = ref('')

// 냉장고 열기/닫기 토글 함수
const toggleFridge = (userId) => {
  // 현재 냉장고 상태의 반대로 설정
  openFridges.value[userId] = !openFridges.value[userId]
  
  // 냉장고가 열리면 과일 애니메이션 활성화 (1-3위만)
  if (openFridges.value[userId]) {
    triggerFruitAnimation(userId)
  }
}

// 과일 애니메이션 트리거 함수
const triggerFruitAnimation = (userId) => {
  // 현재 사용자의 순위 가져오기
  const userIndex = users.value.findIndex(u => u.id === userId)
  
  // 1위, 2위, 3위만 애니메이션 활성화
  if (userIndex >= 0 && userIndex < 3) {
    activeFruitAnimation.value = { userId, rank: userIndex + 1 }
    
    // 애니메이션이 끝나면 상태 초기화
    setTimeout(() => {
      activeFruitAnimation.value = null
    }, 1500)
  }
}

// 방 제목 가져오기 함수
const fetchRoomTitle = async () => {
  try {
    // kakao_room 테이블에서 room_tag가 roomId인 room_title을 가져옵니다.
    const { data, error: supabaseError } = await supabase
      .from('kakao_room')
      .select('room_title')
      .eq('room_tag', roomId.value)
      .single()
    
    if (supabaseError) {
      console.error('Error fetching room title:', supabaseError)
      roomTitle.value = `🧊 ${roomId.value} 냉장고 🧊`
      return
    }
    
    // 가져온 데이터가 있으면 방 제목을 설정합니다. 없으면 기본 제목을 설정합니다.
    if (data && data.room_title) {
      roomTitle.value = `🧊 ${data.room_title} 냉장고 🧊`
    } else {
      roomTitle.value = `🧊 ${roomId.value} 냉장고 🧊`
    }
  } catch (err) {
    console.error('Error in fetchRoomTitle:', err)
    roomTitle.value = `🧊 ${roomId.value} 냉장고 🧊`
  }
}

// 사용자 데이터 가져오기
const fetchUserData = async () => {
  try {
    loading.value = true
    error.value = null
    
    const { data, error: supabaseError } = await supabase
      .from('quiz_score')
      .select('*')
      .eq('room_id', roomId.value)
      
    if (supabaseError) throw supabaseError
    
    // 점수에 따라 정렬된 사용자 배열
    users.value = data ? data.sort((a, b) => (b.score || 0) - (a.score || 0)) : []
    console.log('Fetched users:', users.value)
  } catch (err) {
    console.error('Error fetching data:', err)
    error.value = '데이터를 불러오는 중 오류가 발생했습니다.'
  } finally {
    loading.value = false
  }
}

// 사용자의 순위 계산
const getUserRank = (user) => {
  if (!users.value.length) return '-'
  const index = users.value.findIndex(u => u.id === user.id)
  return index !== -1 ? index + 1 : '-'
}

// 세로 스크롤을 가로 스크롤로 변환하는 함수
const handleVerticalScroll = (event) => {
  if (horizontalScrollContainer.value) {
    // 모바일 장치에서만 적용
    if (window.innerWidth <= 768) {
      event.preventDefault()
      horizontalScrollContainer.value.scrollLeft += event.deltaY
    }
  }
}

// 터치 이벤트에 대한 처리 추가
let startY = 0
let startX = 0
let lastY = 0
let isScrolling = false

const handleTouchStart = (e) => {
  if (window.innerWidth > 768) return
  startY = e.touches[0].clientY
  startX = e.touches[0].clientX
  lastY = startY
  isScrolling = false
}

const handleTouchMove = (e) => {
  if (!horizontalScrollContainer.value || window.innerWidth > 768) return
  
  const currentY = e.touches[0].clientY
  const currentX = e.touches[0].clientX
  
  // Y축 변화량이 X축보다 크면 (세로 스크롤 의도)
  if (!isScrolling) {
    isScrolling = Math.abs(currentY - startY) > Math.abs(currentX - startX)
  }
  
  if (isScrolling) {
    // 세로 스크롤 방지
    e.preventDefault()
    
    // 세로 스크롤 양 계산
    const moveY = lastY - currentY
    
    // 가로 스크롤 이동 (속도 조절 가능)
    horizontalScrollContainer.value.scrollLeft += moveY * 3
    
    // 마지막 스크롤 위치 저장
    lastY = currentY
  }
}

const handleTouchEnd = () => {
  isScrolling = false
}

// 과일 위치 계산 함수 - 처음 한 번만 계산하고 저장
const getFruitPosition = (userId, fruitIndex) => {
  const key = `${userId}-${fruitIndex}`
  
  // 이미 위치가 계산되어 있다면 그대로 반환
  if (fruitPositions.value[key]) {
    return fruitPositions.value[key]
  }
  
  // 새로운 위치 계산 및 저장
  const position = {
    '--fruit-left': `${20 + Math.random() * 65}%`,
    '--fruit-top': `${10 + Math.random() * 75}%`,
    '--animation-delay': `${Math.random() * 0.5}s`,
    '--animation-duration': `${0.5 + Math.random() * 1}s`,
    '--fly-distance': `${30 + Math.random() * 50}px`,
    '--fly-direction': `${-150 + Math.random() * 300}deg`
  }
  
  fruitPositions.value[key] = position
  return position
}

// 이모지 파싱
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

// 과일 개수 계산
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

// 총 과일 개수 가져오기
const getTotalFruits = (user) => {
  const { fruits } = parseFruitEmojis(user.items)
  return fruits.length
}

// 과일별 색상 정의
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
  fetchRoomTitle() // 방 제목 가져오기
  fetchUserData()
  
  // 세로 스크롤을 가로 스크롤로 변환하는 이벤트 리스너 추가
  window.addEventListener('wheel', handleVerticalScroll, { passive: false })
  
  // 터치 이벤트에 대한 처리 추가
  if (typeof window !== 'undefined') {
    document.addEventListener('touchstart', handleTouchStart, { passive: true })
    document.addEventListener('touchmove', handleTouchMove, { passive: false })
    document.addEventListener('touchend', handleTouchEnd, { passive: true })
  }
  
  // 언마운트 시 이벤트 리스너 제거를 위해 함수 저장
  onBeforeUnmount(() => {
    window.removeEventListener('wheel', handleVerticalScroll)
    if (typeof window !== 'undefined') {
      document.removeEventListener('touchstart', handleTouchStart)
      document.removeEventListener('touchmove', handleTouchMove)
      document.removeEventListener('touchend', handleTouchEnd)
    }
  })
})
</script>

<style scoped>
.fridge-view {
  max-width: 100%;
  width: 100%;
  margin: 0;
  padding: 0;
  background-color: #f5f7fa;
  min-height: 100vh;
  position: relative;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.room-header {
  text-align: center;
  margin: 0;
  background-color: rgba(255, 255, 255, 0.8);
  padding: 0.8rem;
  border-radius: 0;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
  max-width: 100%;
  overflow: hidden;
  z-index: 10;
}

h1 {
  font-size: 1.3rem;
  color: #2c3e50;
  margin: 0;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

@media (min-width: 640px) {
  h1 {
    font-size: 1.5rem;
  }
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
  margin: 2rem;
}

.loader {
  width: 40px;
  height: 40px;
  border: 4px solid #3498db;
  border-radius: 50%;
  border-top-color: transparent;
  animation: spin 1s linear infinite;
  margin-bottom: 1rem;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.error {
  color: #e74c3c;
}

.fridges-container {
  display: flex;
  flex-direction: row;
  overflow-x: auto;
  justify-content: flex-start;
  align-items: center;
  gap: 30px;
  padding: 20px;
  -webkit-overflow-scrolling: touch;
  scrollbar-width: thin;
  scrollbar-color: #3498db transparent;
  width: 100%;
  max-width: 100vw;
  margin: 0;
  flex: 1;
  min-height: 400px;
  scroll-behavior: smooth;
  overscroll-behavior-y: none;
  touch-action: pan-x;
}

/* 모바일에서 스크롤바 숨기기 */
@media (max-width: 768px) {
  .fridges-container {
    scrollbar-width: none;
    -ms-overflow-style: none;
    overflow-y: hidden;
  }
  
  .fridges-container::-webkit-scrollbar {
    display: none;
  }
  
  .fridge-view {
    overflow: hidden;
    touch-action: none;
  }
}

.fridge-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 15px;
  flex: 0 0 260px;
  min-width: 260px;
  transition: transform 0.3s;
  position: relative;
}

.user-info {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  margin-bottom: 15px;
  position: relative;
  background-color: rgba(255, 255, 255, 0.9);
  border-radius: 8px;
  padding: 0.6rem 1rem;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  z-index: 5;
}

.user-name {
  font-weight: 700;
  font-size: 1rem;
  color: #2c3e50;
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
  background-color: #3498db;
  color: white;
  font-size: 0.8rem;
  font-weight: bold;
  border-radius: 50%;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

/* 냉장고 스타일 */
.fridge {
  position: relative;
  width: 100%;
  height: 350px;
  background-color: #f0f0f0;
  border-radius: 10px;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.1);
  overflow: hidden;
  transition: all 0.5s ease;
  perspective: 1000px;
  cursor: pointer;
}

.fridge-door {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(135deg, #ecf0f1 0%, #bdc3c7 100%);
  border-radius: 10px;
  border: 2px solid #ddd;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
  transform-origin: left;
  transition: transform 0.5s ease;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 20px;
  z-index: 2;
}

.fridge-handle {
  position: absolute;
  top: 50%;
  right: 20px;
  transform: translateY(-50%);
  width: 10px;
  height: 80px;
  background-color: #7f8c8d;
  border-radius: 5px;
  box-shadow: 2px 0 5px rgba(0, 0, 0, 0.1);
}

.fridge-info {
  font-size: 1rem;
  color: #34495e;
  text-align: center;
  font-weight: 600;
  background-color: rgba(255, 255, 255, 0.8);
  border-radius: 8px;
  padding: 0.5rem 1rem;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
  margin-top: 20px;
}

.fridge-open .fridge-door {
  transform: rotateY(-100deg);
}

.fridge-interior {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(135deg, #f3f7fa 0%, #e0e5e9 100%);
  border-radius: 8px;
  padding: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
}

/* 냉장고 조명 효과 */
.fridge-light {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: radial-gradient(ellipse at center, rgba(255, 255, 255, 0.8) 0%, rgba(255, 255, 255, 0.1) 100%);
  opacity: 0;
  transition: opacity 0.5s ease;
  pointer-events: none;
  z-index: 1;
}

.fridge-open .fridge-light {
  opacity: 1;
  animation: fridge-light-flicker 0.5s ease-in-out;
}

@keyframes fridge-light-flicker {
  0% { opacity: 0; }
  20% { opacity: 0.4; }
  40% { opacity: 0.2; }
  60% { opacity: 0.6; }
  80% { opacity: 0.4; }
  100% { opacity: 1; }
}

/* 냉장고 선반 효과 */
.fridge-interior::before,
.fridge-interior::after {
  content: '';
  position: absolute;
  left: 0;
  width: 100%;
  height: 2px;
  background-color: rgba(0, 0, 0, 0.1);
}

.fridge-interior::before {
  top: 33%;
}

.fridge-interior::after {
  top: 66%;
}

/* 과일 스타일 */
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
  opacity: 0;
  transform: scale(0.8);
}

.fruit-active {
  opacity: 1;
  transform: scale(1);
  transition: opacity 0.4s ease-in-out, transform 0.4s ease-in-out;
  transition-delay: calc(0.05s * var(--fruit-index, 0) + 0.2s);
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

.fruit-animate {
  animation: fruit-hover 0.8s ease-in-out infinite alternate;
}

@keyframes fruit-hover {
  0% {
    transform: translateY(0);
  }
  100% {
    transform: translateY(-8px);
  }
}

/* 1-3위 냉장고 스타일 차별화 */
.fridge-container:nth-child(1) .fridge {
  background: linear-gradient(135deg, #f9fafb 0%, #f1f3f5 100%);
  border: 2px solid #ffd700;
  box-shadow: 0 5px 20px rgba(255, 215, 0, 0.3);
}

.fridge-container:nth-child(2) .fridge {
  background: linear-gradient(135deg, #f7f8fa 0%, #edf0f3 100%);
  border: 2px solid #c0c0c0;
  box-shadow: 0 5px 20px rgba(192, 192, 192, 0.3);
}

.fridge-container:nth-child(3) .fridge {
  background: linear-gradient(135deg, #f5f6f8 0%, #ebeef1 100%);
  border: 2px solid #cd7f32;
  box-shadow: 0 5px 20px rgba(205, 127, 50, 0.3);
}
</style>