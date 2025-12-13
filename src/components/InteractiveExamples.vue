<template>
  <div class="interactive-examples">
    <div class="examples-header">
      <h1>인터랙티브 예제</h1>
      <div class="example-controls">
        <button 
          v-for="example in examples" 
          :key="example.id"
          @click="loadExample(example.id)"
          :class="['example-btn', { active: currentExample?.id === example.id }]"
        >
          {{ example.name }}
        </button>
      </div>
    </div>
    
    <div class="examples-content">
      <div class="code-panel">
        <div class="code-header">
          <h3>{{ currentExample?.name }} 코드</h3>
          <div class="code-actions">
            <button @click="copyCode" class="copy-btn">복사</button>
            <button @click="resetExample" class="reset-btn">초기화</button>
          </div>
        </div>
        <div class="code-content">
          <pre><code>{{ currentExample?.code }}</code></pre>
        </div>
      </div>
      
      <div class="demo-panel">
        <div class="demo-header">
          <h3>실행 결과</h3>
          <div class="demo-controls">
            <button @click="runExample" class="run-btn">실행</button>
            <button @click="clearDemo" class="clear-btn">지우기</button>
          </div>
        </div>
        <div class="demo-content">
          <DxDiagram
            ref="demoDiagramRef"
            @request-edit-operation="onDemoRequestEditOperation"
            :custom-shape-render="demoCustomShapeRender"
          >
            <DxToolbox :visible="true" />
          </DxDiagram>
        </div>
        
        <div class="demo-log">
          <h4>실행 로그</h4>
          <div class="log-content">
            <div 
              v-for="(log, index) in demoLogs" 
              :key="index"
              :class="['log-item', log.type]"
            >
              <span class="log-time">{{ log.time }}</span>
              <span class="log-message">{{ log.message }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>
    
    <div class="examples-description">
      <div class="description-content">
        <h3>{{ currentExample?.name }} 설명</h3>
        <p>{{ currentExample?.description }}</p>
        
        <div class="features" v-if="currentExample?.features">
          <h4>주요 기능</h4>
          <ul>
            <li v-for="feature in currentExample.features" :key="feature">
              {{ feature }}
            </li>
          </ul>
        </div>
        
        <div class="usage-tips" v-if="currentExample?.tips">
          <h4>사용 팁</h4>
          <ul>
            <li v-for="tip in currentExample.tips" :key="tip">
              {{ tip }}
            </li>
          </ul>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { DxDiagram, DxToolbox } from 'devextreme-vue/diagram'
import type { RequestEditOperationEvent, DiagramCustomShapeRenderEvent } from 'devextreme/ui/diagram'

// 예제 정의
const examples = ref([
  {
    id: 'basic-validation',
    name: '기본 유효성 검사',
    description: '도형 추가, 삭제, 이동에 대한 기본적인 유효성 검사를 구현합니다.',
    features: [
      '최대 도형 수 제한',
      '도형 이동 범위 제한',
      '기본 권한 검사',
      '작업 거부 시 사용자 피드백'
    ],
    tips: [
      '도형을 마음껏 추가하고 이동해보세요',
      '최대 도형 수(10개)를 초과하면 추가가 제한됩니다',
      '도형은 지정된 영역 내에서만 이동 가능합니다'
    ],
    code: `const onRequestEditOperation = (e: RequestEditOperationEvent) => {
  switch (e.operation) {
    case 'addShape':
      // 최대 도형 수 제한
      const maxShapes = 10;
      const currentCount = diagram.getItems().length;
      
      if (currentCount >= maxShapes) {
        e.allowed = false;
        e.reason = \`최대 \${maxShapes}개의 도형까지만 추가 가능합니다.\`;
      }
      break;
      
    case 'moveShape':
      // 이동 범위 제한
      const position = e.args.position;
      const bounds = { minX: 0, maxX: 800, minY: 0, maxY: 600 };
      
      if (position.x < bounds.minX || position.x > bounds.maxX ||
          position.y < bounds.minY || position.y > bounds.maxY) {
        e.allowed = false;
        e.reason = '도형은 지정된 영역 내에서만 이동 가능합니다.';
      }
      break;
      
    case 'deleteShape':
      // 기본 삭제 허용
      e.allowed = true;
      break;
      
    default:
      e.allowed = true;
  }
};`
  },
  {
    id: 'permission-based',
    name: '권한 기반 제어',
    description: '사용자 역할에 따른 편집 권한을 제어합니다.',
    features: [
      '관리자/편집자/뷰어 역할 구분',
      '역할별 작업 권한 설정',
      '동적 권한 변경',
      '권한 거부 사유 표시'
    ],
    tips: [
      '역할을 변경하면 즉시 권한이 적용됩니다',
      '뷰어는 어떤 편집 작업도 수행할 수 없습니다',
      '편집자는 삭제 작업만 수행할 수 없습니다'
    ],
    code: `interface UserPermission {
  canAdd: boolean;
  canDelete: boolean;
  canEdit: boolean;
  canMove: boolean;
}

const userPermissions: Record<string, UserPermission> = {
  admin: {
    canAdd: true,
    canDelete: true,
    canEdit: true,
    canMove: true
  },
  editor: {
    canAdd: true,
    canDelete: false,
    canEdit: true,
    canMove: true
  },
  viewer: {
    canAdd: false,
    canDelete: false,
    canEdit: false,
    canMove: false
  }
};

const currentRole = ref<string>('editor');

const onRequestEditOperation = (e: RequestEditOperationEvent) => {
  const permission = userPermissions[currentRole.value];
  
  switch (e.operation) {
    case 'addShape':
      e.allowed = permission.canAdd;
      if (!e.allowed) {
        e.reason = \`\${currentRole.value}는 도형을 추가할 수 없습니다.\`;
      }
      break;
      
    case 'deleteShape':
    case 'deleteConnection':
      e.allowed = permission.canDelete;
      if (!e.allowed) {
        e.reason = \`\${currentRole.value}는 삭제할 수 없습니다.\`;
      }
      break;
      
    case 'changeShapeText':
    case 'changeConnection':
      e.allowed = permission.canEdit;
      if (!e.allowed) {
        e.reason = \`\${currentRole.value}는 편집할 수 없습니다.\`;
      }
      break;
      
    case 'moveShape':
      e.allowed = permission.canMove;
      if (!e.allowed) {
        e.reason = \`\${currentRole.value}는 이동할 수 없습니다.\`;
      }
      break;
      
    default:
      e.allowed = true;
  }
};`
  },
  {
    id: 'business-rules',
    name: '비즈니스 규칙',
    description: '복잡한 비즈니스 규칙을 적용하여 다이어그램의 무결성을 유지합니다.',
    features: [
      '고아 도형 방지',
      '최대 깊이 제한',
      '순환 참조 방지',
      '도형 타입별 제약'
    ],
    tips: [
      '연결되지 않은 도형은 허용되지 않습니다',
      '최대 3단계 깊이까지만 허용됩니다',
      '특정 도형 타입은 다른 타입과만 연결될 수 있습니다'
    ],
    code: `interface BusinessRule {
  id: string;
  name: string;
  condition: (operation: string, args: any) => boolean;
  action: 'allow' | 'deny';
  reason: string;
  priority: number;
}

const businessRules: BusinessRule[] = [
  {
    id: 'no_orphan_shapes',
    name: '고아 도형 금지',
    condition: (operation, args) => {
      if (operation === 'deleteConnection') {
        const shape = args.shape;
        const connections = diagram.getConnections()
          .filter(conn => conn.from === shape.id || conn.to === shape.id);
        return connections.length === 1;
      }
      return false;
    },
    action: 'deny',
    reason: '연결되지 않은 도형은 허용되지 않습니다.',
    priority: 1
  },
  {
    id: 'max_depth_limit',
    name: '최대 깊이 제한',
    condition: (operation, args) => {
      if (operation === 'addConnection') {
        const toShape = args.connection.to;
        return calculateDepth(toShape) >= 3;
      }
      return false;
    },
    action: 'deny',
    reason: '최대 3단계 깊이까지만 허용됩니다.',
    priority: 2
  }
];

const onRequestEditOperation = (e: RequestEditOperationEvent) => {
  const applicableRules = businessRules
    .filter(rule => rule.condition(e.operation, e.args))
    .sort((a, b) => b.priority - a.priority);
  
  if (applicableRules.length > 0) {
    const rule = applicableRules[0];
    e.allowed = rule.action === 'allow';
    e.reason = rule.reason;
  } else {
    e.allowed = true;
  }
};

function calculateDepth(shape: any): number {
  // 깊이 계산 로직 구현
  return 0;
}`
  },
  {
    id: 'advanced-validation',
    name: '고급 유효성 검사',
    description: '캐싱, 디바운싱, 비동기 검증 등 고급 기법을 활용한 성능 최적화.',
    features: [
      '결과 캐싱으로 성능 향상',
      '디바운싱으로 불필요한 검사 방지',
      '비동기 서버 검증',
      '실시간 메트릭 수집'
    ],
    tips: [
      '최적화를 토글하여 성능 차이를 확인해보세요',
      '빠른 연속 작업에서 캐싱 효과를 볼 수 있습니다',
      '네트워크 지연 시에도 반응성이 유지됩니다'
    ],
    code: `const validationCache = new Map<string, { result: boolean; timestamp: number }>();
const CACHE_DURATION = 2000; // 2초

const onRequestEditOperation = (e: RequestEditOperationEvent) => {
  const startTime = performance.now();
  const cacheKey = generateCacheKey(e.operation, e.args);
  
  // 캐시 확인
  const cached = getCachedResult(cacheKey);
  if (cached) {
    e.allowed = cached.result;
    e.reason = cached.reason;
    return;
  }
  
  // 빠른 유효성 검사
  const result = performFastValidation(e);
  
  // 결과 캐싱
  cacheResult(cacheKey, result);
  
  // 백그라운드에서 상세 검증 (비동기)
  if (result.allowed) {
    performAsyncValidation(e).then(asyncResult => {
      if (asyncResult.result !== result.result) {
        // 결과가 다르면 캐시 업데이트
        updateCacheWithAsyncResult(cacheKey, asyncResult);
      }
    });
  }
  
  e.allowed = result.result;
  e.reason = result.reason;
  
  // 메트릭 수집
  const duration = performance.now() - startTime;
  collectMetrics(e.operation, duration, result.result);
};

function generateCacheKey(operation: string, args: any): string {
  return \`\${operation}_\${JSON.stringify(args)}\`;
}

function getCachedResult(key: string): { result: boolean; reason?: string } | null {
  const cached = validationCache.get(key);
  if (!cached) return null;
  
  if (Date.now() - cached.timestamp > CACHE_DURATION) {
    validationCache.delete(key);
    return null;
  }
  
  return cached;
}

function cacheResult(key: string, result: { result: boolean; reason?: string }) {
  validationCache.set(key, {
    result: result.result,
    reason: result.reason,
    timestamp: Date.now()
  });
}`
  }
])

// 상태 관리
const currentExample = ref(examples.value[0])
const demoDiagramRef = ref<InstanceType<typeof DxDiagram>>()
const demoLogs = ref<Array<{
  time: string
  message: string
  type: 'info' | 'success' | 'error'
}>>([])

// 계산된 속성
const currentExampleCode = computed(() => currentExample.value?.code || '')

// 예제 로드
const loadExample = (exampleId: string) => {
  const example = examples.value.find(ex => ex.id === exampleId)
  if (example) {
    currentExample.value = example
    addDemoLog(`예제 '${example.name}'이(가) 로드되었습니다.`, 'info')
    clearDemoData()
  }
}

// 코드 복사
const copyCode = async () => {
  try {
    await navigator.clipboard.writeText(currentExampleCode.value)
    addDemoLog('코드가 클립보드에 복사되었습니다.', 'success')
  } catch (error) {
    addDemoLog('코드 복사에 실패했습니다.', 'error')
  }
}

// 예제 초기화
const resetExample = () => {
  clearDemoData()
  addDemoLog('예제가 초기화되었습니다.', 'info')
}

// 예제 실행
const runExample = () => {
  addDemoLog(`'${currentExample.value.name}' 예제를 실행합니다.`, 'info')
  loadDemoData()
}

// 데모 지우기
const clearDemo = () => {
  clearDemoData()
  addDemoLog('데모 데이터가 지워졌습니다.', 'info')
}

// 데모 데이터 로드
const loadDemoData = () => {
  const diagram = demoDiagramRef.value?.instance
  if (!diagram) return
  
  const sampleData = {
    nodeKeyProperty: 'id',
    edgeKeyProperty: 'id',
    nodes: [
      { id: 1, text: '시작', type: 'terminator', x: 100, y: 100 },
      { id: 2, text: '처리', type: 'process', x: 300, y: 100 },
      { id: 3, text: '결정', type: 'decision', x: 500, y: 100 },
      { id: 4, text: '종료', type: 'terminator', x: 700, y: 100 }
    ],
    edges: [
      { id: 1, from: 1, to: 2, text: '' },
      { id: 2, from: 2, to: 3, text: '' },
      { id: 3, from: 3, to: 4, text: '예' }
    ]
  }
  
  diagram.import(sampleData)
  addDemoLog('샘플 데이터가 로드되었습니다.', 'success')
}

// 데모 데이터 지우기
const clearDemoData = () => {
  const diagram = demoDiagramRef.value?.instance
  if (!diagram) return
  
  diagram.import({
    nodeKeyProperty: 'id',
    edgeKeyProperty: 'id',
    nodes: [],
    edges: []
  })
}

// 데모 로그 추가
const addDemoLog = (message: string, type: 'info' | 'success' | 'error' = 'info') => {
  const time = new Date().toLocaleTimeString('ko-KR')
  demoLogs.value.unshift({ time, message, type })
  
  // 최근 50개만 유지
  if (demoLogs.value.length > 50) {
    demoLogs.value = demoLogs.value.slice(0, 50)
  }
}

// 데모용 onRequestEditOperation
const onDemoRequestEditOperation = (e: RequestEditOperationEvent) => {
  const operation = e.operation
  
  try {
    switch (currentExample.value.id) {
      case 'basic-validation':
        handleBasicValidation(e)
        break
      case 'permission-based':
        handlePermissionBased(e)
        break
      case 'business-rules':
        handleBusinessRules(e)
        break
      case 'advanced-validation':
        handleAdvancedValidation(e)
        break
      default:
        e.allowed = true
    }
    
    addDemoLog(
      `\${operation} 작업: \${e.allowed ? '허용' : '거부'}\${e.reason ? ' - ' + e.reason : ''}`,
      e.allowed ? 'success' : 'error'
    )
    
  } catch (error) {
    console.error('데모 처리 오류:', error)
    e.allowed = false
    e.reason = '처리 중 오류 발생'
    addDemoLog(`\${operation} 작업: 오류 발생`, 'error')
  }
}

// 기본 유효성 검사 처리
const handleBasicValidation = (e: RequestEditOperationEvent) => {
  switch (e.operation) {
    case 'addShape':
      const maxShapes = 10
      const diagram = demoDiagramRef.value?.instance
      const currentCount = diagram ? diagram.getItems().length : 0
      
      if (currentCount >= maxShapes) {
        e.allowed = false
        e.reason = `최대 ${maxShapes}개의 도형까지만 추가 가능합니다.`
      } else {
        e.allowed = true
      }
      break
      
    case 'moveShape':
      const position = e.args.position
      const bounds = { minX: 0, maxX: 800, minY: 0, maxY: 600 }
      
      if (position.x < bounds.minX || position.x > bounds.maxX ||
          position.y < bounds.minY || position.y > bounds.maxY) {
        e.allowed = false
        e.reason = '도형은 지정된 영역 내에서만 이동 가능합니다.'
      } else {
        e.allowed = true
      }
      break
      
    default:
      e.allowed = true
  }
}

// 권한 기반 처리
const handlePermissionBased = (e: RequestEditOperationEvent) => {
  const permissions = {
    canAdd: ['admin', 'editor'].includes(currentRole.value),
    canDelete: currentRole.value === 'admin',
    canEdit: ['admin', 'editor'].includes(currentRole.value),
    canMove: ['admin', 'editor'].includes(currentRole.value)
  }
  
  switch (e.operation) {
    case 'addShape':
      e.allowed = permissions.canAdd
      if (!e.allowed) e.reason = `${currentRole.value}는 도형을 추가할 수 없습니다.`
      break
      
    case 'deleteShape':
    case 'deleteConnection':
      e.allowed = permissions.canDelete
      if (!e.allowed) e.reason = `${currentRole.value}는 삭제할 수 없습니다.`
      break
      
    case 'changeShapeText':
    case 'changeConnection':
      e.allowed = permissions.canEdit
      if (!e.allowed) e.reason = `${currentRole.value}는 편집할 수 없습니다.`
      break
      
    case 'moveShape':
      e.allowed = permissions.canMove
      if (!e.allowed) e.reason = `${currentRole.value}는 이동할 수 없습니다.`
      break
      
    default:
      e.allowed = true
  }
}

// 비즈니스 규칙 처리
const handleBusinessRules = (e: RequestEditOperationEvent) => {
  const diagram = demoDiagramRef.value?.instance
  
  switch (e.operation) {
    case 'addConnection':
      const connection = e.args.connection
      
      // 자기 자신으로의 연결 금지
      if (connection.from.id === connection.to.id) {
        e.allowed = false
        e.reason = '자기 자신으로의 연결은 허용되지 않습니다.'
        return
      }
      
      // 고아 도형 방지 (연결되지 않은 도형 금지)
      const orphanedShapes = diagram ? diagram.getItems()
        .filter(item => item.itemType === 'shape')
        .filter(shape => {
          const connections = diagram.getConnections()
            .filter(conn => conn.from === shape.id || conn.to === shape.id)
          return connections.length === 0
        }) : []
      
      if (orphanedShapes.length > 0) {
        e.allowed = false
        e.reason = '연결되지 않은 도형은 허용되지 않습니다.'
        return
      }
      
      e.allowed = true
      break
      
    case 'deleteConnection':
      // 연결 삭제 시 고아 도형 방지
      const shape = e.args.shape
      if (shape) {
        const connections = diagram ? diagram.getConnections()
          .filter(conn => conn.from === shape.id || conn.to === shape.id) : []
        
        if (connections.length <= 1) {
          e.allowed = false
          e.reason = '연결되지 않은 도형은 허용되지 않습니다.'
          return
        }
      }
      
      e.allowed = true
      break
      
    default:
      e.allowed = true
  }
}

// 고급 유효성 검사 처리
const handleAdvancedValidation = (e: RequestEditOperationEvent) => {
  // 간단한 캐싱 시뮬레이션
  const cacheKey = `${e.operation}_${Date.now() % 1000}`
  const isCached = Math.random() > 0.7 // 30% 확률로 캐시 적중 시뮬레이션
  
  if (isCached) {
    e.allowed = true
    e.reason = '캐시에서 빠른 결과 반환'
    return
  }
  
  // 실제 검증 로직
  switch (e.operation) {
    case 'addShape':
      const diagram = demoDiagramRef.value?.instance
      const count = diagram ? diagram.getItems().length : 0
      e.allowed = count < 15
      if (!e.allowed) e.reason = '최대 도형 수를 초과했습니다.'
      break
      
    case 'moveShape':
      const pos = e.args.position
      e.allowed = pos.x >= 0 && pos.y >= 0 && pos.x <= 1000 && pos.y <= 800
      if (!e.allowed) e.reason = '이동 범위를 벗어났습니다.'
      break
      
    default:
      e.allowed = true
  }
}

// 커스텀 도형 렌더링
const demoCustomShapeRender = (e: DiagramCustomShapeRenderEvent) => {
  if (e.shape.type === 'decision') {
    e.contentTemplate = (container: HTMLElement, shape: any) => {
      container.innerHTML = `
        <div style="
          width: 100%;
          height: 100%;
          display: flex;
          align-items: center;
          justify-content: center;
          transform: rotate(45deg);
          background-color: #ffeaa7;
          border: 2px solid #fdcb6e;
          border-radius: 4px;
        ">
          <div style="
            transform: rotate(-45deg);
            font-weight: bold;
            text-align: center;
            font-size: 12px;
            padding: 4px;
          ">
            ${shape.text || '결정'}
          </div>
        </div>
      `
    }
  }
}

// 역할 설정 (데모용)
const currentRole = ref<string>('editor')
</script>

<style scoped>
.interactive-examples {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background-color: #f5f5f5;
}

.examples-header {
  background: white;
  padding: 16px 24px;
  border-bottom: 1px solid #e0e0e0;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.examples-header h1 {
  margin: 0;
  font-size: 20px;
  font-weight: 600;
  color: #333;
}

.example-controls {
  display: flex;
  gap: 8px;
  align-items: center;
}

.example-btn {
  padding: 8px 16px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: white;
  cursor: pointer;
  transition: all 0.2s;
  font-size: 14px;
}

.example-btn:hover {
  background: #f8f9fa;
  border-color: #999;
}

.example-btn.active {
  background: #3b82f6;
  color: white;
  border-color: #3b82f6;
}

.examples-content {
  flex: 1;
  display: flex;
  overflow: hidden;
}

.code-panel {
  width: 600px;
  background: white;
  border-right: 1px solid #e0e0e0;
  display: flex;
  flex-direction: column;
}

.code-header {
  padding: 16px;
  border-bottom: 1px solid #e0e0e0;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.code-header h3 {
  margin: 0;
  font-size: 16px;
  font-weight: 600;
  color: #333;
}

.code-actions {
  display: flex;
  gap: 8px;
}

.copy-btn, .reset-btn {
  padding: 6px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: white;
  cursor: pointer;
  font-size: 12px;
  transition: all 0.2s;
}

.copy-btn:hover, .reset-btn:hover {
  background: #f8f9fa;
  border-color: #999;
}

.code-content {
  flex: 1;
  overflow: auto;
  padding: 16px;
  background: #f8f9fa;
}

.code-content pre {
  margin: 0;
  font-family: 'Consolas', 'Monaco', 'Courier New', monospace;
  font-size: 13px;
  line-height: 1.5;
  white-space: pre-wrap;
}

.code-content code {
  color: #333;
}

.demo-panel {
  flex: 1;
  background: white;
  display: flex;
  flex-direction: column;
}

.demo-header {
  padding: 16px;
  border-bottom: 1px solid #e0e0e0;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.demo-header h3 {
  margin: 0;
  font-size: 16px;
  font-weight: 600;
  color: #333;
}

.demo-controls {
  display: flex;
  gap: 8px;
}

.run-btn, .clear-btn {
  padding: 6px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: white;
  cursor: pointer;
  font-size: 12px;
  transition: all 0.2s;
}

.run-btn:hover, .clear-btn:hover {
  background: #f8f9fa;
  border-color: #999;
}

.run-btn {
  background: #10b981;
  color: white;
  border-color: #10b981;
}

.run-btn:hover {
  background: #059669;
  border-color: #059669;
}

.demo-content {
  flex: 1;
  min-height: 300px;
}

.demo-log {
  border-top: 1px solid #e0e0e0;
  background: #f8f9fa;
}

.demo-log h4 {
  margin: 0;
  padding: 12px 16px;
  font-size: 14px;
  font-weight: 600;
  color: #333;
  border-bottom: 1px solid #e0e0e0;
}

.log-content {
  max-height: 200px;
  overflow-y: auto;
  padding: 8px;
}

.log-item {
  display: flex;
  gap: 8px;
  padding: 4px 8px;
  margin-bottom: 2px;
  border-radius: 4px;
  font-size: 12px;
  font-family: 'Consolas', 'Monaco', 'Courier New', monospace;
}

.log-item.info {
  background-color: #e0f2fe;
  color: #0277bd;
}

.log-item.success {
  background-color: #e8f5e8;
  color: #2e7d32;
}

.log-item.error {
  background-color: #ffebee;
  color: #c62828;
}

.log-time {
  color: #666;
  min-width: 60px;
}

.log-message {
  flex: 1;
}

.examples-description {
  background: white;
  border-top: 1px solid #e0e0e0;
  padding: 24px;
}

.description-content h3 {
  margin: 0 0 16px 0;
  font-size: 18px;
  font-weight: 600;
  color: #333;
}

.description-content p {
  margin: 0 0 16px 0;
  line-height: 1.6;
  color: #666;
}

.features, .usage-tips {
  margin-top: 20px;
}

.features h4, .usage-tips h4 {
  margin: 0 0 12px 0;
  font-size: 16px;
  font-weight: 600;
  color: #333;
}

.features ul, .usage-tips ul {
  margin: 0;
  padding-left: 20px;
}

.features li, .usage-tips li {
  margin-bottom: 8px;
  line-height: 1.5;
  color: #666;
}

/* 스크롤바 스타일 */
.code-content::-webkit-scrollbar,
.log-content::-webkit-scrollbar {
  width: 6px;
}

.code-content::-webkit-scrollbar-track,
.log-content::-webkit-scrollbar-track {
  background: #f1f1f1;
}

.code-content::-webkit-scrollbar-thumb,
.log-content::-webkit-scrollbar-thumb {
  background: #c1c1c1;
  border-radius: 3px;
}

.code-content::-webkit-scrollbar-thumb:hover,
.log-content::-webkit-scrollbar-thumb:hover {
  background: #a8a8a8;
}
</style>