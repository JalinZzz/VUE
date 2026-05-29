<template>
  <div class="app-root">
    <!-- 顶部导航 -->
    <header class="global-header">
      <div class="logo"><span class="logo-icon">🎭</span>WebTest</div>
      <nav class="global-nav">
        <span class="nav-item">📊 仪表盘</span>
        <span class="nav-item">📁 项目</span>
        <span class="nav-item active">🧪 用例管理 <span class="nav-badge">{{ totalCount }}</span></span>
        <span class="nav-item">📋 测试计划</span>
        <span class="nav-item">📈 报告</span>
      </nav>
      <div class="header-filters">
        <div class="filter-group">
          <label>📁 项目</label>
          <select v-model="selectedProject" @change="onProjectChange">
            <option v-for="p in projectList" :key="p.id" :value="p.id">{{ p.name }}</option>
          </select>
        </div>
        <div class="filter-group">
          <label>📍 地区</label>
          <select v-model="selectedRegion" @change="onRegionChange">
            <option v-for="r in availableRegions" :key="r" :value="r">📍 {{ r }}</option>
          </select>
        </div>
      </div>
      <div class="header-right"><span class="env-dot"></span><span>Chrome 130</span><span class="avatar">A</span></div>
    </header>

    <!-- 主布局 -->
    <div class="main-layout">
      <!-- 左侧用例列表 -->
      <aside class="case-sidebar">
        <div class="case-sidebar-header">
          <div class="section-title">📋 测试用例 <span class="count-badge">{{ filteredCases.length }}</span></div>
          <div class="search-box"><span class="search-icon">🔍</span><input type="text" v-model="searchQuery" placeholder="搜索用例名称..."></div>
          <button class="btn-new" @click="createCase">＋ 新建测试用例</button>
        </div>
        <div class="case-list-wrap">
          <div v-if="filteredCases.length===0" class="empty-hint">暂无测试用例</div>
          <div v-for="tc in filteredCases" :key="tc.id" class="case-item" :class="{active:tc.id===currentCaseId}" @click="selectCase(tc.id)">
            <span class="c-icon">{{ getEmoji(tc.name) }}</span>
            <div class="c-info">
              <div class="c-name">{{ tc.name }}</div>
              <div class="c-meta"><span>{{ tc.region }}</span><span>·</span><span>{{ tc.stepsCount }}步</span><span>·</span><span>{{ tc.lastRun }}</span></div>
            </div>
            <span class="c-status" :class="tc.status"></span>
            <span class="c-actions" @click.stop>
              <button class="c-act-btn edit" @click="openEditModal(tc.id)">✏️</button>
              <button class="c-act-btn copy" @click="copyCase(tc.id)">📋</button>
              <button class="c-act-btn del" @click="deleteCase(tc.id)">🗑️</button>
            </span>
          </div>
        </div>
      </aside>

      <!-- 右侧编辑区 -->
      <section class="editor-area">
        <div class="top-bar">
          <div class="breadcrumb">🏠 首页 / <span>用例管理</span> / <span class="cur">{{ currentCaseName }}</span></div>
          <div class="info-tags"><span class="info-tag proj">📁 {{ currentProjectName }}</span><span class="info-tag reg">📍 {{ selectedRegion }}</span></div>
        </div>
        <div class="editor-toolbar">
          <input type="text" v-model="currentCaseName" @change="updateCaseTitle" placeholder="输入测试用例名称...">
          <span class="chip-sm"><span class="chrome-dot"></span> Chrome</span>
          <div class="toggle-grp">
            <button :class="{active:!isHeadless}" @click="isHeadless=false">🖥️ 有头</button>
            <button :class="{active:isHeadless}" @click="isHeadless=true">📦 无头</button>
          </div>
          <button class="btn-run" :class="{running:isRunning}" :disabled="isRunning" @click="runTest">
            {{ isRunning ? '⟳ 执行中...' : '▶ 运行测试' }}
          </button>
        </div>

        <div class="step-controls">
          <button class="btn-add-step" @click="addTestElement">➕ 添加测试元素</button>
          <button class="btn-flow" @click="addFlowStep('if')">🔀 IF 判断</button>
          <button class="btn-flow" @click="addFlowStep('for')">🔁 FOR 循环</button>
          <div class="combo-wrap">
            <button class="btn-combo" @click="showComboDropdown=!showComboDropdown">📦 元素组合 <span class="badge">模块</span></button>
            <div class="combo-drop" v-if="showComboDropdown" @click.stop>
              <div class="combo-opt" v-for="c in comboOptions" :key="c.type" @click="openComboConfig(c.type)">
                <span class="co-icon">{{ c.icon }}</span><div class="co-info"><div class="co-name">{{ c.name }}</div><div class="co-desc">{{ c.desc }}</div></div><span class="co-num">{{ c.steps }}步</span>
              </div>
            </div>
          </div>
          <span class="step-count">共 <strong>{{ flattenedStepsCount }}</strong> 个步骤</span>
        </div>

        <!-- 步骤列表 -->
        <div class="step-list">
          <div v-if="currentSteps.length===0" class="empty-state"><span class="empty-emoji">🧪</span><span class="empty-title">暂无测试步骤</span></div>
          <template v-for="(step, idx) in currentSteps" :key="step.id">
            <div class="step-card" :class="[step.expanded?'expanded':'', step._executing?'executing':'', step.result==='pass'?'passed':step.result==='fail'?'failed':'', isFlowStep(step)?'flow-card':'']">
              <div class="step-hdr" @click="step.expanded=!step.expanded">
                <span class="step-idx">{{ idx+1 }}</span>
                <span class="step-summary">
                  <span class="op-tag" :class="'tag-'+step.opType">{{ opLabel(step.opType) }}</span>
                  <span class="summary-page" v-if="step.page">{{ step.page }}</span>
                  <span class="sel-snip" v-if="step.selector && !isFlowStep(step)">{{ step.selector }}</span>
                  <span v-if="isFlowStep(step) && step.opType==='if'">条件: {{ step.condition || '未设置' }}</span>
                  <span v-if="isFlowStep(step) && step.opType==='for'">循环变量: {{ step.loopVar || '未设置' }}</span>
                </span>
                <span class="step-hdr-actions" @click.stop>
                  <button @click="moveStep(idx,-1)" :disabled="idx===0">▲</button>
                  <button @click="moveStep(idx,1)" :disabled="idx===currentSteps.length-1">▼</button>
                  <button class="del" @click="deleteStep(idx)">✕</button>
                </span>
                <span class="chevron">▼</span>
              </div>
              <div class="step-body">
                <!-- 普通步骤表单 -->
                <template v-if="!isFlowStep(step)">
                  <div class="form-row cols3">
                    <div class="f-group"><label>操作类型</label><select :value="step.opType" @change="updateStepField(step,'opType',$event.target.value)"><option v-for="o in opTypes" :key="o.value" :value="o.value">{{ o.label }}</option></select></div>
                    <div class="f-group"><label>页面URL</label><input type="text" :value="step.page" @change="updateStepField(step,'page',$event.target.value)" class="mono"></div>
                    <div class="f-group"><label>选择器类型</label><select :value="step.selectorType" @change="updateStepField(step,'selectorType',$event.target.value)"><option v-for="st in selectorTypes" :key="st" :value="st">{{ st.toUpperCase() }}</option></select></div>
                  </div>
                  <div class="form-row cols2">
                    <div class="f-group">
                      <label>元素选择器</label>
                      <div class="input-test">
                        <input type="text" :value="step.selector" @change="updateStepField(step,'selector',$event.target.value)" @input="onSelectorInput(step, $event)" @focus="showSelectorDropdown(step)" @blur="hideSelectorDropdown(step)" class="mono" autocomplete="off">
                        <button class="btn-test" @click="testSelector(step,$event)">🔍检测</button>
                        <div class="selector-search-dropdown" :class="{show: step._showDropdown}" @mousedown.prevent>
                          <div v-for="opt in filteredSelectorOptions(step)" :key="opt" class="selector-search-item" @mousedown="selectSelectorOption(step, opt)">{{ opt }}</div>
                          <div v-if="filteredSelectorOptions(step).length===0" class="selector-search-item" style="color:#9ca3af;">无匹配建议</div>
                        </div>
                      </div>
                    </div>
                    <div class="f-group" v-if="step.opType==='input'"><label>输入值</label><input type="text" :value="step.inputValue" @change="updateStepField(step,'inputValue',$event.target.value)"></div>
                    <div class="f-group" v-if="step.opType==='wait'"><label>等待(ms)</label><input type="number" :value="step.waitTime" @change="updateStepField(step,'waitTime',parseInt($event.target.value))" min="100"></div>
                    <template v-if="step.opType==='assert'">
                      <div class="f-group"><label>断言类型</label><select :value="step.assertType" @change="updateStepField(step,'assertType',$event.target.value)"><option v-for="at in assertTypes" :key="at" :value="at">{{ at }}</option></select></div>
                      <div class="f-group"><label>期望值</label><input type="text" :value="step.expectedValue" @change="updateStepField(step,'expectedValue',$event.target.value)"></div>
                    </template>
                  </div>
                  <div class="form-row cols2"><div class="f-group"><label>步骤描述</label><input type="text" :value="step.description" @change="updateStepField(step,'description',$event.target.value)"></div></div>
                  <div v-if="step.result" class="step-result-mini" :class="step.result==='pass'?'ok':'err'">{{ step.result==='pass'?'✅ 通过':'❌ 失败' }} — {{ step.resultMsg }}</div>
                </template>
                <!-- 控制流步骤表单 -->
                <template v-if="isFlowStep(step)">
                  <div class="form-row cols2">
                    <div class="f-group" v-if="step.opType==='if'"><label>条件表达式</label><input type="text" :value="step.condition" @change="updateStepField(step,'condition',$event.target.value)" placeholder="page.url().includes('/login')"></div>
                    <div class="f-group" v-if="step.opType==='for'"><label>循环变量</label><input type="text" :value="step.loopVar" @change="updateStepField(step,'loopVar',$event.target.value)" placeholder="item"></div>
                    <div class="f-group" v-if="step.opType==='for'"><label>迭代对象</label><input type="text" :value="step.iterateOver" @change="updateStepField(step,'iterateOver',$event.target.value)" placeholder="items"></div>
                  </div>
                  <div class="flow-children">
                    <div class="children-header">
                      <span>子步骤 ({{ step.children ? step.children.length : 0 }})</span>
                      <button class="btn-add-step small" @click="addChildStep(step)">＋ 添加子步骤</button>
                    </div>
                    <div v-if="step.children && step.children.length">
                      <div v-for="(child, cIdx) in step.children" :key="child.id" class="step-card child-card">
                        <div class="step-hdr" @click="child.expanded=!child.expanded">
                          <span class="step-idx">{{ cIdx+1 }}</span>
                          <span class="step-summary">
                            <span class="op-tag" :class="'tag-'+child.opType">{{ opLabel(child.opType) }}</span>
                            <span class="summary-page" v-if="child.page">{{ child.page }}</span>
                            <span class="sel-snip" v-if="child.selector">{{ child.selector }}</span>
                          </span>
                          <span class="step-hdr-actions" @click.stop><button class="del" @click="deleteChildStep(step, cIdx)">✕</button></span>
                          <span class="chevron">▼</span>
                        </div>
                        <div class="step-body" v-if="child.expanded">
                          <div class="form-row cols3">
                            <div class="f-group"><label>操作类型</label><select :value="child.opType" @change="updateChildField(step, cIdx, 'opType', $event.target.value)"><option v-for="o in opTypes" :key="o.value" :value="o.value">{{ o.label }}</option></select></div>
                            <div class="f-group"><label>页面URL</label><input type="text" :value="child.page" @change="updateChildField(step, cIdx, 'page', $event.target.value)" class="mono"></div>
                            <div class="f-group"><label>选择器类型</label><select :value="child.selectorType" @change="updateChildField(step, cIdx, 'selectorType', $event.target.value)"><option v-for="st in selectorTypes" :key="st" :value="st">{{ st.toUpperCase() }}</option></select></div>
                          </div>
                          <div class="form-row cols2"><div class="f-group"><label>元素选择器</label><input type="text" :value="child.selector" @change="updateChildField(step, cIdx, 'selector', $event.target.value)" class="mono"></div></div>
                        </div>
                      </div>
                    </div>
                    <div v-else class="empty-children">暂无子步骤</div>
                  </div>
                </template>
              </div>
            </div>
          </template>
        </div>

        <!-- 结果面板 -->
        <div class="result-panel" :class="{collapsed:resultCollapsed}">
          <div class="result-header" @click="resultCollapsed=!resultCollapsed">
            <span class="r-title">📋 执行结果 <span class="r-stats"><span class="g">✅{{ runStats.pass }}</span><span class="b">❌{{ runStats.fail }}</span><span class="n" v-if="runStats.time">⏱{{ runStats.time }}</span></span></span>
            <button class="btn-rerun" @click.stop="reRunTest" :disabled="isRunning">🔄 重新运行</button>
            <span class="collapse-hint">{{ resultCollapsed ? '展开' : '收起' }}</span>
          </div>
          <div class="result-content" v-show="!resultCollapsed">
            <div class="step-results-list" v-if="hasAnyResult">
              <div v-for="(s, idx) in currentSteps" :key="'sr-'+s.id" class="step-result-item">
                <span class="sr-idx">{{ idx+1 }}</span>
                <span class="sr-tag" :class="'tag-'+s.opType">{{ opLabel(s.opType) }}</span>
                <span class="sr-desc">{{ s.description || '未命名' }}</span>
                <span class="sr-status" :class="s.result">{{ s.result==='pass' ? '✅' : s.result==='fail' ? '❌' : '⏳' }}</span>
                <span class="sr-time" v-if="s.resultMsg">{{ s.resultMsg }}</span>
              </div>
            </div>
            <div class="console-log" ref="consoleLog">
              <div v-if="runLogs.length===0" class="log-placeholder">等待执行测试用例...</div>
              <div v-for="(log,idx) in runLogs" :key="idx" class="log-line" :class="log.type"><span class="ts">[{{ log.time }}]</span>{{ log.msg }}</div>
            </div>
          </div>
        </div>
      </section>
    </div>

    <!-- Toast -->
    <div class="toast-container">
      <div v-for="(t,idx) in toasts" :key="idx" class="toast" :class="t.type">{{ t.msg }}</div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'TestCaseManager',
  data() {
    return {
      projectList: [
        { id:'proj001', name:'🛒 电商平台', regions:['华东','华南','华北','西南'] },
        { id:'proj002', name:'⚙️ 后台管理', regions:['华东','华北','华中'] },
        { id:'proj003', name:'📱 移动端H5', regions:['华南','西南','西北'] },
      ],
      selectedProject:'proj001',
      selectedRegion:'华东',
      allTestCases:[
        { id:'tc001',projectId:'proj001',region:'华东',name:'登录功能测试',status:'passed',stepsCount:3,lastRun:'2026-05-26 14:30' },
        { id:'tc002',projectId:'proj001',region:'华东',name:'搜索商品功能',status:'failed',stepsCount:4,lastRun:'2026-05-26 11:15' },
        { id:'tc003',projectId:'proj001',region:'华南',name:'购物车添加商品',status:'passed',stepsCount:5,lastRun:'2026-05-25 16:42' },
        { id:'tc004',projectId:'proj001',region:'华南',name:'订单提交验证',status:'idle',stepsCount:7,lastRun:'未运行' },
      ],
      currentCaseId:'tc001',
      searchQuery:'',
      currentSteps:[],
      stepIdCounter:4,
      isRunning:false,
      isHeadless:false,
      runLogs:[],
      runStats:{ pass:0, fail:0, time:'' },
      resultCollapsed:false,
      showComboDropdown:false,
      toasts:[],
      opTypes:[
        { value:'click', label:'🖱️ 点击' },{ value:'input', label:'⌨️ 输入' },
        { value:'wait', label:'⏳ 等待' },{ value:'assert', label:'✅ 断言' },
        { value:'navigate', label:'🌐 导航' },{ value:'hover', label:'👆 悬停' },
        { value:'select', label:'📋 选择' },
      ],
      selectorTypes:['css','xpath','text','id','role','url'],
      assertTypes:['visible','text','url','title'],
      comboOptions:[
        { type:'login',icon:'🔐',name:'登录模块',desc:'导航→输入用户名→输入密码→点击登录',steps:4 },
        { type:'search',icon:'🔍',name:'搜索模块',desc:'导航→输入关键词→点击搜索→等待结果',steps:4 },
        { type:'form',icon:'📝',name:'表单填充模块',desc:'填写多字段→选择下拉→点击提交',steps:4 },
        { type:'navigate',icon:'🌐',name:'页面导航模块',desc:'打开URL→等待加载→断言标题',steps:3 },
      ],
      selectorSuggestions:[
        '#username','#password','#login-btn','#search-input','.search-btn','.results-container',
        '#name','#email','#submit-btn','h1','.status-msg','.content-loaded','#main-btn','.result'
      ],
    };
  },
  computed:{
    availableRegions(){ const p=this.projectList.find(x=>x.id===this.selectedProject); return p?p.regions:[]; },
    currentProjectName(){ const p=this.projectList.find(x=>x.id===this.selectedProject); return p?p.name:''; },
    currentCaseName:{
      get(){ const tc=this.allTestCases.find(c=>c.id===this.currentCaseId); return tc?tc.name:'无用例'; },
      set(val){ const tc=this.allTestCases.find(c=>c.id===this.currentCaseId); if(tc&&val.trim()) tc.name=val.trim(); }
    },
    projectCases(){ return this.allTestCases.filter(tc=>tc.projectId===this.selectedProject&&tc.region===this.selectedRegion); },
    filteredCases(){ const q=this.searchQuery.toLowerCase().trim(); return q?this.projectCases.filter(tc=>tc.name.toLowerCase().includes(q)):this.projectCases; },
    totalCount(){ return this.projectCases.length; },
    flattenedStepsCount(){
      let count=0;
      const countSteps=(steps)=>{ steps.forEach(s=>{ count++; if(s.children) countSteps(s.children); }); };
      countSteps(this.currentSteps);
      return count;
    },
    hasAnyResult(){ return this.currentSteps.some(s=>s.result); },
  },
  watch:{
    selectedProject:{ immediate:true, handler(){ this.onProjectChange(); } },
    selectedRegion(){ this.onRegionChange(); },
    currentCaseId(val){ if(val) this.loadSteps(); }
  },
  methods:{
    getEmoji(n){ if(!n)return'📄'; if(n.includes('登录'))return'🔐'; if(n.includes('搜索'))return'🔍'; if(n.includes('注册'))return'📝'; if(n.includes('购物车'))return'🛒'; return'📄'; },
    opLabel(op){ const m={click:'🖱️点击',input:'⌨️输入',wait:'⏳等待',assert:'✅断言',navigate:'🌐导航',hover:'👆悬停',select:'📋选择',if:'🔀 IF',for:'🔁 FOR'}; return m[op]||op; },
    isFlowStep(step){ return step.opType==='if'||step.opType==='for'; },
    toast(msg,type='info'){ const t={msg,type};this.toasts.push(t);setTimeout(()=>{const i=this.toasts.indexOf(t);if(i>=0)this.toasts.splice(i,1);},2000); },
    onProjectChange(){
      const proj = this.projectList.find(p => p.id === this.selectedProject);
      if (proj && !proj.regions.includes(this.selectedRegion)) {
        this.selectedRegion = proj.regions[0] || '';
      }
      this.searchQuery = '';
      this.resetResults();
      if (!this.projectCases.some(c => c.id === this.currentCaseId)) {
        this.currentCaseId = this.projectCases.length > 0 ? this.projectCases[0].id : null;
      }
    },
    onRegionChange(){
      this.searchQuery = '';
      this.resetResults();
      if (!this.projectCases.some(c => c.id === this.currentCaseId)) {
        this.currentCaseId = this.projectCases.length > 0 ? this.projectCases[0].id : null;
      }
    },
    selectCase(id){ if(this.isRunning) return; this.currentCaseId=id; this.resetResults(); this.showComboDropdown=false; },
    loadSteps(){
      const defSteps=[
        { id:'s1',opType:'navigate',page:'https://example.com/login',selectorType:'url',selector:'https://example.com/login',description:'打开登录页面',expanded:false,result:null,inputValue:'',waitTime:3000,assertType:'visible',expectedValue:'',children:[] },
        { id:'s2',opType:'input',page:'https://example.com/login',selectorType:'css',selector:'#username',description:'输入用户名',expanded:false,result:null,inputValue:'testuser',waitTime:3000,assertType:'visible',expectedValue:'',children:[] },
        { id:'s3',opType:'click',page:'https://example.com/login',selectorType:'css',selector:'#login-btn',description:'点击登录',expanded:false,result:null,inputValue:'',waitTime:3000,assertType:'visible',expectedValue:'',children:[] },
      ];
      this.currentSteps=JSON.parse(JSON.stringify(defSteps));
      this.stepIdCounter=4;
    },
    createCase(){
      const newId='tc'+Date.now().toString(36);
      this.allTestCases.push({ id:newId,projectId:this.selectedProject,region:this.selectedRegion,name:'新建用例',status:'idle',stepsCount:0,lastRun:'未运行' });
      this.currentCaseId=newId;
      this.currentSteps=[];
      this.resetResults();
    },
    openEditModal(id){ /* 可集成弹窗逻辑 */ },
    copyCase(id){ /* 复制用例逻辑 */ },
    deleteCase(id){ /* 删除用例逻辑 */ },
    addTestElement(){
      const s={ id:'s'+this.stepIdCounter++,opType:'click',page:'',selectorType:'css',selector:'',description:'新步骤',expanded:true,result:null,inputValue:'',waitTime:3000,assertType:'visible',expectedValue:'',children:[] };
      this.currentSteps.push(s);
    },
    addFlowStep(type){
      const s={ id:'s'+this.stepIdCounter++,opType:type,page:'',selectorType:'css',selector:'',description:(type==='if'?'条件判断':'循环'),expanded:true,result:null,condition:'',loopVar:'item',iterateOver:'items',children:[] };
      this.currentSteps.push(s);
    },
    addChildStep(parent){
      const child={ id:'cs'+this.stepIdCounter++,opType:'click',page:'',selectorType:'css',selector:'',description:'子步骤',expanded:true,result:null,inputValue:'',waitTime:3000,assertType:'visible',expectedValue:'' };
      if(!parent.children) parent.children=[];
      parent.children.push(child);
    },
    deleteChildStep(parent,idx){ parent.children.splice(idx,1); },
    updateChildField(parent,idx,field,value){ parent.children[idx][field]=value; },
    updateStepField(step,field,value){ step[field]=value; },
    moveStep(idx,dir){ const n=idx+dir; if(n<0||n>=this.currentSteps.length)return; [this.currentSteps[idx],this.currentSteps[n]]=[this.currentSteps[n],this.currentSteps[idx]]; },
    deleteStep(idx){ if(confirm('删除此步骤？')){ this.currentSteps.splice(idx,1); } },
    onSelectorInput(step,e){ step._selectorInput=e.target.value; step._showDropdown=true; },
    showSelectorDropdown(step){ step._showDropdown=true; },
    hideSelectorDropdown(step){ setTimeout(()=>{ step._showDropdown=false; },200); },
    filteredSelectorOptions(step){
      const input=(step._selectorInput||step.selector||'').toLowerCase();
      return this.selectorSuggestions.filter(s=>s.toLowerCase().includes(input));
    },
    selectSelectorOption(step,opt){ step.selector=opt; step._selectorInput=opt; step._showDropdown=false; },
    testSelector(step,event){ /* 模拟检测，可对接Playwright */ },
    async runTest(){
      if(this.isRunning) return;
      this.isRunning=true; this.resultCollapsed=false;
      this.currentSteps.forEach(s=>{ s.result=null; s.resultMsg=null; s._executing=false; });
      this.runLogs=[]; this.runStats={pass:0,fail:0,time:''};
      let pc=0, fc=0;
      for(let i=0;i<this.currentSteps.length;i++){
        const s=this.currentSteps[i]; s._executing=true; s.expanded=true;
        await new Promise(r=>setTimeout(r,400+Math.random()*500));
        const ok=Math.random()<0.85;
        s.result=ok?'pass':'fail';
        s.resultMsg=ok?'0.3s':'元素未找到 (0.5s)';
        if(ok) pc++; else fc++;
        this.runStats={pass:pc,fail:fc,time:''};
        s._executing=false;
      }
      this.runStats.time='1.2s';
      this.isRunning=false;
    },
    reRunTest(){ this.runTest(); },
    resetResults(){ /* 重置结果状态 */ },
    updateCaseTitle(){ /* 更新标题逻辑 */ },
  },
  mounted(){
    this.loadSteps();
    document.addEventListener('click',(e)=>{
      if(this.showComboDropdown){
        const wrap=this.$el.querySelector('.combo-wrap');
        if(wrap&&!wrap.contains(e.target)) this.showComboDropdown=false;
      }
    });
  }
};
</script>

<style scoped>
/* 请将完整的CSS样式粘贴至此，或使用全局样式文件 */
/* 因篇幅限制，此处省略详细样式，实际使用时需包含上一版HTML中的完整CSS */
</style>