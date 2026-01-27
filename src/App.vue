<script setup lang="ts">
import { ref, computed } from 'vue'
import MarkdownIt from 'markdown-it'
import { useDark, useToggle } from '@vueuse/core'

const isDark = useDark()
const toggleDark = useToggle(isDark)

// --- State ---
const file = ref<File | null>(null)
const isProcessing = ref(false)
const isDownloadingModel = ref(false)
const resultText = ref('')
const resultFilename = ref('')
const activeTab = ref(0) // 0: Source, 1: Preview
const md = new MarkdownIt()
const renderedHtml = computed(() => md.render(resultText.value))
const dragOver = ref(false)

// --- File Handling ---
const handleDrop = (event: DragEvent) => {
  event.preventDefault()
  dragOver.value = false
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
  resultText.value = ''
  resultFilename.value = ''
  activeTab.value = 0
  startConversion() // Auto start as per requirement "download... then start calling"
}

const triggerFileInput = () => {
  const input = document.getElementById('file-upload') as HTMLInputElement
  if (input) input.click()
}

// --- Conversion Logic ---
// We use Pyodide to run the official Python markitdown package in the browser.
let pyodide: any = null

const loadPyodideEnvironment = async () => {
  if (pyodide) return pyodide
  
  isDownloadingModel.value = true
  try {
    // @ts-ignore
    pyodide = await loadPyodide()
    
    // Load necessary packages
    await pyodide.loadPackage("micropip")
    const micropip = pyodide.pyimport("micropip")
    
    // 1. Mock heavy/binary dependencies that break Pyodide (WASM)
    // MarkItDown depends on onnxruntime and magika (Google's file detection)
    // Both rely on C-extensions/WASM not easily available or too heavy.
    // We mock them. MarkItDown will fallback to extension-based detection or fail gracefully on specific features.
    pyodide.runPython(`
      import sys
      from unittest.mock import MagicMock
      
      class MockModule(MagicMock):
          def __getattr__(self, name):
              return MagicMock()
      
      sys.modules["onnxruntime"] = MockModule()
      sys.modules["magika"] = MockModule()
    `)

    // 2. Install Safe Dependencies Manually
    // We load standard data science stack supported by Pyodide
    await pyodide.loadPackage(['pandas', 'beautifulsoup4', 'micropip'])
    
    // Install other pure python dependencies
    // Note: We try best effort support.
    try {
        // These are pure python wheels available on PyPI
        await micropip.install([
            'openpyxl', 
            'python-docx', 
            'pypdf', 
            'markdown', 
            'lxml', 
            'markdownify', 
            'defusedxml',
            'charset-normalizer',
            'requests',
            'mammoth',
            'xlrd',
            'python-pptx',
            'olefile',
            'pdfminer.six'
        ])
    } catch(e) { console.warn("Failed to install some optional deps", e) }

    // 3. Install markitdown WITHOUT dependencies
    // This requires passing deps=False to micropip.install
    // We do this via Python to ensure kwargs are handled correctly if JS bridge is finicky
    await pyodide.runPythonAsync(`
        import micropip
        await micropip.install('markitdown', deps=False)
    `)
    
    // We create a simple python wrapper script
    pyodide.runPython(`
      from markitdown import MarkItDown
      import os
      
      def convert_file(input_path):
          md = MarkItDown()
          result = md.convert(input_path)
          return result.text_content
    `)
    
    return pyodide
  } catch (e: any) {
    console.error("Failed to load Pyodide or MarkItDown", e)
    throw new Error(`Failed to initialize Python environment: ${e.message}`)
  } finally {
    isDownloadingModel.value = false
  }
}

const startConversion = async () => {
  if (!file.value) return

  isProcessing.value = true
  resultText.value = ''
  
  try {
    const py = await loadPyodideEnvironment()
    
    // 1. Write file to Pyodide FS
    const arrayBuffer = await file.value.arrayBuffer()
    const filename = file.value.name
    // Remove spaces/special chars from temp path to avoid issues, though FS handles most.
    const tempPath = '/tmp/' + filename
    
    py.FS.writeFile(tempPath, new Uint8Array(arrayBuffer))
    
    // 2. Call Python function
    const convertFunc = py.globals.get('convert_file')
    const markdown = convertFunc(tempPath)
    
    // 3. Cleanup
    try {
        py.FS.unlink(tempPath)
    } catch(e) {} // ignore cleanup errors
    
    if (markdown) {
       resultText.value = markdown
       const nameParts = file.value.name.split('.')
       if (nameParts.length > 1) nameParts.pop()
       resultFilename.value = nameParts.join('.') + '.md'
    } else {
       throw new Error("Conversion returned empty result")
    }

  } catch (error: any) {
    console.error(error)
    alert(`Conversion Failed: ${error.message || error}`)
  } finally {
    isProcessing.value = false
  }
}

// --- Helpers ---
const copyToClipboard = () => {
  if (!resultText.value) return
  navigator.clipboard.writeText(resultText.value)
  alert('Copied to clipboard!')
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
  <div :class="{ 'dark': isDark }">
    <div class="min-h-screen bg-gray-50 dark:bg-gray-950 text-gray-900 dark:text-gray-100 transition-colors duration-300">
      
      <!-- Navbar / Header -->
      <nav class="border-b border-gray-200 dark:border-gray-800 bg-white/80 dark:bg-gray-900/80 backdrop-blur-md sticky top-0 z-50">
        <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 h-16 flex items-center justify-between">
          <div class="flex items-center gap-3">
             <div class="w-10 h-10 rounded-xl bg-primary-600 flex items-center justify-center text-white shadow-lg shadow-primary-500/30">
                <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6">
                  <path stroke-linecap="round" stroke-linejoin="round" d="M19.5 14.25v-2.625a3.375 3.375 0 0 0-3.375-3.375h-1.5A1.125 1.125 0 0 1 13.5 7.125v-1.5a3.375 3.375 0 0 0-3.375-3.375H8.25m2.25 0H5.625c-.621 0-1.125.504-1.125 1.125v17.25c0 .621.504 1.125 1.125 1.125h12.75c.621 0 1.125-.504 1.125-1.125V11.25a9 9 0 0 0-9-9Z" />
                </svg>
             </div>
             <h1 class="text-xl font-bold bg-clip-text text-transparent bg-gradient-to-r from-primary-600 to-purple-600">MarkItDown</h1>
          </div>
          
          <div class="flex items-center gap-4">
            <button @click="toggleDark()" class="p-2 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-800 transition-colors">
              <svg v-if="isDark" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-5 h-5">
                <path stroke-linecap="round" stroke-linejoin="round" d="M12 3v2.25m6.364.386-1.591 1.591M21 12h-2.25m-.386 6.364-1.591-1.591M12 18.75V21m-4.773-4.227-1.591 1.591M5.25 12H3m4.227-4.773L5.636 5.636M15.75 12a3.75 3.75 0 1 1-7.5 0 3.75 3.75 0 0 1 7.5 0Z" />
              </svg>
              <svg v-else xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-5 h-5">
                <path stroke-linecap="round" stroke-linejoin="round" d="M21.752 15.002A9.75 9.75 0 0 1 18 15.75c-5.385 0-9.75-4.365-9.75-9.75 0-1.33.266-2.597.748-3.752A9.753 9.753 0 0 0 3 11.25C3 16.635 7.365 21 12.75 21a9.753 9.753 0 0 0 9.002-5.998Z" />
              </svg>
            </button>
            <a href="https://github.com/budaobu/markitdown" target="_blank" class="p-2 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-800 transition-colors" title="GitHub">
              <svg role="img" viewBox="0 0 24 24" fill="currentColor" class="w-5 h-5"><path d="M12 .297c-6.63 0-12 5.373-12 12 0 5.303 3.438 9.8 8.205 11.385.6.113.82-.258.82-.577 0-.285-.01-1.04-.015-2.04-3.338.724-4.042-1.61-4.042-1.61C4.422 18.07 3.633 17.7 3.633 17.7c-1.087-.744.084-.729.084-.729 1.205.084 1.838 1.236 1.838 1.236 1.07 1.835 2.809 1.305 3.495.998.108-.776.417-1.305.76-1.605-2.665-.3-5.466-1.332-5.466-5.93 0-1.31.465-2.38 1.235-3.22-.135-.303-.54-1.523.105-3.176 0 0 1.005-.322 3.3 1.23.96-.267 1.98-.399 3-.405 1.02.006 2.04.138 3 .405 2.28-1.552 3.285-1.23 3.285-1.23.645 1.653.24 2.873.12 3.176.765.84 1.23 1.91 1.23 3.22 0 4.61-2.805 5.625-5.475 5.92.42.36.81 1.096.81 2.22 0 1.606-.015 2.896-.015 3.286 0 .315.21.69.825.57C20.565 22.092 24 17.592 24 12.297c0-6.627-5.373-12-12-12"/></svg>
            </a>
          </div>
        </div>
      </nav>

      <main class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-12">
        <div class="text-center mb-16">
          <h2 class="text-4xl sm:text-5xl font-extrabold text-gray-900 dark:text-white tracking-tight mb-4">
            Convert Any File <br class="sm:hidden" /> to <span class="text-primary-600">Markdown</span> in Seconds.
          </h2>
          <p class="max-w-2xl mx-auto text-lg text-gray-600 dark:text-gray-400">
            Support for PDF, Word, Excel, PowerPoint and more. Secure, private, and runs entirely in your browser.
          </p>
        </div>

        <div class="grid grid-cols-1 lg:grid-cols-2 gap-8 min-h-[600px]">
          
          <!-- Upload Section -->
          <div class="bg-white dark:bg-gray-900 rounded-3xl shadow-xl overflow-hidden border border-gray-100 dark:border-gray-800 flex flex-col relative transition-all duration-300 hover:shadow-2xl">
            <div class="p-6 border-b border-gray-100 dark:border-gray-800 bg-gray-50/50 dark:bg-gray-900/50">
               <h3 class="text-xl font-bold text-gray-900 dark:text-white flex items-center gap-2">
                 Source File
               </h3>
            </div>
            
            <div 
               class="flex-1 p-8 flex flex-col items-center justify-center text-center cursor-pointer transition-colors relative"
               :class="[
                 dragOver ? 'bg-primary-50/50 dark:bg-primary-900/20' : '',
                 !file ? 'hover:bg-gray-50 dark:hover:bg-gray-800/50' : ''
               ]"
               @click="!file && triggerFileInput()"
               @dragover.prevent="dragOver = true"
               @dragleave.prevent="dragOver = false"
               @drop="handleDrop"
             >
               <input id="file-upload" type="file" class="hidden" @change="handleFileChange">
               
               <!-- Loading Overlay -->
               <div v-if="isProcessing || isDownloadingModel" class="absolute inset-0 z-20 bg-white/95 dark:bg-gray-900/95 flex flex-col items-center justify-center backdrop-blur-sm">
                  <div class="relative w-24 h-24 mb-6">
                    <svg class="animate-spin w-full h-full text-primary-500" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                      <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                      <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                    </svg>
                  </div>
                  <h3 class="text-2xl font-bold text-gray-900 dark:text-white mb-2">
                    {{ isDownloadingModel ? 'Downloading Model...' : 'Converting...' }}
                  </h3>
                  <p class="text-gray-500 dark:text-gray-400 max-w-xs text-center">
                    {{ isDownloadingModel ? 'Setting up the conversion engine for the first time.' : 'Processing your file securely.' }}
                  </p>
               </div>

               <div v-if="!file" class="flex flex-col items-center gap-6">
                 <div class="w-24 h-24 rounded-full bg-primary-50 dark:bg-gray-800 flex items-center justify-center text-primary-500 dark:text-primary-400">
                    <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-10 h-10">
                      <path stroke-linecap="round" stroke-linejoin="round" d="M12 16.5V9.75m0 0 3 3m-3-3-3 3M6.75 19.5a4.5 4.5 0 0 1-1.41-8.775 5.25 5.25 0 0 1 10.233-2.33 3 3 0 0 1 3.758 3.848A3.752 3.752 0 0 1 18 19.5H6.75Z" />
                    </svg>
                 </div>
                 <div>
                   <p class="text-xl font-semibold text-gray-900 dark:text-white mb-2">Click or Drag file here</p>
                   <p class="text-sm text-gray-500 dark:text-gray-400">Max file size 100MB</p>
                 </div>
               </div>

               <div v-else class="flex flex-col items-center w-full max-w-sm">
                 <div class="w-20 h-20 bg-green-50 dark:bg-green-900/20 rounded-2xl flex items-center justify-center text-green-600 dark:text-green-400 mb-6">
                    <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-10 h-10">
                      <path stroke-linecap="round" stroke-linejoin="round" d="M19.5 14.25v-2.625a3.375 3.375 0 0 0-3.375-3.375h-1.5A1.125 1.125 0 0 1 13.5 7.125v-1.5a3.375 3.375 0 0 0-3.375-3.375H8.25m0 12.75h7.5m-7.5 3H12M10.5 2.25H5.625c-.621 0-1.125.504-1.125 1.125v17.25c0 .621.504 1.125 1.125 1.125h12.75c.621 0 1.125-.504 1.125-1.125V11.25a9 9 0 0 0-9-9Z" />
                    </svg>
                 </div>
                 <h4 class="text-lg font-bold text-gray-900 dark:text-white break-all mb-2">{{ file.name }}</h4>
                 <p class="text-sm text-gray-500 dark:text-gray-400 mb-8">{{ formatSize(file.size) }}</p>
                 
                 <div class="flex gap-4 w-full">
                    <button @click.stop="triggerFileInput" class="flex-1 py-2.5 px-4 rounded-xl bg-gray-100 dark:bg-gray-800 text-gray-700 dark:text-gray-300 font-medium hover:bg-gray-200 dark:hover:bg-gray-700 transition-colors">
                      Change
                    </button>
                    <button @click.stop="file = null; resultText = ''" class="flex-1 py-2.5 px-4 rounded-xl bg-red-50 dark:bg-red-900/20 text-red-600 dark:text-red-400 font-medium hover:bg-red-100 dark:hover:bg-red-900/30 transition-colors">
                      Remove
                    </button>
                 </div>
               </div>
            </div>
          </div>

          <!-- Result Section -->
          <div class="bg-white dark:bg-gray-900 rounded-3xl shadow-xl overflow-hidden border border-gray-100 dark:border-gray-800 flex flex-col relative transition-all duration-300 hover:shadow-2xl">
             <div class="p-4 border-b border-gray-100 dark:border-gray-800 flex items-center justify-between bg-gray-50/50 dark:bg-gray-900/50 h-[72px]">
                <div class="flex bg-gray-200 dark:bg-gray-800 p-1 rounded-xl">
                   <button 
                     v-for="(t, i) in ['Markdown', 'Preview']" 
                     :key="i"
                     @click="activeTab = i"
                     class="px-4 py-1.5 rounded-lg text-sm font-medium transition-all"
                     :class="activeTab === i ? 'bg-white dark:bg-gray-700 shadow-sm text-gray-900 dark:text-white' : 'text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-gray-200'"
                   >
                     {{ t }}
                   </button>
                </div>
                
                <div class="flex gap-2">
                   <button 
                      @click="copyToClipboard" 
                      :disabled="!resultText"
                      class="p-2 text-gray-500 hover:text-primary-600 dark:text-gray-400 dark:hover:text-primary-400 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
                      title="Copy"
                   >
                      <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-5 h-5">
                        <path stroke-linecap="round" stroke-linejoin="round" d="M15.75 17.25v3.375c0 .621-.504 1.125-1.125 1.125h-9.75a1.125 1.125 0 0 1-1.125-1.125V7.875c0-.621.504-1.125 1.125-1.125H6.75a9.06 9.06 0 0 1 1.5.124m7.5 10.376h3.375c.621 0 1.125-.504 1.125-1.125V11.25c0-4.46-3.243-8.161-7.5-8.876a9.06 9.06 0 0 0-1.5-.124H9.375c-.621 0-1.125.504-1.125 1.125v3.5m7.5 10.381a9.06 9.06 0 0 1-1.5-.124A9.06 9.06 0 0 1 13.5 17.25" />
                      </svg>
                   </button>
                   <button 
                      @click="downloadFile" 
                      :disabled="!resultText"
                      class="p-2 text-gray-500 hover:text-primary-600 dark:text-gray-400 dark:hover:text-primary-400 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
                      title="Download"
                   >
                      <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-5 h-5">
                        <path stroke-linecap="round" stroke-linejoin="round" d="M3 16.5v2.25A2.25 2.25 0 0 0 5.25 21h13.5A2.25 2.25 0 0 0 21 18.75V16.5M16.5 12 12 16.5m0 0L7.5 12m4.5 4.5V3" />
                      </svg>
                   </button>
                </div>
             </div>

             <div class="flex-1 overflow-auto bg-gray-50/50 dark:bg-gray-950/50 relative">
                <textarea 
                  v-if="activeTab === 0"
                  class="w-full h-full p-6 bg-transparent resize-none focus:outline-none font-mono text-sm text-gray-800 dark:text-gray-300 leading-relaxed" 
                  readonly 
                  :value="resultText"
                  placeholder="Converted markdown will appear here..."
                ></textarea>
                <div 
                  v-if="activeTab === 1" 
                  class="p-8 prose dark:prose-invert max-w-none prose-sm sm:prose-base"
                  v-html="renderedHtml"
                ></div>
                
                <div v-if="!resultText" class="absolute inset-0 flex flex-col items-center justify-center text-gray-400 dark:text-gray-600 pointer-events-none opacity-50">
                    <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-16 h-16 mb-4">
                      <path stroke-linecap="round" stroke-linejoin="round" d="M19.5 14.25v-2.625a3.375 3.375 0 0 0-3.375-3.375h-1.5A1.125 1.125 0 0 1 13.5 7.125v-1.5a3.375 3.375 0 0 0-3.375-3.375H8.25m2.25 0H5.625c-.621 0-1.125.504-1.125 1.125v17.25c0 .621.504 1.125 1.125 1.125h12.75c.621 0 1.125-.504 1.125-1.125V11.25a9 9 0 0 0-9-9Z" />
                    </svg>
                    <p class="text-lg font-medium">Waiting for content...</p>
                </div>
             </div>
          </div>
          
        </div>
      </main>
    </div>
  </div>
</template>

<style>
/* Utilities */
.prose pre {
  background-color: #f3f4f6;
  color: #1f2937;
  border-radius: 0.5rem;
}
.dark .prose pre {
  background-color: #1f2937;
  color: #e5e7eb;
}
</style>
