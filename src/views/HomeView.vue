<script setup>
import { ref } from 'vue'
import { useRouter } from 'vue-router'

const roomId = ref('')
const router = useRouter()

const goToRoom = () => {
  if (roomId.value.trim()) {
    router.push({ name: 'room', params: { roomId: roomId.value } })
  }
}
</script>

<template>
  <div class="home">
    <h1>과일 퀴즈 점수 보기</h1>
    <div class="input-container">
      <input 
        v-model="roomId" 
        placeholder="Room ID를 입력하세요" 
        @keyup.enter="goToRoom"
      />
      <button @click="goToRoom">입장</button>
    </div>
  </div>
</template>

<style scoped>
.home {
  max-width: 800px;
  margin: 0 auto;
  padding: 2rem;
  text-align: center;
}

h1 {
  font-size: 2rem;
  color: #2c3e50;
  margin-bottom: 2rem;
}

.input-container {
  display: flex;
  justify-content: center;
  gap: 0.5rem;
}

input {
  padding: 0.5rem 1rem;
  font-size: 1rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  width: 300px;
}

button {
  padding: 0.5rem 1.5rem;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 1rem;
  transition: background-color 0.3s;
}

button:hover {
  background-color: #45a049;
}
</style>
