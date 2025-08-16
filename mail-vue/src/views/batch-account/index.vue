<template>
  <div class="batch-account-container">
    <div class="header">
      <h2>{{ $t('batchAddAccount') }}</h2>
      <div class="actions">
        <el-button type="primary" @click="showImportDialog = true">
          <Icon icon="material-symbols:upload" width="16" height="16" />
          {{ $t('importExcel') }}
        </el-button>
        <el-button @click="exportTemplate">
          <Icon icon="material-symbols:download" width="16" height="16" />
          {{ $t('downloadTemplate') }}
        </el-button>
        <el-button @click="exportAccounts" :disabled="accounts.length === 0">
          <Icon icon="material-symbols:table-view" width="16" height="16" />
          {{ $t('exportExcel') }}
        </el-button>
      </div>
    </div>

    <!-- 自动生成区域 -->
    <el-card class="auto-generate-card">
      <template #header>
        <span>{{ $t('autoGenerate') }}</span>
      </template>
      <div class="auto-generate-form">
        <el-input-number
          v-model="generateCount"
          :min="1"
          :max="100"
          :placeholder="$t('enterCount')"
          style="width: 200px; margin-right: 10px;"
        />
        <el-select v-model="selectedSuffix" style="width: 150px; margin-right: 10px;">
          <el-option
            v-for="suffix in domainList"
            :key="suffix"
            :label="suffix"
            :value="suffix"
          />
        </el-select>
        <el-button type="primary" @click="generateAccounts" :disabled="!generateCount || generateCount <= 0">
          {{ $t('generateAccounts') }}
        </el-button>
      </div>
    </el-card>

    <!-- 手动添加区域 -->
    <el-card class="manual-add-card">
      <template #header>
        <span>{{ $t('manualAdd') }}</span>
      </template>
      <div class="manual-add-form">
        <el-input
          v-model="newEmail"
          :placeholder="$t('enterEmail')"
          @keyup.enter="addSingleAccount"
          style="width: 300px; margin-right: 10px;"
        />
        <el-select v-model="selectedSuffix" style="width: 150px; margin-right: 10px;">
          <el-option
            v-for="suffix in domainList"
            :key="suffix"
            :label="suffix"
            :value="suffix"
          />
        </el-select>
        <el-button type="primary" @click="addSingleAccount">
          {{ $t('add') }}
        </el-button>
      </div>
    </el-card>

    <!-- 批量添加列表 -->
    <el-card class="accounts-list-card">
      <template #header>
        <div class="list-header">
          <span>{{ $t('accountsList') }} ({{ accounts.length }})</span>
          <div>
            <el-button 
              type="success" 
              @click="batchSubmit" 
              :loading="batchLoading"
              :disabled="accounts.length === 0"
            >
              {{ $t('batchSubmit') }}
            </el-button>
            <el-button @click="clearAll" :disabled="accounts.length === 0">
              {{ $t('clearAll') }}
            </el-button>
          </div>
        </div>
      </template>
      
      <div class="accounts-table">
        <el-table :data="accounts" style="width: 100%" max-height="400">
          <el-table-column prop="email" :label="$t('email')" />
          <el-table-column prop="status" :label="$t('status')" width="120">
            <template #default="{ row }">
              <el-tag 
                :type="row.status === 'pending' ? 'info' : row.status === 'success' ? 'success' : 'danger'"
              >
                {{ $t(row.status) }}
              </el-tag>
            </template>
          </el-table-column>
          <el-table-column prop="message" :label="$t('message')" />
          <el-table-column :label="$t('actions')" width="100">
            <template #default="{ row, $index }">
              <el-button 
                type="danger" 
                size="small" 
                @click="removeAccount($index)"
                :disabled="row.status === 'processing'"
              >
                {{ $t('delete') }}
              </el-button>
            </template>
          </el-table-column>
        </el-table>
      </div>
    </el-card>

    <!-- Excel导入对话框 -->
    <el-dialog v-model="showImportDialog" :title="$t('importExcel')" width="500px">
      <el-upload
        ref="uploadRef"
        :auto-upload="false"
        :on-change="handleFileChange"
        :show-file-list="false"
        accept=".xlsx,.xls"
        drag
      >
        <Icon icon="material-symbols:upload" width="40" height="40" color="#409EFF" />
        <div class="el-upload__text">
          {{ $t('dropFileHere') }} <em>{{ $t('clickToUpload') }}</em>
        </div>
        <template #tip>
          <div class="el-upload__tip">
            {{ $t('onlyExcelFiles') }}
          </div>
        </template>
      </el-upload>
      
      <template #footer>
        <div class="dialog-footer">
          <el-button @click="showImportDialog = false">{{ $t('cancel') }}</el-button>
          <el-button type="primary" @click="importExcel" :loading="importLoading">
            {{ $t('import') }}
          </el-button>
        </div>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, reactive } from 'vue'
import { Icon } from '@iconify/vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import { useI18n } from 'vue-i18n'
import { useSettingStore } from '@/store/setting.js'
import { accountAdd } from '@/request/account.js'
import { isEmail } from '@/utils/verify-utils.js'
import * as XLSX from 'xlsx'

const { t } = useI18n()
const settingStore = useSettingStore()

// 响应式数据
const accounts = reactive([])
const newEmail = ref('')
const generateCount = ref(10) // 默认生成10个账号
const selectedSuffix = ref(settingStore.domainList[0])
const domainList = settingStore.domainList
const showImportDialog = ref(false)
const importLoading = ref(false)
const batchLoading = ref(false)
const uploadRef = ref()
let selectedFile = null

// 生成随机邮箱前缀的函数（参考现有代码）
function generateRandomEmail() {
  const chars = 'abcdefghijklmnpqrstuvwxyz0123456789';
  let result = '';
  for (let i = 0; i < 8; i++) {
    result += chars.charAt(Math.floor(Math.random() * chars.length));
  }
  return result;
}

// 自动生成指定数量的账号
const generateAccounts = () => {
  if (!generateCount.value || generateCount.value <= 0) {
    ElMessage.error(t('invalidCount'))
    return
  }
  
  if (generateCount.value > 100) {
    ElMessage.error(t('countTooLarge'))
    return
  }
  
  const newAccounts = []
  const existingEmails = new Set(accounts.map(acc => acc.email))
  
  let attempts = 0
  const maxAttempts = generateCount.value * 10 // 防止无限循环
  
  while (newAccounts.length < generateCount.value && attempts < maxAttempts) {
    const emailPrefix = generateRandomEmail()
    const fullEmail = emailPrefix + selectedSuffix.value
    
    // 检查是否已存在
    if (!existingEmails.has(fullEmail) && !newAccounts.some(acc => acc.email === fullEmail)) {
      newAccounts.push({
        email: fullEmail,
        status: 'pending',
        message: ''
      })
      existingEmails.add(fullEmail)
    }
    attempts++
  }
  
  if (newAccounts.length < generateCount.value) {
    ElMessage.warning(t('generatePartialSuccess', { generated: newAccounts.length, requested: generateCount.value }))
  } else {
    ElMessage.success(t('generateSuccess', { count: newAccounts.length }))
  }
  
  accounts.push(...newAccounts)
}

// 添加单个账号到列表
const addSingleAccount = () => {
  if (!newEmail.value.trim()) {
    ElMessage.error(t('emptyEmailMsg'))
    return
  }
  
  const fullEmail = newEmail.value.trim() + selectedSuffix.value
  
  if (!isEmail(fullEmail)) {
    ElMessage.error(t('notEmailMsg'))
    return
  }
  
  // 检查是否已存在
  if (accounts.some(acc => acc.email === fullEmail)) {
    ElMessage.error(t('emailAlreadyExists'))
    return
  }
  
  accounts.push({
    email: fullEmail,
    status: 'pending',
    message: ''
  })
  
  newEmail.value = ''
  ElMessage.success(t('addedToList'))
}

// 移除账号
const removeAccount = (index) => {
  accounts.splice(index, 1)
}

// 清空所有
const clearAll = async () => {
  try {
    await ElMessageBox.confirm(t('confirmClearAll'), t('warning'), {
      confirmButtonText: t('confirm'),
      cancelButtonText: t('cancel'),
      type: 'warning'
    })
    accounts.splice(0)
  } catch {
    // 用户取消
  }
}

// 处理文件选择
const handleFileChange = (file) => {
  selectedFile = file.raw
}

// 导入Excel
const importExcel = async () => {
  if (!selectedFile) {
    ElMessage.error(t('pleaseSelectFile'))
    return
  }
  
  importLoading.value = true
  
  try {
    const data = await readExcelFile(selectedFile)
    const newAccounts = []
    
    data.forEach((row, index) => {
      if (index === 0) return // 跳过标题行
      
      const email = row[0]?.toString().trim()
      if (email && isEmail(email)) {
        // 检查是否已存在
        if (!accounts.some(acc => acc.email === email) && !newAccounts.some(acc => acc.email === email)) {
          newAccounts.push({
            email,
            status: 'pending',
            message: ''
          })
        }
      }
    })
    
    accounts.push(...newAccounts)
    showImportDialog.value = false
    selectedFile = null
    
    ElMessage.success(t('importSuccess', { count: newAccounts.length }))
  } catch (error) {
    ElMessage.error(t('importFailed'))
    console.error('Import error:', error)
  } finally {
    importLoading.value = false
  }
}

// 读取Excel文件
const readExcelFile = (file) => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.onload = (e) => {
      try {
        const data = new Uint8Array(e.target.result)
        const workbook = XLSX.read(data, { type: 'array' })
        const sheetName = workbook.SheetNames[0]
        const worksheet = workbook.Sheets[sheetName]
        const jsonData = XLSX.utils.sheet_to_json(worksheet, { header: 1 })
        resolve(jsonData)
      } catch (error) {
        reject(error)
      }
    }
    reader.onerror = reject
    reader.readAsArrayBuffer(file)
  })
}

// 批量提交
const batchSubmit = async () => {
  if (accounts.length === 0) {
    ElMessage.error(t('noAccountsToSubmit'))
    return
  }
  
  batchLoading.value = true
  
  // 重置所有状态
  accounts.forEach(acc => {
    acc.status = 'processing'
    acc.message = ''
  })
  
  let successCount = 0
  let failCount = 0
  
  // 逐个添加账号
  for (const account of accounts) {
    try {
      await accountAdd(account.email, '')
      account.status = 'success'
      account.message = t('addSuccess')
      successCount++
    } catch (error) {
      account.status = 'failed'
      account.message = error.message || t('addFailed')
      failCount++
    }
    
    // 添加小延迟避免请求过快
    await new Promise(resolve => setTimeout(resolve, 100))
  }
  
  batchLoading.value = false
  
  ElMessage.success(t('batchSubmitComplete', { success: successCount, fail: failCount }))
}

// 下载模板
const exportTemplate = () => {
  const templateData = [
    [t('email')],
    ['example1@domain.com'],
    ['example2@domain.com']
  ]
  
  const ws = XLSX.utils.aoa_to_sheet(templateData)
  const wb = XLSX.utils.book_new()
  XLSX.utils.book_append_sheet(wb, ws, 'Template')
  XLSX.writeFile(wb, 'account_template.xlsx')
}

// 导出账号列表
const exportAccounts = () => {
  if (accounts.length === 0) {
    ElMessage.error(t('noAccountsToExport'))
    return
  }
  
  const exportData = [
    [t('email'), t('username'), t('status'), t('message')],
    ...accounts.map(acc => {
      // 从完整邮箱地址中提取用户名部分（去掉@及后面的部分）
      const username = acc.email.split('@')[0]
      // 添加两位随机数字
      const randomNum = Math.floor(Math.random() * 100).toString().padStart(2, '0')
      const usernameWithRandom = `${username}.${randomNum}`
      return [acc.email, usernameWithRandom, t(acc.status), acc.message]
    })
  ]
  
  const ws = XLSX.utils.aoa_to_sheet(exportData)
  const wb = XLSX.utils.book_new()
  XLSX.utils.book_append_sheet(wb, ws, 'Accounts')
  
  const now = new Date()
  const timestamp = now.toISOString().slice(0, 19).replace(/:/g, '-')
  XLSX.writeFile(wb, `accounts_${timestamp}.xlsx`)
  
  ElMessage.success(t('exportSuccess'))
}
</script>

<style scoped lang="scss">
.batch-account-container {
  padding: 20px;
  
  .header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 20px;
    
    h2 {
      margin: 0;
      color: #303133;
    }
    
    .actions {
      display: flex;
      gap: 10px;
    }
  }
  
  .auto-generate-card {
    margin-bottom: 20px;
    
    .auto-generate-form {
      display: flex;
      align-items: center;
      flex-wrap: wrap;
      gap: 10px;
    }
  }
  
  .manual-add-card {
    margin-bottom: 20px;
    
    .manual-add-form {
      display: flex;
      align-items: center;
      flex-wrap: wrap;
      gap: 10px;
    }
  }
  
  .accounts-list-card {
    .list-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      
      > div {
        display: flex;
        gap: 10px;
      }
    }
    
    .accounts-table {
      margin-top: 10px;
    }
  }
  
  .dialog-footer {
    text-align: right;
  }
}

:deep(.el-upload-dragger) {
  width: 100%;
}
</style>