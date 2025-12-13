<template>
  <div id="app">
    <div class="app-header">
      <h1>DevExpress Diagram onRequestEditOperation 학습 센터</h1>
      <nav class="main-nav">
        <button 
          v-for="tab in tabs" 
          :key="tab.id"
          @click="currentTab = tab.id"
          :class="['nav-btn', { active: currentTab === tab.id }]"
        >
          {{ tab.name }}
        </button>
      </nav>
    </div>
    
    <div class="app-content">
      <DiagramDemo v-if="currentTab === 'demo'" />
      <PerformanceDemo v-else-if="currentTab === 'performance'" />
      <InteractiveExamples v-else-if="currentTab === 'examples'" />
      <GuideView v-else-if="currentTab === 'guide'" />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import DiagramDemo from './components/DiagramDemo.vue'
import PerformanceDemo from './components/PerformanceDemo.vue'
import InteractiveExamples from './components/InteractiveExamples.vue'
import GuideView from './components/GuideView.vue'

const tabs = ref([
  { id: 'demo', name: '데모' },
  { id: 'examples', name: '인터랙티브 예제' },
  { id: 'performance', name: '성능 최적화' },
  { id: 'guide', name: '학습 가이드' }
])

const currentTab = ref('demo')
</script>

<style>
@import 'devextreme/dist/css/dx.material.blue.light.css';

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

#app {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  height: 100vh;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.app-header {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 20px 24px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
}

.app-header h1 {
  margin: 0;
  font-size: 24px;
  font-weight: 600;
}

.main-nav {
  display: flex;
  gap: 8px;
}

.nav-btn {
  padding: 10px 20px;
  border: none;
  border-radius: 6px;
  background: rgba(255,255,255,0.2);
  color: white;
  cursor: pointer;
  transition: all 0.3s ease;
  font-size: 14px;
  font-weight: 500;
}

.nav-btn:hover {
  background: rgba(255,255,255,0.3);
  transform: translateY(-1px);
}

.nav-btn.active {
  background: white;
  color: #667eea;
  box-shadow: 0 2px 8px rgba(0,0,0,0.15);
}

.app-content {
  flex: 1;
  overflow: hidden;
}
</style>
