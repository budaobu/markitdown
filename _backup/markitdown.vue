<script setup lang="ts">
import MarkdownIt from 'markdown-it'

useHead({
  title: '万能格式转 Markdown',
  meta: [
    { name: 'description', content: '使用 MarkitDown 将 PDF, Word, PowerPoint, Excel 等文件转换为 Markdown 格式。' }
  ]
})

// --- 状态管理 ---
const file = ref<File | null>(null)
const isProcessing = ref(false)
const resultText = ref('')
const resultFilename = ref('')
const activeTab = ref(0) // 0: Source, 1: Preview
const md = new MarkdownIt()
const renderedHtml = computed(() => md.render(resultText.value))

// --- 文件处理 ---
const handleDrop = (event: DragEvent) => {
  event.preventDefault()
  if (event.dataTransfer?.files && event.dataTransfer.files[0]) {
    setFile(event.dataTransfer.files[0])
  }
}

const handleFileChange = (event: Event) => {
  const input = event.target as HTMLInputElement
  if (input.files && input.files[0]) {
    setFile(input.files[0])
  }
}

const setFile = (f: File) => {
  file.value = f
  // 重置结果
  resultText.value = ''
  resultFilename.value = ''
  activeTab.value = 0
}

const triggerFileInput = () => {
  const input = document.getElementById('file-upload') as HTMLInputElement
  if (input) input.click()
}

// --- 转换逻辑 ---
const startConversion = async () => {
  if (!file.value) return

  isProcessing.value = true
  resultText.value = ''
  
  const formData = new FormData()
  formData.append('file', file.value)

  try {
    const data = await $fetch('/api/markitdown', {
      method: 'POST',
      body: formData
    })

    if (data.success) {
      resultText.value = data.textContent
      // 将原文件名后缀改为 .md
      const nameParts = file.value.name.split('.')
      if (nameParts.length > 1) nameParts.pop()
      resultFilename.value = nameParts.join('.') + '.md'
      
      useToast().add({
        title: '转换成功',
        description: '文件已成功转换为 Markdown 格式',
        color: 'green'
      })
    }
  } catch (error: any) {
    console.error(error)
    useToast().add({
      title: '转换失败',
      description: error.data?.statusMessage || '服务器处理出错，请稍后重试',
      color: 'red'
    })
  } finally {
    isProcessing.value = false
  }
}

// --- 辅助功能 ---
const copyToClipboard = () => {
  if (!resultText.value) return
  navigator.clipboard.writeText(resultText.value)
  useToast().add({
    title: '已复制',
    description: 'Markdown 内容已复制到剪贴板',
    color: 'primary'
  })
}

const downloadFile = () => {
  if (!resultText.value) return
  const blob = new Blob([resultText.value], { type: 'text/markdown;charset=utf-8' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = resultFilename.value || 'converted.md'
  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
  URL.revokeObjectURL(url)
}

const formatSize = (bytes: number) => {
  if (bytes === 0) return '0 B'
  const k = 1024
  const sizes = ['B', 'KB', 'MB', 'GB']
  const i = Math.floor(Math.log(bytes) / Math.log(k))
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i]
}
</script>

<template>
  <div class="min-h-screen bg-gray-50 dark:bg-gray-950 py-12 px-4 sm:px-6">
    <!-- Header -->
    <div class="max-w-3xl mx-auto text-center mb-12">
      <div class="inline-flex items-center justify-center w-16 h-16 rounded-full bg-primary-100 dark:bg-primary-900/30 text-primary-600 dark:text-primary-400 mb-4">
        <UIcon name="i-heroicons-document-text" class="text-3xl" />
      </div>
      <h1 class="text-3xl font-bold text-gray-900 dark:text-white mb-3">万能格式转 Markdown</h1>
      <p class="text-gray-500 dark:text-gray-400">
        支持 PDF, Word (docx), PowerPoint (pptx), Excel (xlsx) 等多种格式。
        <br>基于 <a href="https://github.com/microsoft/markitdown" target="_blank" class="underline hover:text-primary-500">MarkItDown</a> 技术。
      </p>
    </div>

    <!-- Main Card -->
    <div class="max-w-7xl mx-auto">
      <UCard 
        :ui="{ 
          body: { base: 'p-0 sm:p-0' }, 
          header: { base: 'border-b border-gray-200 dark:border-gray-800' } 
        }"
      >
        <div class="grid grid-cols-1 lg:grid-cols-2 min-h-[600px] divide-y lg:divide-y-0 lg:divide-x divide-gray-200 dark:divide-gray-800">
          
          <!-- Left: Upload Area -->
          <div class="p-6 sm:p-8 flex flex-col gap-6 bg-white dark:bg-gray-900">
            <h2 class="text-lg font-semibold flex items-center gap-2 text-gray-900 dark:text-white">
              <UIcon name="i-heroicons-cloud-arrow-up" class="text-primary-500" />
              上传文件
            </h2>

            <div 
              class="flex-1 border-2 border-dashed border-gray-300 dark:border-gray-700 rounded-xl p-8 text-center hover:border-primary-500 dark:hover:border-primary-500 hover:bg-primary-50 dark:hover:bg-primary-900/10 transition-all cursor-pointer group flex flex-col items-center justify-center relative"
              @click="triggerFileInput"
              @dragover.prevent
              @drop="handleDrop"
            >
              <input 
                id="file-upload" 
                type="file" 
                class="hidden" 
                @change="handleFileChange"
              >
              
              <div v-if="!file" class="flex flex-col items-center">
                <UIcon name="i-heroicons-document-plus" class="text-6xl text-gray-300 dark:text-gray-600 group-hover:text-primary-500 mb-4 transition-colors" />
                <p class="text-lg font-medium text-gray-700 dark:text-gray-200 mb-2">点击或拖拽文件到此处</p>
                <p class="text-sm text-gray-500 dark:text-gray-400">支持 PDF, DOCX, PPTX, XLSX, HTML 等</p>
                <p class="text-xs text-gray-400 mt-4">最大支持 100MB</p>
              </div>

              <div v-else class="flex flex-col items-center w-full">
                <UIcon name="i-heroicons-document-check" class="text-6xl text-primary-500 mb-4" />
                <p class="text-lg font-bold text-gray-900 dark:text-white break-all max-w-full mb-1">
                  {{ file.name }}
                </p>
                <p class="text-sm text-gray-500 mb-6">{{ formatSize(file.size) }}</p>
                
                <div class="flex gap-3">
                  <UButton size="sm" color="gray" variant="soft" @click.stop="triggerFileInput">
                    更换文件
                  </UButton>
                  <UButton size="sm" color="red" variant="soft" @click.stop="file = null">
                    移除
                  </UButton>
                </div>
              </div>

               <!-- Loading Overlay -->
              <div v-if="isProcessing" class="absolute inset-0 bg-white/90 dark:bg-gray-900/90 z-20 flex flex-col items-center justify-center p-8 backdrop-blur-sm rounded-xl">
                 <UIcon name="i-heroicons-arrow-path" class="text-5xl text-primary-500 animate-spin mb-4" />
                 <h3 class="text-xl font-bold text-gray-800 dark:text-white">正在转换...</h3>
                 <p class="text-gray-500 mt-2">大型文件可能需要几分钟，请耐心等待</p>
              </div>
            </div>

            <UButton 
              block 
              size="xl" 
              :color="isProcessing ? 'gray' : 'primary'"
              :loading="isProcessing"
              :disabled="!file || isProcessing"
              @click="startConversion"
            >
              <template #leading>
                <UIcon name="i-heroicons-sparkles" />
              </template>
              {{ isProcessing ? '转换中...' : '开始转换' }}
            </UButton>
          </div>

          <!-- Right: Preview Area -->
          <div class="flex flex-col h-full bg-gray-50 dark:bg-gray-950/50">
            <!-- Tabs Header -->
            <div class="flex items-center justify-between px-4 py-3 border-b border-gray-200 dark:border-gray-800 bg-white dark:bg-gray-900">
              <div class="flex space-x-1">
                 <button 
                  v-for="(tab, index) in ['Markdown 源码', '预览效果']" 
                  :key="index"
                  @click="activeTab = index"
                  class="px-4 py-2 text-sm font-medium rounded-md transition-colors"
                  :class="activeTab === index 
                    ? 'bg-primary-50 text-primary-600 dark:bg-primary-900/20 dark:text-primary-400' 
                    : 'text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-200'"
                 >
                   {{ tab }}
                 </button>
              </div>
              
              <div class="flex items-center gap-2">
                <UButton 
                  size="sm" 
                  color="gray" 
                  variant="ghost" 
                  icon="i-heroicons-clipboard-document"
                  :disabled="!resultText"
                  @click="copyToClipboard"
                >
                  复制
                </UButton>
                <UButton 
                  size="sm" 
                  color="gray" 
                  variant="solid"
                  icon="i-heroicons-arrow-down-tray"
                  :disabled="!resultText"
                  @click="downloadFile"
                >
                  保存
                </UButton>
              </div>
            </div>

            <!-- Content Area -->
            <div class="flex-1 overflow-auto p-6 relative">
              <div v-if="!resultText" class="absolute inset-0 flex flex-col items-center justify-center text-gray-400 dark:text-gray-600 pointer-events-none">
                <UIcon name="i-heroicons-document-magnifying-glass" class="text-5xl mb-3 opacity-50" />
                <p>转换后的内容将显示在这里</p>
              </div>

              <div v-else>
                <!-- Source View -->
                <div v-show="activeTab === 0">
                  <textarea 
                    readonly 
                    class="w-full h-full min-h-[500px] bg-transparent font-mono text-sm resize-none focus:outline-none text-gray-800 dark:text-gray-300"
                    :value="resultText"
                  ></textarea>
                </div>

                <!-- Preview View -->
                <div v-show="activeTab === 1">
                  <div 
                    class="prose dark:prose-invert max-w-none"
                    v-html="renderedHtml"
                  ></div>
                </div>
              </div>
            </div>
          </div>

        </div>
      </UCard>
    </div>
  </div>
</template>

<style scoped>
/* Custom scrollbar for textarea */
textarea::-webkit-scrollbar {
  width: 8px;
}
textarea::-webkit-scrollbar-track {
  background: transparent;
}
textarea::-webkit-scrollbar-thumb {
  background-color: rgba(156, 163, 175, 0.5);
  border-radius: 4px;
}
textarea::-webkit-scrollbar-thumb:hover {
  background-color: rgba(156, 163, 175, 0.8);
}
</style>
