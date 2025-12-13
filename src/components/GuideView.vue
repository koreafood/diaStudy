<template>
  <div class="guide-view">
    <div class="guide-header">
      <h1>DevExpress Diagram onRequestEditOperation 학습 가이드</h1>
      <p>Vue3 + TypeScript 환경에서 DevExpress Diagram의 onRequestEditOperation 이벤트를 완벽하게 마스터하세요.</p>
    </div>
    
    <div class="guide-content">
      <div class="guide-sidebar">
        <div class="sidebar-nav">
          <div 
            v-for="section in sections" 
            :key="section.id"
            @click="scrollToSection(section.id)"
            :class="['nav-item', { active: activeSection === section.id }]"
          >
            {{ section.title }}
          </div>
        </div>
      </div>
      
      <div class="guide-main">
        <div class="section" id="introduction">
          <h2>1. 소개</h2>
          <div class="content-block">
            <h3>onRequestEditOperation 이벤트란?</h3>
            <p>
              <code>onRequestEditOperation</code>은 DevExpress Diagram 컴포넌트에서 사용자의 편집 작업을 제어하는 핵심 이벤트입니다. 
              이 이벤트를 통해 특정 조건에서만 작업을 허용하거나 완전히 차단할 수 있으며, 다이어그램의 무결성과 일관성을 유지하는 데 필수적입니다.
            </p>
            
            <div class="key-features">
              <h4>주요 특징</h4>
              <ul>
                <li><strong>실시간 제어:</strong> 모든 편집 작업에 대한 실시간 검증</li>
                <li><strong>유연한 조건:</strong> 복잡한 비즈니스 규칙 적용 가능</li>
                <li><strong>사용자 피드백:</strong> 작업 거부 시 명확한 이유 제공</li>
                <li><strong>성능 최적화:</strong> 캐싱 및 지연 로딩 지원</li>
              </ul>
            </div>
          </div>
        </div>
        
        <div class="section" id="basic-usage">
          <h2>2. 기본 사용법</h2>
          <div class="content-block">
            <h3>이벤트 핸들러 등록</h3>
            <div class="code-example">
              <pre><code>&lt;template&gt;
  &lt;DxDiagram
    @request-edit-operation="onRequestEditOperation"
  /&gt;
&lt;/template&gt;

&lt;script setup lang="ts"&gt;
import { DxDiagram } from 'devextreme-vue/diagram';
import type { RequestEditOperationEvent } from 'devextreme/ui/diagram';

const onRequestEditOperation = (e: RequestEditOperationEvent) => {
  // 기본적으로 모든 작업 허용
  e.allowed = true;
  
  // 특정 조건에서 작업 제한
  if (e.operation === 'deleteShape') {
    const shape = e.args.shape;
    if (shape.type === 'root') {
      e.allowed = false;
      e.reason = '루트 노드는 삭제할 수 없습니다.';
    }
  }
};
&lt;/script&gt;</code></pre>
            </div>
          </div>
        </div>
        
        <div class="section" id="event-structure">
          <h2>3. 이벤트 구조 분석</h2>
          <div class="content-block">
            <h3>RequestEditOperationEvent 인터페이스</h3>
            <div class="interface-definition">
              <pre><code>interface RequestEditOperationEvent {
  operation: DiagramEditOperation;  // 작업 유형
  allowed: boolean;                   // 허용 여부
  updateUI: boolean;                  // UI 업데이트 필요 여부
  reason: string;                      // 거부 사유
  args: {                              // 작업 관련 인자
    shape?: DiagramShape;             // 도형 관련 작업
    connection?: DiagramConnection;   // 연결선 관련 작업
    position?: { x: number; y: number }; // 위치 정보
    text?: string;                    // 텍스트 정보
    // ... 기타 작업별 인자
  };
}</code></pre>
            </div>
            
            <h3>주요 작업 유형</h3>
            <div class="operation-types">
              <div class="operation-item">
                <code>addShape</code>
                <span>새 도형 추가</span>
              </div>
              <div class="operation-item">
                <code>deleteShape</code>
                <span>도형 삭제</span>
              </div>
              <div class="operation-item">
                <code>moveShape</code>
                <span>도형 이동</span>
              </div>
              <div class="operation-item">
                <code>changeShapeText</code>
                <span>도형 텍스트 변경</span>
              </div>
              <div class="operation-item">
                <code>addConnection</code>
                <span>연결선 추가</span>
              </div>
              <div class="operation-item">
                <code>deleteConnection</code>
                <span>연결선 삭제</span>
              </div>
              <div class="operation-item">
                <code>changeConnection</code>
                <span>연결선 변경</span>
              </div>
              <div class="operation-item">
                <code>resizeShape</code>
                <span>도형 크기 조정</span>
              </div>
            </div>
          </div>
        </div>
        
        <div class="section" id="validation-patterns">
          <h2>4. 검증 패턴</h2>
          <div class="content-block">
            <h3>1. 기본 제한 패턴</h3>
            <div class="code-example">
              <pre><code>// 최대 도형 수 제한
const MAX_SHAPES = 50;
const onRequestEditOperation = (e: RequestEditOperationEvent) => {
  if (e.operation === 'addShape') {
    const currentCount = diagram.getItems().length;
    if (currentCount >= MAX_SHAPES) {
      e.allowed = false;
      e.reason = `최대 ${MAX_SHAPES}개의 도형까지만 추가 가능합니다.`;
    }
  }
};</code></pre>
            </div>
            
            <h3>2. 권한 기반 제어</h3>
            <div class="code-example">
              <pre><code>const userPermissions = {
  admin: { canAdd: true, canDelete: true, canEdit: true },
  editor: { canAdd: true, canDelete: false, canEdit: true },
  viewer: { canAdd: false, canDelete: false, canEdit: false }
};

const onRequestEditOperation = (e: RequestEditOperationEvent) => {
  const permission = userPermissions[currentUserRole];
  
  switch (e.operation) {
    case 'addShape':
      e.allowed = permission.canAdd;
      break;
    case 'deleteShape':
      e.allowed = permission.canDelete;
      break;
    case 'changeShapeText':
      e.allowed = permission.canEdit;
      break;
  }
  
  if (!e.allowed) {
    e.reason = '권한이 부족합니다.';
  }
};</code></pre>
            </div>
            
            <h3>3. 비즈니스 규칙 엔진</h3>
            <div class="code-example">
              <pre><code>interface BusinessRule {
  id: string;
  condition: (operation: string, args: any) => boolean;
  action: 'allow' | 'deny';
  reason: string;
  priority: number;
}

const businessRules: BusinessRule[] = [
  {
    id: 'no_orphan_shapes',
    condition: (op, args) => {
      // 고아 도형 방지 로직
      return op === 'deleteConnection' && wouldCreateOrphan(args);
    },
    action: 'deny',
    reason: '연결되지 않은 도형은 허용되지 않습니다.',
    priority: 1
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
};</code></pre>
            </div>
          </div>
        </div>
        
        <div class="section" id="performance-optimization">
          <h2>5. 성능 최적화</h2>
          <div class="content-block">
            <h3>1. 캐싱 전략</h3>
            <div class="code-example">
              <pre><code>const validationCache = new Map&lt;string, { result: boolean; timestamp: number }&gt;();
const CACHE_DURATION = 5000; // 5초

const onRequestEditOperation = (e: RequestEditOperationEvent) => {
  const cacheKey = generateCacheKey(e.operation, e.args);
  const cached = getCachedResult(cacheKey);
  
  if (cached) {
    e.allowed = cached.result;
    return;
  }
  
  // 새로운 검사 수행
  const result = performValidation(e);
  cacheResult(cacheKey, result);
  
  e.allowed = result;
};</code></pre>
            </div>
            
            <h3>2. 디바운싱</h3>
            <div class="code-example">
              <pre><code>import { debounce } from 'lodash-es';

const debouncedValidation = debounce((e: RequestEditOperationEvent) => {
  // 무거운 유효성 검사 로직
  performHeavyValidation(e);
}, 300);

const onRequestEditOperation = (e: RequestEditOperationEvent) => {
  // 가벼운 검사는 즉시 수행
  if (!performLightValidation(e)) {
    e.allowed = false;
    return;
  }
  
  // 무거운 검사는 디바운스 적용
  debouncedValidation(e);
};</code></pre>
            </div>
            
            <h3>3. 비동기 처리</h3>
            <div class="code-example">
              <pre><code>const onRequestEditOperation = async (e: RequestEditOperationEvent) => {
  // 즉시 UI 피드백을 위한 기본값 설정
  e.allowed = true;
  
  try {
    // 백그라운드에서 서버 검증
    const validation = await validateOperationOnServer({
      operation: e.operation,
      args: e.args,
      userId: currentUserId.value
    });
    
    // 검증 결과 반영
    e.allowed = validation.allowed;
    e.reason = validation.reason;
    
    if (e.updateUI) {
      diagramInstance.repaint();
    }
  } catch (error) {
    console.error('검증 오류:', error);
    e.allowed = false;
    e.reason = '검증 중 오류가 발생했습니다.';
  }
};</code></pre>
            </div>
          </div>
        </div>
        
        <div class="section" id="best-practices">
          <h2>6. 모범 사례</h2>
          <div class="content-block">
            <div class="best-practice-item">
              <h4>✅ 할 것</h4>
              <ul>
                <li>명확한 거부 사유 제공</li>
                <li>가능한 한 빠른 검증 수행</li>
                <li>복잡한 규칙은 캐싱 활용</li>
                <li>에러 처리 구현</li>
                <li>성능 모니터링</li>
              </ul>
            </div>
            
            <div class="best-practice-item">
              <h4>❌ 하지 말 것</h4>
              <ul>
                <li>무거운 동기식 연산 수행</li>
                <li>불필요한 상태 변경</li>
                <li>모호한 거부 사유</li>
                <li>예외 처리 누락</li>
                <li>과도한 로깅</li>
              </ul>
            </div>
            
            <h3>디버깅 팁</h3>
            <div class="debugging-tips">
              <pre><code>const onRequestEditOperation = (e: RequestEditOperationEvent) => {
  console.group(`편집 작업: ${e.operation}`);
  console.log('시간:', new Date().toISOString());
  console.log('인자:', e.args);
  console.log('사용자:', currentUser.value);
  
  try {
    // 검증 로직
    const result = validateOperation(e);
    e.allowed = result.allowed;
    e.reason = result.reason;
    
    console.log('결과:', result);
  } catch (error) {
    console.error('오류:', error);
    e.allowed = false;
    e.reason = '처리 중 오류 발생';
  } finally {
    console.groupEnd();
  }
};</code></pre>
            </div>
          </div>
        </div>
        
        <div class="section" id="common-scenarios">
          <h2>7. 일반적인 시나리오</h2>
        <div class="content-block">
            <div class="scenario-item">
              <h3>시나리오 1: 워크플로우 다이어그램</h3>
              <p>워크플로우 다이어그램에서 특정 단계는 필수이며 삭제할 수 없습니다.</p>
              <div class="code-example">
                <pre><code>const onRequestEditOperation = (e: RequestEditOperationEvent) => {
  if (e.operation === 'deleteShape') {
    const shape = e.args.shape;
    
    // 필수 단계 보호
    if (shape.dataItem?.isRequired) {
      e.allowed = false;
      e.reason = '이 단계는 필수 단계입니다. 삭제할 수 없습니다.';
      return;
    }
    
    // 시작/종료 노드 보호
    if (shape.type === 'terminator') {
      e.allowed = false;
      e.reason = '시작/종료 노드는 삭제할 수 없습니다.';
      return;
    }
  }
  
  e.allowed = true;
};</code></pre>
              </div>
            </div>
            
            <div class="scenario-item">
              <h3>시나리오 2: 조직도</h3>
              <p>조직도에서 관리자는 하위 직원만 추가할 수 있습니다.</p>
              <div class="code-example">
                <pre><code>const onRequestEditOperation = (e: RequestEditOperationEvent) => {
  if (e.operation === 'addConnection') {
    const connection = e.args.connection;
    const fromShape = connection.from;
    const toShape = connection.to;
    
    // 계층 구조 검증
    if (fromShape.dataItem?.level >= toShape.dataItem?.level) {
      e.allowed = false;
      e.reason = '상급자는 하위 직원만 연결할 수 있습니다.';
      return;
    }
    
    // 최대 보고선 제한
    const maxReports = 10;
    const currentReports = diagram.getConnections()
      .filter(conn => conn.from === fromShape.id).length;
      
    if (currentReports >= maxReports) {
      e.allowed = false;
      e.reason = `한 명의 관리자는 최대 ${maxReports}명의 하위 직원만 가질 수 있습니다.`;
      return;
    }
  }
  
  e.allowed = true;
};</code></pre>
              </div>
            </div>
            
            <div class="scenario-item">
              <h3>시나리오 3: 데이터 흐름도</h3>
              <p>데이터 흐름도에서 프로세스는 데이터 저장소와만 연결될 수 있습니다.</p>
              <div class="code-example">
                <pre><code>const onRequestEditOperation = (e: RequestEditOperationEvent) => {
  if (e.operation === 'addConnection') {
    const connection = e.args.connection;
    const fromType = connection.from.type;
    const toType = connection.to.type;
    
    // 유효한 연결 타입 검증
    const validConnections = [
      { from: 'process', to: 'datastore' },
      { from: 'datastore', to: 'process' },
      { from: 'process', to: 'external' },
      { from: 'external', to: 'process' }
    ];
    
    const isValid = validConnections.some(rule => 
      rule.from === fromType && rule.to === toType
    );
    
    if (!isValid) {
      e.allowed = false;
      e.reason = `${fromType} 타입은 ${toType} 타입과 연결될 수 없습니다.`;
      return;
    }
  }
  
  e.allowed = true;
};</code></pre>
              </div>
            </div>
          </div>
        </div>
        
        <div class="section" id="troubleshooting">
          <h2>8. 문제 해결</h2>
          <div class="content-block">
            <div class="troubleshooting-item">
              <h3>자주 발생하는 문제</h3>
              <div class="problem-solution">
                <div class="problem">
                  <h4>문제: 이벤트가 호출되지 않음</h4>
                  <p>onRequestEditOperation 이벤트가 트리거되지 않습니다.</p>
                </div>
                <div class="solution">
                  <h4>해결책</h4>
                  <ul>
                    <li>이벤트 이름이 올바른지 확인 (@request-edit-operation)</li>
                    <li>DevExpress 버전이 최신인지 확인</li>
                    <li>다이어그램이 편집 가능한 상태인지 확인</li>
                  </ul>
                </div>
              </div>
              
              <div class="problem-solution">
                <div class="problem">
                  <h4>문제: 성능 저하</h4>
                  <p>복잡한 검증 로직으로 인한 반응 속도 저하</p>
                </div>
                <div class="solution">
                  <h4>해결책</h4>
                  <ul>
                    <li>캐싱 메커니즘 구현</li>
                    <li>무거운 연산은 비동기로 처리</li>
                    <li>불필요한 검증 제거</li>
                    <li>debouncing/throttling 적용</li>
                  </ul>
                </div>
              </div>
              
              <div class="problem-solution">
                <div class="problem">
                  <h4>문제: 상태 불일치</h4>
                  <p>검증 결과와 실제 상태가 일치하지 않음</p>
                </div>
                <div class="solution">
                  <h4>해결책</h4>
                  <ul>
                    <li>상태 업데이트 타이밍 확인</li>
                    <li>비동기 처리 시 상태 동기화</li>
                    <li>UI 강제 새로고침 (updateUI 활용)</li>
                  </ul>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const activeSection = ref('introduction')

const sections = ref([
  { id: 'introduction', title: '1. 소개' },
  { id: 'basic-usage', title: '2. 기본 사용법' },
  { id: 'event-structure', title: '3. 이벤트 구조 분석' },
  { id: 'validation-patterns', title: '4. 검증 패턴' },
  { id: 'performance-optimization', title: '5. 성능 최적화' },
  { id: 'best-practices', title: '6. 모범 사례' },
  { id: 'common-scenarios', title: '7. 일반적인 시나리오' },
  { id: 'troubleshooting', title: '8. 문제 해결' }
])

const scrollToSection = (sectionId: string) => {
  const element = document.getElementById(sectionId)
  if (element) {
    element.scrollIntoView({ behavior: 'smooth' })
    activeSection.value = sectionId
  }
}

const updateActiveSection = () => {
  const guideMain = document.querySelector('.guide-main')
  if (!guideMain) return
  
  const scrollPosition = guideMain.scrollTop
  const sections = document.querySelectorAll('.section')
  
  for (let i = sections.length - 1; i >= 0; i--) {
    const section = sections[i]
    const sectionTop = section.offsetTop - 100
    
    if (scrollPosition >= sectionTop) {
      activeSection.value = section.id
      break
    }
  }
}

let scrollTimeout: number | null = null

const handleScroll = () => {
  if (scrollTimeout) {
    window.clearTimeout(scrollTimeout)
  }
  
  scrollTimeout = window.setTimeout(() => {
    updateActiveSection()
  }, 100)
}

onMounted(() => {
  const guideMain = document.querySelector('.guide-main')
  if (guideMain) {
    guideMain.addEventListener('scroll', handleScroll)
  }
})

onUnmounted(() => {
  const guideMain = document.querySelector('.guide-main')
  if (guideMain) {
    guideMain.removeEventListener('scroll', handleScroll)
  }
  
  if (scrollTimeout) {
    window.clearTimeout(scrollTimeout)
  }
})
</script>

<style scoped>
.guide-view {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background-color: #f8f9fa;
}

.guide-header {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 32px 24px;
  text-align: center;
}

.guide-header h1 {
  margin: 0 0 12px 0;
  font-size: 28px;
  font-weight: 600;
}

.guide-header p {
  margin: 0;
  font-size: 16px;
  opacity: 0.9;
}

.guide-content {
  flex: 1;
  display: flex;
  overflow: hidden;
}

.guide-sidebar {
  width: 280px;
  background: white;
  border-right: 1px solid #e0e0e0;
  overflow-y: auto;
}

.sidebar-nav {
  padding: 20px 0;
}

.nav-item {
  padding: 12px 24px;
  cursor: pointer;
  transition: all 0.2s ease;
  border-left: 3px solid transparent;
  font-size: 14px;
  color: #666;
}

.nav-item:hover {
  background-color: #f8f9fa;
  color: #333;
}

.nav-item.active {
  background-color: #f0f7ff;
  border-left-color: #3b82f6;
  color: #3b82f6;
  font-weight: 500;
}

.guide-main {
  flex: 1;
  overflow-y: auto;
  padding: 32px;
}

.section {
  margin-bottom: 48px;
  background: white;
  border-radius: 8px;
  padding: 32px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.05);
}

.section h2 {
  margin: 0 0 24px 0;
  font-size: 24px;
  font-weight: 600;
  color: #333;
  border-bottom: 2px solid #e0e0e0;
  padding-bottom: 12px;
}

.content-block {
  margin-bottom: 32px;
}

.content-block:last-child {
  margin-bottom: 0;
}

.content-block h3 {
  margin: 0 0 16px 0;
  font-size: 18px;
  font-weight: 600;
  color: #333;
}

.content-block h4 {
  margin: 20px 0 12px 0;
  font-size: 16px;
  font-weight: 600;
  color: #444;
}

.content-block p {
  line-height: 1.7;
  color: #666;
  margin-bottom: 16px;
}

.content-block ul {
  margin: 0;
  padding-left: 20px;
}

.content-block li {
  margin-bottom: 8px;
  line-height: 1.6;
  color: #666;
}

.code-example {
  background: #f8f9fa;
  border: 1px solid #e9ecef;
  border-radius: 6px;
  margin: 16px 0;
  overflow: hidden;
}

.code-example pre {
  margin: 0;
  padding: 20px;
  font-family: 'Consolas', 'Monaco', 'Courier New', monospace;
  font-size: 13px;
  line-height: 1.5;
  overflow-x: auto;
}

.code-example code {
  color: #333;
}

.interface-definition {
  background: #f8f9fa;
  border: 1px solid #e9ecef;
  border-radius: 6px;
  margin: 16px 0;
  overflow: hidden;
}

.interface-definition pre {
  margin: 0;
  padding: 20px;
  font-family: 'Consolas', 'Monaco', 'Courier New', monospace;
  font-size: 13px;
  line-height: 1.5;
  overflow-x: auto;
}

.operation-types {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 12px;
  margin: 16px 0;
}

.operation-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px;
  background: #f8f9fa;
  border-radius: 6px;
  border: 1px solid #e9ecef;
}

.operation-item code {
  background: #e9ecef;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 12px;
  font-weight: 500;
}

.key-features {
  background: #f0f7ff;
  border: 1px solid #cce7ff;
  border-radius: 6px;
  padding: 20px;
  margin: 16px 0;
}

.key-features h4 {
  margin: 0 0 12px 0;
  color: #0056b3;
}

.key-features ul {
  margin: 0;
  padding-left: 20px;
}

.key-features li {
  margin-bottom: 8px;
  color: #0056b3;
}

.key-features strong {
  color: #0056b3;
}

.best-practice-item {
  margin-bottom: 24px;
}

.best-practice-item h4 {
  margin: 0 0 12px 0;
  font-size: 16px;
  font-weight: 600;
}

.best-practice-item h4:first-child {
  color: #10b981;
}

.best-practice-item h4:last-child {
  color: #ef4444;
}

.debugging-tips {
  background: #fff3cd;
  border: 1px solid #ffeaa7;
  border-radius: 6px;
  margin: 16px 0;
  overflow: hidden;
}

.debugging-tips pre {
  margin: 0;
  padding: 20px;
  font-family: 'Consolas', 'Monaco', 'Courier New', monospace;
  font-size: 13px;
  line-height: 1.5;
  overflow-x: auto;
}

.scenario-item {
  margin-bottom: 32px;
  padding: 20px;
  background: #f8f9fa;
  border-radius: 6px;
  border: 1px solid #e9ecef;
}

.scenario-item h3 {
  margin: 0 0 12px 0;
  font-size: 18px;
  font-weight: 600;
  color: #333;
}

.scenario-item p {
  margin: 0 0 16px 0;
  color: #666;
}

.problem-solution {
  margin-bottom: 24px;
}

.problem {
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 6px;
  padding: 16px;
  margin-bottom: 12px;
}

.problem h4 {
  margin: 0 0 8px 0;
  color: #dc2626;
}

.problem p {
  margin: 0;
  color: #7f1d1d;
}

.solution {
  background: #f0f9ff;
  border: 1px solid #bae6fd;
  border-radius: 6px;
  padding: 16px;
}

.solution h4 {
  margin: 0 0 8px 0;
  color: #0369a1;
}

.solution ul {
  margin: 0;
  padding-left: 20px;
}

.solution li {
  margin-bottom: 4px;
  color: #0c4a6e;
}

/* 코드 하이라이팅 */
.code-example pre :deep(.keyword) {
  color: #d73a49;
}

.code-example pre :deep(.string) {
  color: #032f62;
}

.code-example pre :deep(.comment) {
  color: #6a737d;
}

.code-example pre :deep(.function) {
  color: #6f42c1;
}

/* 스크롤바 스타일 */
.guide-sidebar::-webkit-scrollbar,
.guide-main::-webkit-scrollbar {
  width: 6px;
}

.guide-sidebar::-webkit-scrollbar-track,
.guide-main::-webkit-scrollbar-track {
  background: #f1f1f1;
}

.guide-sidebar::-webkit-scrollbar-thumb,
.guide-main::-webkit-scrollbar-thumb {
  background: #c1c1c1;
  border-radius: 3px;
}

.guide-sidebar::-webkit-scrollbar-thumb:hover,
.guide-main::-webkit-scrollbar-thumb:hover {
  background: #a8a8a8;
}
</style>
