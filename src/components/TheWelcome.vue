<script setup lang="ts">
import { ref } from 'vue'
import SparkMD5 from 'spark-md5'
import WelcomeItem from './WelcomeItem.vue'
import DocumentationIcon from './icons/IconDocumentation.vue'
import ToolingIcon from './icons/IconTooling.vue'
import EcosystemIcon from './icons/IconEcosystem.vue'
import CommunityIcon from './icons/IconCommunity.vue'
import SupportIcon from './icons/IconSupport.vue'

const openReadmeInEditor = () => fetch('/__open-in-editor?file=README.md')

// 上传配置
const API_BASE = 'http://127.0.0.1:7001/api'
const CHUNK_SIZE = 2 * 1024 * 1024 // 2MB分片
const fileInputRef = ref<HTMLInputElement>()
const logList = ref<string[]>([])

// 日志打印
const log = (text: string) => {
  const time = new Date().toLocaleTimeString()
  logList.value.push(`${time}  ${text}`)
}

// 计算文件MD5

const computeFileMD5 = (file: File): Promise<string> => {
  return new Promise((resolve) => {
    const spark = new SparkMD5.ArrayBuffer()
    const reader = new FileReader()
    reader.readAsArrayBuffer(file)
    reader.onload = (e) => {
      const buf = e.target?.result as ArrayBuffer
      spark.append(buf)
      resolve(spark.end())
    }
  })
}

// 普通上传
const handleNormalUpload = async () => {
  const file = fileInputRef.value?.files?.[0]
  if (!file) {
    alert('请先选择文件')
    return
  }
  log(`【开始普通上传】文件名:${file.name} size:${file.size}`)
  const formData = new FormData()
  formData.append('file', file)
  try {
    const res = await fetch(`${API_BASE}/upload/normal`, {
      method: 'POST',
      body: formData
    })
    const data = await res.json()
    if (!res.ok) {
      log(`普通上传服务异常：${JSON.stringify(data)}`)
      return
    }
    log(`普通上传结果：${JSON.stringify(data)}`)
  } catch (err) {
    log(`普通上传网络异常：${(err as Error).message}`)
  }
}

// 分片上传
const handleChunkUpload = async () => {
  const file = fileInputRef.value?.files?.[0]
  if (!file) {
    alert('请先选择文件')
    return
  }
  log(`【开始切片上传】文件名:${file.name} size:${file.size}`)
  log('正在计算文件MD5...')
  const fileMd5 = await computeFileMD5(file)
  log(`文件md5:${fileMd5}`)
  const totalChunks = Math.ceil(file.size / CHUNK_SIZE)
  log(`总分片数：${totalChunks}`)

  // 串行上传分片
  for (let i = 0; i < totalChunks; i++) {
    const start = i * CHUNK_SIZE
    const end = Math.min(start + CHUNK_SIZE, file.size)
    const chunkBlob = file.slice(start, end)

    const formData = new FormData()
    formData.append('md5', fileMd5)
    formData.append('chunkIndex', String(i))
    formData.append('totalChunks', String(totalChunks))
    // formData.append('chunk', chunkBlob)
    formData.append('chunk', chunkBlob, `part_${i}.png`)

    console.log('测试：', fileMd5, chunkBlob);

    log(`上传分片 ${i} / ${totalChunks - 1}`)
    const res = await fetch(`${API_BASE}/upload/chunk`, {
      method: 'POST',
      body: formData
    })
    // const chunkResp = await res.json()

    let chunkResp;
    try {
      chunkResp = await res.json();
    } catch {
      log(`分片${i}响应非JSON格式，HTTP状态：${res.status}`);
      return;
    }

    if (!res.ok) {
      log(`分片${i}上传失败：${JSON.stringify(chunkResp)}`)
      return
    }
  }

  // 合并分片
  log('全部分片上传完毕，请求后端合并文件')
  const mergeResp = await fetch(`${API_BASE}/upload/merge`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      md5: fileMd5,
      fileName: file.name,
      totalChunks
    })
  })
  const mergeData = await mergeResp.json()
  if (!mergeResp.ok) {
    log(`合并分片失败：${JSON.stringify(mergeData)}`)
    return
  }
  log(`合并完成，返回结果：${JSON.stringify(mergeData)}`)
}
</script>

<template>
  <!-- 原有官方欢迎内容完整保留 -->
  <WelcomeItem>
    <template #icon>
      <DocumentationIcon />
    </template>
    <template #heading>Documentation</template>

    Vue’s
    <a href="https://vuejs.org/" target="_blank" rel="noopener">official documentation</a>
    provides you with all information you need to get started.
  </WelcomeItem>

  <WelcomeItem>
    <template #icon>
      <ToolingIcon />
    </template>
    <template #heading>Tooling</template>

    This project is served and bundled with
    <a href="https://vite.dev/guide/features.html" target="_blank" rel="noopener">Vite</a>. The
    recommended IDE setup is
    <a href="https://code.visualstudio.com/" target="_blank" rel="noopener">VSCode</a>
    +
    <a href="https://github.com/vuejs/language-tools" target="_blank" rel="noopener"
      >Vue - Official</a
    >. If you need to test your components and web pages, check out
    <a href="https://vitest.dev/" target="_blank" rel="noopener">Vitest</a>
    and
    <a href="https://www.cypress.io/" target="_blank" rel="noopener">Cypress</a>
    /
    <a href="https://playwright.dev/" target="_blank" rel="noopener">Playwright</a>.

    <br />

    More instructions are available in
    <a href="javascript:void(0)" @click="openReadmeInEditor"><code>README.md</code></a
    >.
  </WelcomeItem>

  <WelcomeItem>
    <template #icon>
      <EcosystemIcon />
    </template>
    <template #heading>Ecosystem</template>

    Get official tools and libraries for your project:
    <a href="https://pinia.vuejs.org/" target="_blank" rel="noopener">Pinia</a>,
    <a href="https://router.vuejs.org/" target="_blank" rel="noopener">Vue Router</a>,
    <a href="https://test-utils.vuejs.org/" target="_blank" rel="noopener">Vue Test Utils</a>, and
    <a href="https://github.com/vuejs/devtools" target="_blank" rel="noopener">Vue Dev Tools</a>. If
    you need more resources, we suggest paying
    <a href="https://github.com/vuejs/awesome-vue" target="_blank" rel="noopener">Awesome Vue</a>
    a visit.
  </WelcomeItem>

  <WelcomeItem>
    <template #icon>
      <CommunityIcon />
    </template>
    <template #heading>Community</template>

    Got stuck? Ask your question on
    <a href="https://chat.vuejs.org" target="_blank" rel="noopener">Vue Land</a>
    (our official Discord server), or
    <a href="https://stackoverflow.com/questions/tagged/vue.js" target="_blank" rel="noopener"
      >StackOverflow</a
    >. You should also follow the official
    <a href="https://bsky.app/profile/vuejs.org" target="_blank" rel="noopener">@vuejs.org</a>
    Bluesky account or the
    <a href="https://x.com/vuejs" target="_blank" rel="noopener">@vuejs</a>
    X account for latest news in the Vue world.
  </WelcomeItem>

  <WelcomeItem>
    <template #icon>
      <SupportIcon />
    </template>
    <template #heading>Support Vue</template>

    As an independent project, Vue relies on community backing for its sustainability. You can help
    us by
    <a href="https://vuejs.org/sponsor/" target="_blank" rel="noopener">becoming a sponsor</a>.
  </WelcomeItem>

  <!-- 新增：文件上传区域（嵌入原有页面结构内） -->
  <div style="margin-top: 40px;padding:20px;border:1px dashed #888;border-radius:8px;">
    <h3 style="margin:0 0 16px 0;">文件上传测试（普通上传 / 分片上传）</h3>
    <input type="file" ref="fileInputRef" style="margin-bottom:12px;" />
    <div style="gap:12px;display:flex;margin-bottom:16px;">
      <button @click="handleNormalUpload" style="padding:6px 14px;cursor:pointer;">普通完整上传</button>
      <button @click="handleChunkUpload" style="padding:6px 14px;cursor:pointer;">切片分片上传(2MB每块)</button>
    </div>
    <div>日志输出：</div>
    <div style="margin-top:8px;padding:10px;border:1px solid #ccc;min-height:180px;white-space:pre-wrap;font-size:12px;overflow:auto;">
      <div v-for="(item,idx) in logList" :key="idx">{{ item }}</div>
    </div>
  </div>
</template>