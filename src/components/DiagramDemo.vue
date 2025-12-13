<template>
  <div class="diagram-demo">
    <div class="demo-header">
      <h1>DevExpress Diagram onRequestEditOperation 데모</h1>
      <div class="controls">
        <select v-model="currentRole" @change="onRoleChange">
          <option value="admin">관리자</option>
          <option value="editor">편집자</option>
          <option value="viewer">뷰어</option>
        </select>
        <button @click="clearLogs">로그 지우기</button>
        <button @click="loadSampleData">샘플 데이터 로드</button>
      </div>
    </div>
    
    <div class="demo-content">
      <div class="diagram-container">
        <DxDiagram
          ref="diagramRef"
          :custom-shape-render="customShapeRender"
          @request-edit-operation="onRequestEditOperation"
          @content-ready="onContentReady"
          @item-click="onItemClick"
          @item-dbl-click="onItemDoubleClick"
          @selection-changed="onSelectionChanged"
          :page-orientation="pageOrientation"
          :page-size="pageSize"
        >
          <DxContextMenu :enabled="true" :commands="contextMenuCommands" />
          <DxToolbox :visibility="'visible'" :groups="toolboxGroups" :show-search="true" :shape-icons-per-row="4" />
          <DxContextToolbox :enabled="true" :category="'flowchart'" :shape-icons-per-row="3" :width="220" :shapes="['terminator','process','decision','document']" />
          <DxPropertiesPanel :visibility="'visible'" />
          <DxEditing 
            :allow-add-shape="true"
            :allow-delete-shape="true"
            :allow-move-shape="true"
            :allow-change-connection="true"
            :allow-change-shape-text="true"
            :allow-resize-shape="true"
          />
        </DxDiagram>
      </div>
      
      <div class="log-panel">
        <div class="log-header">
          <h3>이벤트 로그</h3>
          <div class="log-stats">
            <span>총: {{ eventLogs.length }}</span>
            <span>허용: {{ allowedCount }}</span>
            <span>거부: {{ deniedCount }}</span>
          </div>
        </div>
        <div class="log-content" ref="logContent">
          <div 
            v-for="(log, index) in eventLogs" 
            :key="index"
            :class="['log-item', log.allowed ? 'allowed' : 'denied']"
          >
            <div class="log-time">{{ log.timestamp }}</div>
            <div class="log-operation">{{ log.operation }}</div>
            <div class="log-status">
              <span :class="['status-badge', log.allowed ? 'success' : 'error']">
                {{ log.allowed ? '허용' : '거부' }}
              </span>
            </div>
            <div class="log-reason" v-if="log.reason">{{ log.reason }}</div>
          </div>
        </div>
      </div>
    </div>
    
    <div class="status-bar">
      <div class="status-info">
        <span>역할: {{ currentRoleDisplay }}</span>
        <span>도형 수: {{ shapeCount }}</span>
        <span>연결선 수: {{ connectionCount }}</span>
        <span>선택된 항목: {{ selectedItemsCount }}</span>
      </div>
      <div class="last-operation">
        마지막 작업: {{ lastOperation || '없음' }}
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, nextTick } from 'vue'
import { DxDiagram, DxContextMenu, DxToolbox, DxPropertiesPanel, DxContextToolbox, DxEditing } from 'devextreme-vue/diagram'
import type { 
  RequestEditOperationEvent, 
  DiagramCustomShapeRenderEvent,
  DiagramContentReadyEvent,
  DiagramItemClickEvent,
  DiagramItemDoubleClickEvent,
  DiagramSelectionChangedEvent
} from 'devextreme/ui/diagram'

// 상태 관리
const diagramRef = ref<InstanceType<typeof DxDiagram>>()
const currentRole = ref<string>('editor')
const eventLogs = ref<Array<{
  timestamp: string
  operation: string
  allowed: boolean
  reason?: string
  details?: any
}>>([])
const lastOperation = ref<string>('')
const shapeCount = ref<number>(0)
const connectionCount = ref<number>(0)
const selectedItemsCount = ref<number>(0)

// 계산된 속성
const currentRoleDisplay = computed(() => {
  const roleMap = {
    admin: '관리자',
    editor: '편집자',
    viewer: '뷰어'
  }
  return roleMap[currentRole.value as keyof typeof roleMap]
})

const allowedCount = computed(() => eventLogs.value.filter(log => log.allowed).length)
const deniedCount = computed(() => eventLogs.value.filter(log => !log.allowed).length)

// 권한 설정
const permissions = computed(() => ({
  canAdd: ['admin', 'editor'].includes(currentRole.value),
  canDelete: currentRole.value === 'admin',
  canEdit: ['admin', 'editor'].includes(currentRole.value),
  canMove: ['admin', 'editor'].includes(currentRole.value),
  canConnect: ['admin', 'editor'].includes(currentRole.value)
}))

// 다이어그램 설정
const pageOrientation = ref<'portrait' | 'landscape'>('landscape')
const pageSize = ref({ width: 1169, height: 827 })

// 도구 상자 그룹
const toolboxGroups = ref([
  {
    category: 'general',
    title: '일반',
    shapes: ['rectangle', 'ellipse', 'triangle', 'diamond']
  },
  {
    category: 'flowchart',
    title: '플로우차트',
    shapes: ['terminator', 'process', 'decision', 'document']
  }
])

// 커스텀 도형 렌더링
const customShapeRender = (e: DiagramCustomShapeRenderEvent) => {
  if (e.shape.type === 'decision') {
    e.contentTemplate = (container: HTMLElement, shape: any) => {
      container.innerHTML = `
        <div class="decision-shape">
          <div class="decision-text">${shape.text || '결정'}</div>
        </div>
      `
    }
  }
}

// 편집 작업 제어
const onRequestEditOperation = (e: RequestEditOperationEvent) => {
  const startTime = performance.now()
  const operation = e.operation
  let allowed = true
  let reason = ''

  try {
    // 기본 권한 검사
    if (!checkBasicPermissions(e)) {
      allowed = false
      reason = e.reason || '권한이 부족합니다.'
    } else {
      // 비즈니스 규칙 적용
      const businessResult = applyBusinessRules(e)
      allowed = businessResult.allowed
      reason = businessResult.reason
    }

    // 이벤트 로그 추가
    addEventLog({
      operation,
      allowed,
      reason,
      details: { ...e.args }
    })

    // 결과 적용
    e.allowed = allowed
    e.reason = reason
    lastOperation.value = `${operation} (${allowed ? '허용' : '거부'})`

  } catch (error) {
    console.error('편집 작업 오류:', error)
    e.allowed = false
    e.reason = '작업 처리 중 오류가 발생했습니다.'
    addEventLog({
      operation,
      allowed: false,
      reason: e.reason,
      details: { error }
    })
  } finally {
    const endTime = performance.now()
    console.log(`편집 작업 검증 시간: ${endTime - startTime}ms`)
  }
}

// 기본 권한 검사
const checkBasicPermissions = (e: RequestEditOperationEvent): boolean => {
  const perm = permissions.value
  
  switch (e.operation) {
    case 'addShape':
      if (!perm.canAdd) {
        e.allowed = false
        e.reason = `${currentRoleDisplay.value}는 도형을 추가할 수 없습니다.`
        return false
      }
      break
      
    case 'deleteShape':
    case 'deleteConnection':
      if (!perm.canDelete) {
        e.allowed = false
        e.reason = `${currentRoleDisplay.value}는 삭제할 수 없습니다.`
        return false
      }
      break
      
    case 'changeShapeText':
    case 'changeConnection':
    case 'resizeShape':
      if (!perm.canEdit) {
        e.allowed = false
        e.reason = `${currentRoleDisplay.value}는 편집할 수 없습니다.`
        return false
      }
      break
      
    case 'moveShape':
      if (!perm.canMove) {
        e.allowed = false
        e.reason = `${currentRoleDisplay.value}는 이동할 수 없습니다.`
        return false
      }
      break
  }
  
  return true
}

// 비즈니스 규칙 적용
const applyBusinessRules = (e: RequestEditOperationEvent): { allowed: boolean; reason: string } => {
  const diagram = diagramRef.value?.instance
  if (!diagram) return { allowed: true, reason: '' }

  switch (e.operation) {
    case 'addShape': {
      // 최대 도형 수 제한
      const maxShapes = 20
      const currentCount = diagram.getItems().filter(item => item.itemType === 'shape').length
      
      if (currentCount >= maxShapes) {
        return {
          allowed: false,
          reason: `최대 ${maxShapes}개의 도형까지만 추가 가능합니다. (현재: ${currentCount})`
        }
      }
      
      // 특정 도형 타입 제한
      const shapeType = e.args.shape.type
      const restrictedTypes = ['triangle']
      if (restrictedTypes.includes(shapeType) && currentRole.value !== 'admin') {
        return {
          allowed: false,
          reason: `${shapeType} 타입 도형은 관리자만 추가할 수 있습니다.`
        }
      }
      break
    }
    
    case 'deleteShape': {
      // 보호된 도형 확인
      const shape = e.args.shape
      if (shape.dataItem?.protected) {
        return {
          allowed: false,
          reason: '보호된 도형은 삭제할 수 없습니다.'
        }
      }
      
      // 루트 도형 보호
      if (shape.dataItem?.isRoot) {
        return {
          allowed: false,
          reason: '루트 도형은 삭제할 수 없습니다.'
        }
      }
      break
    }
    
    case 'addConnection': {
      // 자기 자신으로의 연결 금지
      const connection = e.args.connection
      if (connection.from.id === connection.to.id) {
        return {
          allowed: false,
          reason: '자기 자신으로의 연결은 허용되지 않습니다.'
        }
      }
      
      // 최대 연결선 수 제한
      const maxConnections = 30
      const currentConnections = diagram.getItems().filter(item => item.itemType === 'connection').length
      
      if (currentConnections >= maxConnections) {
        return {
          allowed: false,
          reason: `최대 ${maxConnections}개의 연결선만 가능합니다. (현재: ${currentConnections})`
        }
      }
      break
    }
    
    case 'moveShape': {
      // 이동 범위 제한
      const position = e.args.position
      const bounds = { minX: 0, maxX: 2000, minY: 0, maxY: 1500 }
      
      if (position.x < bounds.minX || position.x > bounds.maxX ||
          position.y < bounds.minY || position.y > bounds.maxY) {
        return {
          allowed: false,
          reason: `도형은 (${bounds.minX},${bounds.minY}) ~ (${bounds.maxX},${bounds.maxY}) 범위 내에서만 이동 가능합니다.`
        }
      }
      break
    }
  }
  
  return { allowed: true, reason: '' }
}

// 이벤트 로그 추가
const addEventLog = (log: any) => {
  const timestamp = new Date().toLocaleTimeString('ko-KR')
  eventLogs.value.unshift({
    timestamp,
    operation: log.operation,
    allowed: log.allowed,
    reason: log.reason,
    details: log.details
  })
  
  // 최대 100개 로그만 유지
  if (eventLogs.value.length > 100) {
    eventLogs.value = eventLogs.value.slice(0, 100)
  }
  
  // 스크롤을 최상단으로
  nextTick(() => {
    const logContent = document.querySelector('.log-content')
    if (logContent) {
      logContent.scrollTop = 0
    }
  })
}

// 로그 지우기
const clearLogs = () => {
  eventLogs.value = []
}

// 역할 변경
const onRoleChange = () => {
  addEventLog({
    operation: 'roleChange',
    allowed: true,
    reason: `역할이 ${currentRoleDisplay.value}(으)로 변경되었습니다.`
  })
}

// 컨텐츠 준비 완료
const onContentReady = (e: DiagramContentReadyEvent) => {
  const diagram = e.component
  const items = diagram.getItems()
  shapeCount.value = items.filter(item => item.itemType === 'shape').length
  connectionCount.value = items.filter(item => item.itemType === 'connection').length
  
  addEventLog({
    operation: 'contentReady',
    allowed: true,
    reason: '다이어그램이 준비되었습니다.'
  })
  
  // 초기 데이터 로드
  if (items.length === 0) {
    loadInitialData()
  }
}

// 항목 클릭
const onItemClick = (e: DiagramItemClickEvent) => {
  console.log('항목 클릭:', e.item)
}

// 항목 더블 클릭
const onItemDoubleClick = (e: DiagramItemDoubleClickEvent) => {
  console.log('항목 더블 클릭:', e.item)
}

// 선택 변경
const onSelectionChanged = (e: DiagramSelectionChangedEvent) => {
  selectedItemsCount.value = e.items.length
}

// 초기 데이터 로드
const loadInitialData = () => {
  const diagram = diagramRef.value?.instance
  if (!diagram) return
  
  const initialData = {
    nodeKeyProperty: 'id',
    edgeKeyProperty: 'id',
    nodes: [
      { id: 1, text: '시작', type: 'terminator', x: 100, y: 100, isRoot: true },
      { id: 2, text: '처리 1', type: 'process', x: 300, y: 100 },
      { id: 3, text: '결정', type: 'decision', x: 500, y: 100 },
      { id: 4, text: '처리 2', type: 'process', x: 700, y: 50 },
      { id: 5, text: '처리 3', type: 'process', x: 700, y: 150 },
      { id: 6, text: '종료', type: 'terminator', x: 900, y: 100 }
    ],
    edges: [
      { id: 1, from: 1, to: 2, text: '' },
      { id: 2, from: 2, to: 3, text: '' },
      { id: 3, from: 3, to: 4, text: '예' },
      { id: 4, from: 3, to: 5, text: '아니오' },
      { id: 5, from: 4, to: 6, text: '' },
      { id: 6, from: 5, to: 6, text: '' }
    ]
  }
  
  diagram.import(initialData)
  
  // 카운트 업데이트
  const items = diagram.getItems()
  shapeCount.value = items.filter(item => item.itemType === 'shape').length
  connectionCount.value = items.filter(item => item.itemType === 'connection').length
}

// 샘플 데이터 로드
const loadSampleData = () => {
  loadInitialData()
  addEventLog({
    operation: 'loadSampleData',
    allowed: true,
    reason: '샘플 데이터를 로드했습니다.'
  })
}

// 컨텍스트 메뉴 명령어
const contextMenuCommands = ref([
  {
    name: 'protect',
    text: '보호 설정/해제',
    icon: 'lock',
    visible: (component: any, args: any) => args.shape && currentRole.value === 'admin',
    execute: (component: any, args: any) => {
      const shape = args.shape
      if (shape.dataItem) {
        shape.dataItem.protected = !shape.dataItem.protected
        diagramRef.value?.instance.repaint()
        addEventLog({
          operation: 'toggleProtection',
          allowed: true,
          reason: `도형 보호가 ${shape.dataItem.protected ? '설정' : '해제'}되었습니다.`
        })
      }
    }
  },
  {
    name: 'properties',
    text: '속성',
    icon: 'info',
    visible: () => true,
    execute: (component: any, args: any) => {
      console.log('도형 속성:', args.shape)
    }
  }
])
</script>

<style scoped>
.diagram-demo {
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

.controls select {
  padding: 6px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: white;
}

.controls button {
  padding: 6px 16px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: white;
  cursor: pointer;
  transition: all 0.2s;
}

.controls button:hover {
  background: #f8f9fa;
  border-color: #999;
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

.log-panel {
  width: 400px;
  background: white;
  display: flex;
  flex-direction: column;
  border-left: 1px solid #e0e0e0;
}

.log-header {
  padding: 16px;
  border-bottom: 1px solid #e0e0e0;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.log-header h3 {
  margin: 0;
  font-size: 16px;
  font-weight: 600;
  color: #333;
}

.log-stats {
  display: flex;
  gap: 12px;
  font-size: 12px;
  color: #666;
}

.log-content {
  flex: 1;
  overflow-y: auto;
  padding: 8px;
}

.log-item {
  padding: 8px 12px;
  margin-bottom: 4px;
  border-radius: 4px;
  border-left: 3px solid;
  font-size: 12px;
  line-height: 1.4;
}

.log-item.allowed {
  background-color: #f0f9ff;
  border-left-color: #3b82f6;
}

.log-item.denied {
  background-color: #fef2f2;
  border-left-color: #ef4444;
}

.log-time {
  color: #666;
  font-size: 10px;
  margin-bottom: 2px;
}

.log-operation {
  font-weight: 600;
  color: #333;
  margin-bottom: 2px;
}

.log-status {
  margin-bottom: 2px;
}

.status-badge {
  padding: 2px 6px;
  border-radius: 2px;
  font-size: 10px;
  font-weight: 600;
  text-transform: uppercase;
}

.status-badge.success {
  background-color: #10b981;
  color: white;
}

.status-badge.error {
  background-color: #ef4444;
  color: white;
}

.log-reason {
  color: #666;
  font-style: italic;
}

.status-bar {
  background: white;
  padding: 8px 24px;
  border-top: 1px solid #e0e0e0;
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 12px;
  color: #666;
}

.status-info {
  display: flex;
  gap: 24px;
}

.status-info span {
  display: flex;
  align-items: center;
}

.last-operation {
  color: #999;
}

.decision-shape {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  transform: rotate(45deg);
  background-color: #ffeaa7;
  border: 2px solid #fdcb6e;
}

.decision-text {
  transform: rotate(-45deg);
  font-weight: bold;
  text-align: center;
  font-size: 12px;
}

/* 스크롤바 스타일 */
.log-content::-webkit-scrollbar {
  width: 6px;
}

.log-content::-webkit-scrollbar-track {
  background: #f1f1f1;
}

.log-content::-webkit-scrollbar-thumb {
  background: #c1c1c1;
  border-radius: 3px;
}

.log-content::-webkit-scrollbar-thumb:hover {
  background: #a8a8a8;
}
</style>
