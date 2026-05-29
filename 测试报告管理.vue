<template>
  <div class="report-manager">
    <!-- 顶部导航栏 (平台统一样式) -->
    <div class="top-nav">
      <div class="nav-container">
        <div class="logo-area">
          <div class="logo-icon"><i class="fas fa-vial"></i></div>
          <div class="logo-text">AutoTest <span>UI自动化平台</span></div>
        </div>
        <div class="nav-links">
          <a href="#" class="nav-link">仪表盘</a>
          <a href="#" class="nav-link">测试任务</a>
          <a href="#" class="nav-link active">报告管理</a>
          <a href="#" class="nav-link">测试配置</a>
          <div class="user-avatar"><i class="fas fa-user-circle"></i></div>
        </div>
      </div>
    </div>

    <div class="main-content">
      <div class="page-header">
        <div class="breadcrumb">
          <a href="#"><i class="fas fa-home"></i> 首页</a>
          <i class="fas fa-chevron-right"></i>
          <span>UI自动化测试</span>
          <i class="fas fa-chevron-right"></i>
          <span style="font-weight:500;">报告管理</span>
        </div>
        <div class="title-section">
          <h1><i class="fas fa-chart-line"></i> 测试报告</h1>
          <p>完整报告列表 · 支持在线预览(Allure)、离线下载及详情分析</p>
        </div>
      </div>

      <div class="card">
        <!-- 筛选栏 -->
        <div class="filter-bar">
          <div class="filter-group">
            <label><i class="fas fa-tag"></i> 项目名称</label>
            <input type="text" v-model="filters.projectName" placeholder="测试条件 / 项目名" @keyup.enter="handleSearch">
          </div>
          <div class="filter-group">
            <label><i class="fas fa-map-marker-alt"></i> 地区</label>
            <select v-model="filters.region">
              <option value="all">全部地区</option>
              <option value="华东">华东</option>
              <option value="华南">华南</option>
              <option value="华北">华北</option>
              <option value="西南">西南</option>
              <option value="西北">西北</option>
            </select>
          </div>
          <div class="filter-actions">
            <button class="btn-primary" @click="handleSearch"><i class="fas fa-search"></i> 搜索报告</button>
            <button class="btn-secondary" @click="handleReset"><i class="fas fa-undo-alt"></i> 重置</button>
          </div>
        </div>

        <!-- 报告表格 -->
        <div class="table-container">
          <table class="report-table">
            <thead>
              <tr>
                <th>测试条件</th><th>状态</th><th>测试引擎</th><th>浏览器</th>
                <th>总用例数</th><th>通过数</th><th>失败数</th><th>通过率</th>
                <th>执行人</th><th>执行时间</th><th>执行时长</th><th>地区</th><th>操作</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="report in filteredReports" :key="report.id">
                <td><strong>{{ report.testCondition }}</strong></td>
                <td><span class="status-badge"><i class="fas fa-times-circle"></i> 失败</span></td>
                <td>{{ report.engine }}</td>
                <td>{{ report.browser }}</td>
                <td>{{ report.totalCases }}</td>
                <td>{{ report.passed }}</td>
                <td>{{ report.failed }}</td>
                <td>{{ calcPassRate(report.passed, report.totalCases) }}</td>
                <td>{{ report.executor }}</td>
                <td>{{ report.execTime }}</td>
                <td>{{ report.duration }}</td>
                <td>{{ report.region }}</td>
                <td class="action-buttons">
                  <button class="btn-icon-text btn-detail" @click="showReportDetail(report)"><i class="fas fa-eye"></i> 查看详情</button>
                  <button class="btn-icon-text btn-online" @click="openOnlineReport(report)"><i class="fas fa-globe"></i> 在线报告</button>
                  <button class="btn-icon-text btn-offline" @click="downloadOfflineReport(report)"><i class="fas fa-download"></i> 离线报告</button>
                  <button class="btn-icon-text btn-delete" @click="deleteReport(report)"><i class="fas fa-trash-alt"></i> 删除</button>
                </td>
              </tr>
              <tr v-if="filteredReports.length === 0">
                <td colspan="13" class="empty-row">暂无报告数据，请调整筛选条件</td>
              </tr>
            </tbody>
          </table>
        </div>
        <div class="footer-stats">
          <span><i class="fas fa-database"></i> 共 {{ filteredReports.length }} 条报告</span>
          <span><i class="fas fa-info-circle"></i> 在线报告将跳转至 Allure 报告服务</span>
        </div>
      </div>
    </div>

    <!-- 报告详情弹窗（主模态框） -->
    <div v-if="detailModalVisible" class="modal-overlay active" @click.self="closeDetailModal">
      <div class="modal-container">
        <div class="modal-header">
          <h3><i class="fas fa-chart-bar"></i> 测试报告详情</h3>
          <button class="modal-close" @click="closeDetailModal">&times;</button>
        </div>
        <div class="modal-body">
          <!-- 基础信息 -->
          <div class="info-grid">
            <div class="info-item"><span class="info-label">报告ID：</span><span class="info-value">{{ currentReport?.id }}</span></div>
            <div class="info-item"><span class="info-label">测试套件：</span><span class="info-value">{{ currentReport?.testCondition }}</span></div>
            <div class="info-item"><span class="info-label">执行状态：</span><span class="info-value">失败</span></div>
            <div class="info-item"><span class="info-label">执行者：</span><span class="info-value">{{ currentReport?.executor }}</span></div>
            <div class="info-item"><span class="info-label">测试引擎：</span><span class="info-value">{{ currentReport?.engine }}</span></div>
            <div class="info-item"><span class="info-label">浏览器：</span><span class="info-value">{{ currentReport?.browser }}</span></div>
            <div class="info-item"><span class="info-label">执行模式：</span><span class="info-value">{{ currentReport?.mode || '有头模式' }}</span></div>
            <div class="info-item"><span class="info-label">执行时长：</span><span class="info-value">{{ currentReport?.duration }}</span></div>
            <div class="info-item"><span class="info-label">开始时间：</span><span class="info-value">{{ currentReport?.startTime || currentReport?.execTime }}</span></div>
            <div class="info-item"><span class="info-label">结束时间：</span><span class="info-value">{{ currentReport?.endTime || '--' }}</span></div>
          </div>
          <!-- 统计卡片 -->
          <div class="stats-cards">
            <div class="stat-box"><div class="stat-number">{{ currentReport?.totalCases }}</div><div class="stat-label">总用例数</div></div>
            <div class="stat-box"><div class="stat-number" style="color:#10b981">{{ currentReport?.passed }}</div><div class="stat-label">通过数</div></div>
            <div class="stat-box"><div class="stat-number" style="color:#dc2626">{{ currentReport?.failed }}</div><div class="stat-label">失败数</div></div>
            <div class="stat-box"><div class="stat-number">{{ currentReport?.skipped || 0 }}</div><div class="stat-label">跳过数</div></div>
            <div class="stat-box"><div class="passrate-big">{{ calcPassRate(currentReport?.passed, currentReport?.totalCases) }}</div><div class="stat-label">通过率</div></div>
          </div>
          <!-- 用例列表 -->
          <div class="cases-section">
            <div class="cases-title"><i class="fas fa-list-ul"></i> 执行结果详情</div>
            <table class="cases-table">
              <thead><tr><th>用例名称</th><th>执行状态</th><th>操作</th></tr></thead>
              <tbody>
                <tr v-for="(caseItem, idx) in currentReportCases" :key="idx">
                  <td>{{ caseItem.name }}</td>
                  <td :class="caseItem.status === '失败' ? 'case-status-fail' : (caseItem.status === '成功' ? 'case-status-pass' : '')">{{ caseItem.status }}</td>
                  <td><button class="btn-sm" @click="showCaseDetail(caseItem)">查看详情</button></td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </div>

    <!-- 用例详情弹窗（子模态框，带Tab切换） -->
    <div v-if="caseDetailModalVisible" class="modal-overlay active" @click.self="closeCaseDetailModal">
      <div class="modal-container" style="max-width:750px">
        <div class="modal-header">
          <h3><i class="fas fa-file-alt"></i> 用例执行详情</h3>
          <button class="modal-close" @click="closeCaseDetailModal">&times;</button>
        </div>
        <div class="modal-body">
          <div class="tab-container">
            <div class="tab-headers">
              <button class="tab-btn" :class="{ active: activeTab === 'log' }" @click="activeTab = 'log'"><i class="fas fa-terminal"></i> 执行日志</button>
              <button class="tab-btn" :class="{ active: activeTab === 'error' }" @click="activeTab = 'error'"><i class="fas fa-exclamation-triangle"></i> 错误信息</button>
              <button class="tab-btn" :class="{ active: activeTab === 'screenshot' }" @click="activeTab = 'screenshot'"><i class="fas fa-image"></i> 失败截图</button>
            </div>
            <div class="tab-content">
              <div v-show="activeTab === 'log'" class="tab-pane"><div class="log-content">{{ currentCase?.log || '无日志数据' }}</div></div>
              <div v-show="activeTab === 'error'" class="tab-pane"><div class="error-content">{{ currentCase?.error || '无错误信息' }}</div></div>
              <div v-show="activeTab === 'screenshot'" class="tab-pane">
                <div class="screenshot-placeholder">
                  <img v-if="currentCase?.screenshot" :src="currentCase.screenshot" class="screenshot-img" alt="失败截图">
                  <span v-else>无截图数据</span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

// ---------- Mock 数据 ----------
const rawReports = [
  {
    id: 1, testCondition: "test", status: "失败", engine: "Playwright", browser: "Chrome",
    totalCases: 2, passed: 0, failed: 2, skipped: 0, executor: "jasminexh",
    execTime: "2026-05-03 19:16:28", duration: "34.6秒", region: "华东",
    startTime: "2026/5/3 19:16:28", endTime: "2026/5/3 19:17:02", mode: "有头模式",
    allureReportUrl: "/allure-reports/1/index.html"
  },
  {
    id: 2, testCondition: "登录流程", status: "失败", engine: "Playwright", browser: "Chrome",
    totalCases: 3, passed: 0, failed: 3, skipped: 0, executor: "zhangwei",
    execTime: "2026-05-02 10:22:10", duration: "52.3秒", region: "华南",
    startTime: "2026/5/2 10:22:10", endTime: "2026/5/2 10:23:02", mode: "有头模式",
    allureReportUrl: "/allure-reports/2/index.html"
  },
  {
    id: 3, testCondition: "酒店搜索", status: "失败", engine: "Playwright", browser: "Chrome",
    totalCases: 2, passed: 0, failed: 2, skipped: 0, executor: "lina",
    execTime: "2026-05-01 14:11:05", duration: "28.9秒", region: "华北",
    startTime: "2026/5/1 14:11:05", endTime: "2026/5/1 14:11:34", mode: "有头模式",
    allureReportUrl: "/allure-reports/3/index.html"
  },
  {
    id: 4, testCondition: "订单支付", status: "失败", engine: "Playwright", browser: "Firefox",
    totalCases: 4, passed: 1, failed: 2, skipped: 1, executor: "wangpeng",
    execTime: "2026-04-30 09:42:00", duration: "67.2秒", region: "华东",
    startTime: "2026/4/30 09:42:00", endTime: "2026/4/30 09:43:07", mode: "有头模式",
    allureReportUrl: "/allure-reports/4/index.html"
  }
]

// 用例明细映射（日志、错误、截图）
const caseDetailsMap = {
  1: [
    { name: "验证首页加载", status: "失败", log: "[INFO] 开始加载首页\n[WARN] 超时10秒未响应\n[ERROR] 定位元素 .banner 失败\n[STACK] at Page.waitForSelector", error: "TimeoutError: Waiting for selector '.banner' failed: timeout 30000ms exceeded", screenshot: "https://picsum.photos/id/20/400/300?grayscale" },
    { name: "点击登录按钮", status: "失败", log: "[INFO] 跳转登录页\n[ERROR] 未找到登录表单\n[DEBUG] 页面HTML结构异常", error: "ElementNotFound: 登录按钮不可见，可能被其他元素覆盖", screenshot: "https://picsum.photos/id/26/400/300" }
  ],
  2: [
    { name: "输入用户名密码", status: "失败", log: "[INFO] 输入admin\n[INFO] 输入密码\n[ERROR] 提交失败，接口返回401", error: "ValidationError: 用户不存在或密码错误", screenshot: "https://picsum.photos/id/13/400/300" },
    { name: "校验验证码", status: "失败", log: "[INFO] 请求验证码图片\n[ERROR] OCR识别超时，无法获取验证码", error: "Captcha识别超时，重试3次失败", screenshot: "https://picsum.photos/id/15/400/300" },
    { name: "点击登录", status: "失败", log: "[WARN] 按钮存在但点击无响应\n[ERROR] JS事件未绑定", error: "JavaScript错误：未定义的函数", screenshot: "https://picsum.photos/id/3/400/300" }
  ],
  3: [
    { name: "酒店搜索框输入", status: "失败", log: "[INFO] 输入'上海'\n[ERROR] 下拉框未出现，可能接口延迟", error: "Request failed 500: 后端服务异常", screenshot: "https://picsum.photos/id/104/400/300" },
    { name: "点击搜索按钮", status: "失败", log: "[INFO] 点击事件已触发\n[ERROR] 页面无跳转，控制台报错", error: "NetworkError: 无法连接到搜索服务器", screenshot: "https://picsum.photos/id/169/400/300" }
  ],
  4: [
    { name: "选择酒店房型", status: "成功", log: "[INFO] 房型选择成功，价格获取正常", error: "", screenshot: "" },
    { name: "填写入住人信息", status: "失败", log: "[ERROR] 字段校验失败\n[INFO] 手机号输入不合法", error: "手机号格式错误，应输入11位数字", screenshot: "https://picsum.photos/id/12/400/300" },
    { name: "提交订单", status: "失败", log: "[ERROR] 创建订单接口返回500\n[INFO] 请求参数校验失败", error: "Internal Server Error: 订单服务不可用", screenshot: "https://picsum.photos/id/29/400/300" },
    { name: "支付环节", status: "跳过", log: "[WARN] 前置失败，跳过支付", error: "", screenshot: "" }
  ]
}
// 补全默认用例集
rawReports.forEach(r => {
  if (!caseDetailsMap[r.id]) caseDetailsMap[r.id] = [{ name: "默认用例", status: "失败", log: "无日志", error: "无错误", screenshot: "" }]
})

// 响应式数据
const reportsData = ref([...rawReports])
const filters = ref({ projectName: '', region: 'all' })

// 筛选后的报告
const filteredReports = computed(() => {
  return reportsData.value.filter(r => {
    const matchProject = filters.value.projectName === '' || r.testCondition.toLowerCase().includes(filters.value.projectName.toLowerCase())
    const matchRegion = filters.value.region === 'all' || r.region === filters.value.region
    return matchProject && matchRegion
  })
})

// 弹窗相关
const detailModalVisible = ref(false)
const currentReport = ref(null)
const currentReportCases = ref([])
const caseDetailModalVisible = ref(false)
const currentCase = ref(null)
const activeTab = ref('log')

// 辅助函数
const calcPassRate = (passed, total) => {
  if (total === 0) return '0%'
  return `${Math.round((passed / total) * 100)}%`
}

// 搜索/重置
const handleSearch = () => { /* 计算属性自动响应 */ }
const handleReset = () => {
  filters.value.projectName = ''
  filters.value.region = 'all'
}

// 在线报告（跳转 Allure）
const openOnlineReport = (report) => {
  if (report.allureReportUrl) {
    window.open(report.allureReportUrl, '_blank')
  } else {
    alert('Allure报告地址未配置，演示环境将打开示例报告')
    window.open('https://demo.allure-report.io/', '_blank')
  }
}

// 离线报告模拟
const downloadOfflineReport = (report) => {
  alert(`📥 下载离线报告：测试条件“${report.testCondition}” 的离线报告包（HTML/PDF）已生成。\n实际平台将导出完整数据包。`)
}

// 删除报告
const deleteReport = (report) => {
  if (confirm(`确认删除报告“${report.testCondition}”吗？删除后不可恢复。`)) {
    reportsData.value = reportsData.value.filter(r => r.id !== report.id)
    // 若当前详情弹窗打开且删除的是同一报告，关闭弹窗
    if (currentReport.value?.id === report.id) closeDetailModal()
  }
}

// 展示报告详情（主模态框）
const showReportDetail = (report) => {
  currentReport.value = report
  currentReportCases.value = caseDetailsMap[report.id] || []
  detailModalVisible.value = true
}

const closeDetailModal = () => {
  detailModalVisible.value = false
  currentReport.value = null
  currentReportCases.value = []
}

// 展示用例详情（子模态框，带Tab切换）
const showCaseDetail = (caseItem) => {
  currentCase.value = caseItem
  activeTab.value = 'log'   // 默认激活日志Tab
  caseDetailModalVisible.value = true
}

const closeCaseDetailModal = () => {
  caseDetailModalVisible.value = false
  currentCase.value = null
}
</script>

<style scoped>
/* 样式复用之前的设计，稍作调整以适配组件作用域 */
.report-manager {
  font-family: 'Inter', system-ui, -apple-system, 'Segoe UI', sans-serif;
  background: #f1f5f9;
  min-height: 100vh;
}

/* 顶部导航 */
.top-nav {
  background: #ffffff;
  border-bottom: 1px solid #e2e8f0;
  position: sticky;
  top: 0;
  z-index: 20;
  box-shadow: 0 1px 3px rgba(0,0,0,0.02);
}
.nav-container {
  max-width: 1600px;
  margin: 0 auto;
  padding: 0 28px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 64px;
}
.logo-area {
  display: flex;
  align-items: center;
  gap: 12px;
}
.logo-icon {
  background: #3b82f6;
  width: 34px;
  height: 34px;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-size: 18px;
}
.logo-text {
  font-weight: 700;
  font-size: 18px;
  color: #0f172a;
}
.logo-text span {
  font-weight: 500;
  color: #475569;
  font-size: 15px;
}
.nav-links {
  display: flex;
  gap: 28px;
  align-items: center;
}
.nav-link {
  font-size: 14px;
  font-weight: 500;
  color: #334155;
  text-decoration: none;
  padding: 6px 0;
  border-bottom: 2px solid transparent;
}
.nav-link.active {
  color: #3b82f6;
  border-bottom-color: #3b82f6;
}
.user-avatar {
  width: 38px;
  height: 38px;
  background: #eef2ff;
  border-radius: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #3b82f6;
  cursor: default;
}

/* 主内容 */
.main-content {
  max-width: 1600px;
  margin: 0 auto;
  padding: 28px 28px 48px 28px;
}
.page-header {
  margin-bottom: 28px;
}
.breadcrumb {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
  color: #5b6e8c;
  margin-bottom: 16px;
}
.breadcrumb a {
  color: #3b82f6;
  text-decoration: none;
  font-weight: 500;
}
.title-section h1 {
  font-size: 28px;
  font-weight: 700;
  color: #0f172a;
  display: flex;
  align-items: center;
  gap: 12px;
}
.card {
  background: #ffffff;
  border-radius: 24px;
  border: 1px solid #eef2f8;
  overflow: hidden;
}
.filter-bar {
  padding: 20px 24px 16px 24px;
  border-bottom: 1px solid #eff3f8;
  display: flex;
  flex-wrap: wrap;
  align-items: flex-end;
  gap: 20px;
}
.filter-group {
  display: flex;
  flex-direction: column;
  gap: 6px;
  min-width: 180px;
}
.filter-group label {
  font-size: 12px;
  font-weight: 600;
  color: #5b6e8c;
}
.filter-group input, .filter-group select {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 14px;
  padding: 10px 14px;
  font-size: 13px;
  font-family: inherit;
  outline: none;
}
.filter-group input:focus, .filter-group select:focus {
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59,130,246,0.1);
}
.filter-actions {
  display: flex;
  gap: 12px;
  margin-left: auto;
}
.btn-primary, .btn-secondary {
  border: none;
  padding: 10px 20px;
  border-radius: 40px;
  font-weight: 600;
  font-size: 13px;
  display: inline-flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
}
.btn-primary {
  background: #3b82f6;
  color: white;
}
.btn-primary:hover { background: #2563eb; }
.btn-secondary {
  background: #ffffff;
  border: 1px solid #cbd5e1;
  color: #334155;
}
.btn-secondary:hover { background: #f8fafc; }

.table-container {
  overflow-x: auto;
}
.report-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 13px;
  min-width: 1300px;
}
.report-table th {
  text-align: left;
  padding: 16px 14px;
  background: #fbfdff;
  font-weight: 600;
  border-bottom: 1px solid #eef2f8;
}
.report-table td {
  padding: 14px 14px;
  border-bottom: 1px solid #f1f4f9;
  vertical-align: middle;
}
.report-table tr:hover td {
  background-color: #fafcff;
}
.status-badge {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  background: #fee9e6;
  color: #b91c1c;
  padding: 4px 12px;
  border-radius: 40px;
  font-weight: 600;
  font-size: 12px;
}
.action-buttons {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}
.btn-icon-text {
  background: transparent;
  border: none;
  font-size: 11.5px;
  font-weight: 500;
  padding: 5px 10px;
  border-radius: 30px;
  display: inline-flex;
  align-items: center;
  gap: 6px;
  cursor: pointer;
  color: #475569;
}
.btn-detail:hover { background: #eef2ff; color: #2563eb; }
.btn-online:hover { background: #e6f7ec; color: #10b981; }
.btn-offline:hover { background: #fff4e6; color: #d97706; }
.btn-delete:hover { background: #fee2e2; color: #dc2626; }
.empty-row {
  text-align: center;
  padding: 48px;
  color: #64748b;
}
.footer-stats {
  padding: 14px 24px;
  border-top: 1px solid #eff3f8;
  font-size: 12px;
  color: #5b6e8c;
  display: flex;
  justify-content: space-between;
}

/* 模态窗样式 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0,0,0,0.6);
  backdrop-filter: blur(3px);
  z-index: 1000;
  display: flex;
  align-items: center;
  justify-content: center;
}
.modal-container {
  background: #ffffff;
  width: 90%;
  max-width: 1100px;
  max-height: 85vh;
  border-radius: 28px;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}
.modal-header {
  padding: 20px 28px;
  border-bottom: 1px solid #eef2f8;
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.modal-close {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  color: #94a3b8;
}
.modal-body {
  padding: 24px 28px;
  overflow-y: auto;
}
.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 16px 24px;
  background: #f8fafc;
  padding: 18px 20px;
  border-radius: 20px;
  margin-bottom: 28px;
}
.info-item {
  display: flex;
  font-size: 13px;
}
.info-label {
  width: 90px;
  font-weight: 600;
  color: #475569;
}
.info-value {
  font-weight: 500;
}
.stats-cards {
  display: flex;
  flex-wrap: wrap;
  gap: 18px;
  margin-bottom: 24px;
  border: 1px solid #eef2f8;
  border-radius: 20px;
  padding: 20px;
}
.stat-box {
  flex: 1;
  text-align: center;
  background: #f9fafc;
  border-radius: 18px;
  padding: 12px 8px;
}
.stat-number {
  font-size: 28px;
  font-weight: 800;
}
.passrate-big {
  font-size: 24px;
  font-weight: 800;
  color: #dc2626;
}
.cases-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 13px;
  border: 1px solid #edf2f7;
  border-radius: 18px;
  overflow: hidden;
}
.cases-table th {
  background: #f8fafc;
  padding: 12px;
  text-align: left;
}
.cases-table td {
  padding: 12px;
  border-top: 1px solid #eff3f8;
}
.case-status-fail { color: #dc2626; font-weight: 600; }
.case-status-pass { color: #10b981; }
.btn-sm {
  background: #eef2ff;
  border: none;
  padding: 5px 12px;
  border-radius: 30px;
  font-size: 11px;
  font-weight: 500;
  cursor: pointer;
}
.tab-container {
  border: 1px solid #eef2f8;
  border-radius: 20px;
  overflow: hidden;
}
.tab-headers {
  display: flex;
  background: #f8fafc;
  border-bottom: 1px solid #eef2f8;
}
.tab-btn {
  flex: 1;
  background: transparent;
  border: none;
  padding: 12px 16px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  color: #5b6e8c;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}
.tab-btn.active {
  color: #3b82f6;
  background: white;
  border-bottom: 2px solid #3b82f6;
}
.tab-content {
  padding: 20px;
  background: white;
}
.log-content {
  background: #1e293b;
  color: #e2e8f0;
  padding: 16px;
  border-radius: 16px;
  font-family: monospace;
  font-size: 12px;
  white-space: pre-wrap;
}
.error-content {
  background: #fff0f0;
  border-left: 4px solid #dc2626;
  padding: 14px;
  border-radius: 12px;
}
.screenshot-placeholder {
  text-align: center;
}
.screenshot-img {
  max-width: 100%;
  border-radius: 12px;
  border: 1px solid #e2e8f0;
}
</style>