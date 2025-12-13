<template>
  <div class="performance-demo">
    <div class="demo-header">
      <h1>성능 최적화 데모</h1>
      <div class="controls">
        <button @click="toggleOptimization" :class="{ active: optimizationEnabled }">
          최적화 {{ optimizationEnabled ? '켜짐' : '꺼짐' }}
        </button>
        <button @click="loadLargeDataset">대용량 데이터 로드</button>
        <button @click="clearData">데이터 지우기</button>
        <button @click="resetMetrics">메트릭 초기화</button>
      </div>
    </div>
    
    <div class="demo-content">
      <div class="diagram-container">
        <DxDiagram
          ref="diagramRef"
          @request-edit-operation="onRequestEditOperation"
          @content-ready="onContentReady"
          :custom-shape-render="customShapeRender"
        >
          <DxToolbox :visible="true" />
          <DxPropertiesPanel :visible="false" />
        </DxDiagram>
      </div>
      
      <div class="metrics-panel">
        <div class="metrics-header">
          <h3>성능 메트릭</h3>
          <div class="optimization-status">
            <span :class="['status-indicator', optimizationEnabled ? 'enabled' : 'disabled']"></span>
            {{ optimizationEnabled ? '최적화 활성화' : '최적화 비활성화' }}
          </div>
        </div>
        
        <div class="metrics-content">
          <div class="metric-group">
            <h4>작업 통계</h4>
            <div class="metric-grid">
              <div class="metric-item">
                <span class="metric-label">총 작업:</span>
                <span class="metric-value">{{ metrics.totalOperations }}</span>
              </div>
              <div class="metric-item">
                <span class="metric-label">허용:</span>
                <span class="metric-value success">{{ metrics.allowedOperations }}</span>
              </div>
              <div class="metric-item">
                <span class="metric-label">거부:</span>
                <span class="metric-value error">{{ metrics.deniedOperations }}</span>
              </div>
              <div class="metric-item">
                <span class="metric-label">평균 시간:</span>
                <span class="metric-value">{{ metrics.averageValidationTime.toFixed(2) }}ms</span>
              </div>
            </div>
          </div>
          
          <div class="metric-group">
            <h4>캐시 성능</h4>
            <div class="metric-grid">
              <div class="metric-item">
                <span class="metric-label">캐시 적중:</span>
                <span class="metric-value success">{{ metrics.cacheHits }}</span>
              </div>
              <div class="metric-item">
                <span class="metric-label">캐시 미스:</span>
                <span class="metric-value warning">{{ metrics.cacheMisses }}</span>
              </div>
              <div class="metric-item">
                <span class="metric-label">캐시 비율:</span>
                <span class="metric-value">{{ getCacheHitRate() }}%</span>
              </div>
            </div>
          </div>
          
          <div class="metric-group">
            <h4>데이터 크기</h4>
            <div class="metric-grid">
              <div class="metric-item">
                <span class="metric-label">도형 수:</span>
                <span class="metric-value">{{ shapeCount }}</span>
              </div>
              <div class="metric-item">
                <span class="metric-label">연결선 수:</span>
                <span class="metric-value">{{ connectionCount }}</span>
              </div>
              <div class="metric-item">
                <span class="metric-label">총 항목:</span>
                <span class="metric-value">{{ totalItems }}</span>
              </div>
            </div>
          </div>
          
          <div class="operation-breakdown">
            <h4>작업 유형별 분석</h4>
            <div class="operation-list">
              <div 
                v-for="[operation, count] in operationTypes" 
                :key="operation"
                class="operation-item"
              >
                <span class="operation-name">{{ operation }}:</span>
                <span class="operation-count">{{ count }}</span>
                <div class="operation-bar">
                  <div 
                    class="operation-bar-fill" 
                    :style="{ width: getOperationPercentage(count) + '%' }"
                  ></div>
                </div>
              </div>
            </div>
          </div>
        </div>
        
        <div class="recent-operations">
          <h4>최근 작업 ({{ recentOperations.length }})</h4>
          <div class="operation-log">
            <div 
              v-for="(op, index) in recentOperations" 
              :key="index"
              :class="['operation-log-item', op.allowed ? 'success' : 'error']"
            >
              <span class="op-time">{{ op.time }}</span>
              <span class="op-name">{{ op.operation }}</span>
              <span class="op-duration">{{ op.duration }}ms</span>
              <span class="op-status">{{ op.allowed ? '✓' : '✗' }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, nextTick } from 'vue'
import { DxDiagram, DxToolbox, DxPropertiesPanel } from 'devextreme-vue/diagram'
import type { 
  RequestEditOperationEvent, 
  ContentReadyEvent
} from 'devextreme/ui/diagram'

// 상태 관리
const diagramRef = ref<InstanceType<typeof DxDiagram>>()
const optimizationEnabled = ref<boolean>(true)
const shapeCount = ref<number>(0)
const connectionCount = ref<number>(0)

// 성능 메트릭
const metrics = ref({
  totalOperations: 0,
  allowedOperations: 0,
  deniedOperations: 0,
  averageValidationTime: 0,
  cacheHits: 0,
  cacheMisses: 0
})

// 캐시 시스템
const validationCache = new Map<string, { result: boolean; timestamp: number }>()
const CACHE_DURATION = 3000 // 3초

// 작업 유형별 통계
const operationTypes = ref<Map<string, number>>(new Map())

// 최근 작업 로그
const recentOperations = ref<Array<{
  operation: string
  allowed: boolean
  duration: number
  time: string
}>>([])

// 계산된 속성
const totalItems = computed(() => shapeCount.value + connectionCount.value)

// 캐시 적중률 계산
const getCacheHitRate = () => {
  const total = metrics.value.cacheHits + metrics.value.cacheMisses
  return total > 0 ? Math.round((metrics.value.cacheHits / total) * 100) : 0
}

// 작업 비율 계산
const getOperationPercentage = (count: number) => {
  const total = metrics.value.totalOperations
  return total > 0 ? Math.round((count / total) * 100) : 0
}

// 커스텀 도형 렌더링
const customShapeRender = (e: any) => {
  // 기본 렌더링 사용
}

// 고성능 캐시 키 생성
const generateCacheKey = (operation: string, args: any): string => {
  try {
    const shapeId = args.shape?.id || args.connection?.id || 'unknown'
    const operationData = {
      operation,
      shapeId,
      shapeType: args.shape?.type,
      position: args.position,
      text: args.text
    }
    return JSON.stringify(operationData)
  } catch {
    return `${operation}_${Date.now()}`
  }
}

// 캐시에서 결과 조회
const getCachedResult = (key: string): { allowed: boolean; reason?: string } | null => {
  const cached = validationCache.get(key)
  if (!cached) return null
  
  const now = Date.now()
  if (now - cached.timestamp > CACHE_DURATION) {
    validationCache.delete(key)
    return null
  }
  
  return { allowed: cached.result }
}

// 캐시에 결과 저장
const cacheResult = (key: string, result: boolean) => {
  validationCache.set(key, {
    result,
    timestamp: Date.now()
  })
  
  // 캐시 크기 제한 (100개)
  if (validationCache.size > 100) {
    const firstKey = validationCache.keys().next().value
    validationCache.delete(firstKey)
  }
}

// 고성능 유효성 검사
const performFastValidation = (e: RequestEditOperationEvent): { allowed: boolean; reason?: string } => {
  const operation = e.operation as string
  const args = e.args as any
  
  // 빠른 캐시 조회
  if (optimizationEnabled.value) {
    const cacheKey = generateCacheKey(operation, args)
    const cached = getCachedResult(cacheKey)
    if (cached) {
      metrics.value.cacheHits++
      return cached
    }
    metrics.value.cacheMisses++
  }
  
  // 가벼운 유효성 검사만 수행
  let allowed = true
  let reason = ''
  
  switch (operation) {
    case 'addShape':
      // 메모리 제한 검사
      if (totalItems.value > 1000) {
        allowed = false
        reason = '최대 항목 수(1000)를 초과했습니다.'
      }
      break
      
    case 'deleteShape':
      // 보호된 도형 빠른 검사
      if (args.shape?.dataItem?.protected) {
        allowed = false
        reason = '보호된 도형입니다.'
      }
      break
      
    case 'moveShape':
      // 빠른 경계 검사
      const pos = args.position
      if (pos.x < 0 || pos.y < 0 || pos.x > 5000 || pos.y > 5000) {
        allowed = false
        reason = '이동 범위를 벗어났습니다.'
      }
      break
      
    case 'changeConnection':
      // 빠른 자기 연결 검사
      if (args.connection?.from?.id === args.connection?.to?.id) {
        allowed = false
        reason = '자기 자신으로의 연결은 불가능합니다.'
      }
      break
  }
  
  // 결과 캐싱
  if (optimizationEnabled.value) {
    const cacheKey = generateCacheKey(operation, args)
    cacheResult(cacheKey, allowed)
  }
  
  return { allowed, reason }
}

// 편집 작업 제어 (고성능)
const onRequestEditOperation = (e: RequestEditOperationEvent) => {
  const startTime = performance.now()
  
  try {
    // 빠른 유효성 검사 수행
    const result = performFastValidation(e)
    
    // 결과 적용
    e.allowed = result.allowed
    ;(e as any).reason = result.reason
    
    // 메트릭 업데이트
    updateMetrics(e.operation, result.allowed, startTime)
    
    // 최근 작업 로그 추가
    addRecentOperation(e.operation, result.allowed, startTime)
    
  } catch (error) {
    console.error('편집 작업 오류:', error)
    e.allowed = false
    ;(e as any).reason = '처리 중 오류 발생'
  }
}

// 메트릭 업데이트
const updateMetrics = (operation: string, allowed: boolean, startTime: number) => {
  const duration = performance.now() - startTime
  
  metrics.value.totalOperations++
  if (allowed) {
    metrics.value.allowedOperations++
  } else {
    metrics.value.deniedOperations++
  }
  
  // 평균 시간 업데이트 (이동 평균)
  const alpha = 0.1 // 학습률
  metrics.value.averageValidationTime = 
    metrics.value.averageValidationTime * (1 - alpha) + duration * alpha
  
  // 작업 유형별 카운트
  const currentCount = operationTypes.value.get(operation) || 0
  operationTypes.value.set(operation, currentCount + 1)
}

// 최근 작업 로그 추가
const addRecentOperation = (operation: string, allowed: boolean, startTime: number) => {
  const duration = Math.round(performance.now() - startTime)
  const time = new Date().toLocaleTimeString('ko-KR')
  
  recentOperations.value.unshift({
    operation,
    allowed,
    duration,
    time
  })
  
  // 최근 20개만 유지
  if (recentOperations.value.length > 20) {
    recentOperations.value = recentOperations.value.slice(0, 20)
  }
}

// 대용량 데이터 로드
const loadLargeDataset = () => {
  const diagram = diagramRef.value?.instance
  if (!diagram) return
  
  const nodeCount = 200
  const edgeCount = 300
  const nodes = []
  const edges = []
  
  // 노드 생성
  for (let i = 1; i <= nodeCount; i++) {
    nodes.push({
      id: i,
      text: `노드 ${i}`,
      type: ['rectangle', 'ellipse', 'diamond'][Math.floor(Math.random() * 3)],
      x: Math.random() * 2000,
      y: Math.random() * 1500
    })
  }
  
  // 간선 생성
  for (let i = 1; i <= edgeCount; i++) {
    const from = Math.floor(Math.random() * nodeCount) + 1
    let to = Math.floor(Math.random() * nodeCount) + 1
    
    // 자기 자신으로의 연결 방지
    while (to === from) {
      to = Math.floor(Math.random() * nodeCount) + 1
    }
    
    edges.push({
      id: i,
      from,
      to,
      text: ''
    })
  }
  
  const largeDataset = {
    nodeKeyProperty: 'id',
    edgeKeyProperty: 'id',
    nodes,
    edges
  }
  
  diagram.import(JSON.stringify(largeDataset))
  
  // 카운트 업데이트
  const items = diagram.getItems()
  shapeCount.value = items.filter(item => item.itemType === 'shape').length
  connectionCount.value = items.filter(item => item.itemType === 'connector').length
  
  addRecentOperation('loadLargeDataset', true, performance.now())
}

// 데이터 지우기
const clearData = () => {
  const diagram = diagramRef.value?.instance
  if (!diagram) return
  
  diagram.import(JSON.stringify({
    nodeKeyProperty: 'id',
    edgeKeyProperty: 'id',
    nodes: [],
    edges: []
  }))
  
  shapeCount.value = 0
  connectionCount.value = 0
  
  // 캐시 초기화
  validationCache.clear()
  
  addRecentOperation('clearData', true, performance.now())
}

// 메트릭 초기화
const resetMetrics = () => {
  metrics.value = {
    totalOperations: 0,
    allowedOperations: 0,
    deniedOperations: 0,
    averageValidationTime: 0,
    cacheHits: 0,
    cacheMisses: 0
  }
  
  operationTypes.value.clear()
  recentOperations.value = []
  validationCache.clear()
}

// 최적화 토글
const toggleOptimization = () => {
  optimizationEnabled.value = !optimizationEnabled.value
  
  if (!optimizationEnabled.value) {
    // 최적화 비활성화 시 캐시 초기화
    validationCache.clear()
    metrics.value.cacheHits = 0
    metrics.value.cacheMisses = 0
  }
  
  addRecentOperation(
    `optimization_${optimizationEnabled.value ? 'enabled' : 'disabled'}`,
    true,
    performance.now()
  )
}

// 컨텐츠 준비 완료
const onContentReady = (e: ContentReadyEvent) => {
  const diagram = e.component
  const items = diagram.getItems()
  shapeCount.value = items.filter(item => item.itemType === 'shape').length
  connectionCount.value = items.filter(item => item.itemType === 'connector').length
}
</script>

<style scoped>
.performance-demo {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background-color: #f5f5f5;
}

.demo-header {
  background: white;
  padding: 16px 24px;
  border-bottom: 1px solid #e0e0e0;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.demo-header h1 {
  margin: 0;
  font-size: 20px;
  font-weight: 600;
  color: #333;
}

.controls {
  display: flex;
  gap: 12px;
  align-items: center;
}

.controls button {
  padding: 8px 16px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: white;
  cursor: pointer;
  transition: all 0.2s;
  font-size: 14px;
}

.controls button:hover {
  background: #f8f9fa;
  border-color: #999;
}

.controls button.active {
  background: #3b82f6;
  color: white;
  border-color: #3b82f6;
}

.demo-content {
  flex: 1;
  display: flex;
  overflow: hidden;
}

.diagram-container {
  flex: 1;
  background: white;
  border-right: 1px solid #e0e0e0;
}

.metrics-panel {
  width: 500px;
  background: white;
  display: flex;
  flex-direction: column;
  border-left: 1px solid #e0e0e0;
}

.metrics-header {
  padding: 16px;
  border-bottom: 1px solid #e0e0e0;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.metrics-header h3 {
  margin: 0;
  font-size: 16px;
  font-weight: 600;
  color: #333;
}

.optimization-status {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 14px;
}

.status-indicator {
  width: 8px;
  height: 8px;
  border-radius: 50%;
}

.status-indicator.enabled {
  background-color: #10b981;
}

.status-indicator.disabled {
  background-color: #ef4444;
}

.metrics-content {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
}

.metric-group {
  margin-bottom: 24px;
}

.metric-group h4 {
  margin: 0 0 12px 0;
  font-size: 14px;
  font-weight: 600;
  color: #333;
}

.metric-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
}

.metric-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 12px;
  background: #f8f9fa;
  border-radius: 4px;
  border: 1px solid #e9ecef;
}

.metric-label {
  font-size: 12px;
  color: #666;
}

.metric-value {
  font-size: 14px;
  font-weight: 600;
  color: #333;
}

.metric-value.success {
  color: #10b981;
}

.metric-value.error {
  color: #ef4444;
}

.metric-value.warning {
  color: #f59e0b;
}

.operation-breakdown {
  margin-bottom: 24px;
}

.operation-breakdown h4 {
  margin: 0 0 12px 0;
  font-size: 14px;
  font-weight: 600;
  color: #333;
}

.operation-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.operation-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 8px 12px;
  background: #f8f9fa;
  border-radius: 4px;
  border: 1px solid #e9ecef;
}

.operation-name {
  font-size: 12px;
  color: #666;
  min-width: 120px;
}

.operation-count {
  font-size: 14px;
  font-weight: 600;
  color: #333;
  min-width: 40px;
}

.operation-bar {
  flex: 1;
  height: 4px;
  background: #e9ecef;
  border-radius: 2px;
  overflow: hidden;
}

.operation-bar-fill {
  height: 100%;
  background: #3b82f6;
  border-radius: 2px;
  transition: width 0.3s ease;
}

.recent-operations {
  border-top: 1px solid #e0e0e0;
  padding-top: 16px;
}

.recent-operations h4 {
  margin: 0 0 12px 0;
  font-size: 14px;
  font-weight: 600;
  color: #333;
}

.operation-log {
  max-height: 200px;
  overflow-y: auto;
}

.operation-log-item {
  display: grid;
  grid-template-columns: 60px 120px 50px 30px;
  gap: 8px;
  align-items: center;
  padding: 6px 8px;
  margin-bottom: 2px;
  border-radius: 4px;
  font-size: 12px;
}

.operation-log-item.success {
  background-color: #f0f9ff;
}

.operation-log-item.error {
  background-color: #fef2f2;
}

.op-time {
  color: #666;
}

.op-name {
  font-weight: 500;
  color: #333;
}

.op-duration {
  text-align: right;
  color: #666;
}

.op-status {
  text-align: center;
  font-weight: bold;
}

/* 스크롤바 스타일 */
.metrics-content::-webkit-scrollbar,
.operation-log::-webkit-scrollbar {
  width: 6px;
}

.metrics-content::-webkit-scrollbar-track,
.operation-log::-webkit-scrollbar-track {
  background: #f1f1f1;
}

.metrics-content::-webkit-scrollbar-thumb,
.operation-log::-webkit-scrollbar-thumb {
  background: #c1c1c1;
  border-radius: 3px;
}

.metrics-content::-webkit-scrollbar-thumb:hover,
.operation-log::-webkit-scrollbar-thumb:hover {
  background: #a8a8a8;
}
</style>
