<script setup>
import { ref, computed } from 'vue'
import Swal from 'sweetalert2'
import 'sweetalert2/themes/bootstrap-5.css'

// ==========================================
// 1. State Management (ตัวแปรสถานะ)
// ==========================================
const requestMethod = ref('GET')
const requestBaseUrl = ref('https://jsonplaceholder.typicode.com')
const requestEndpoint = ref('/posts/1')
const activeRequestTab = ref('params')
const isRequestLoading = ref(false)
const apiResponse = ref(null)
const requestError = ref(null)
const activePresetIndex = ref(0)

const requestQueryParams = ref([{ key: '', value: '' }])
const requestHeaders = ref([{ key: 'Content-Type', value: 'application/json' }])
const requestBody = ref(`{
  "title": "foo",
  "body": "bar",
  "userId": 1
}`)

// ==========================================
// 2. Data Presets (ข้อมูลตัวอย่าง API)
// ==========================================
const apiPresets = ref([
    {
        presetName: 'ดึงข้อมูลโพสต์',
        category: 'Posts',
        method: 'GET',
        baseUrl: 'https://jsonplaceholder.typicode.com',
        endpoint: '/posts/1',
        body: '',
    },
    {
        presetName: 'สร้างโพสต์ใหม่',
        category: 'Posts',
        method: 'POST',
        baseUrl: 'https://jsonplaceholder.typicode.com',
        endpoint: '/posts',
        body: '{\n  "title": "foo",\n  "body": "bar",\n  "userId": 1\n}',
    },
    {
        presetName: 'ดึงข้อมูลผู้ใช้',
        category: 'Users',
        method: 'GET',
        baseUrl: 'https://jsonplaceholder.typicode.com',
        endpoint: '/users/1',
        body: '',
    },
    {
        presetName: 'อัพเดทผู้ใช้ (Patch)',
        category: 'Users',
        method: 'PATCH',
        baseUrl: 'https://jsonplaceholder.typicode.com',
        endpoint: '/users/1',
        body: '{\n  "name": "John Doe Updated"\n}',
    },
])

// ==========================================
// 3. Methods (ฟังก์ชันการทำงาน)
// ==========================================
const loadPreset = (presetIndex) => {
    activePresetIndex.value = presetIndex
    const selectedPreset = apiPresets.value[presetIndex]

    requestMethod.value = selectedPreset.method
    requestBaseUrl.value = selectedPreset.baseUrl
    requestEndpoint.value = selectedPreset.endpoint
    requestBody.value = selectedPreset.body

    // รีเซ็ตแท็บและผลลัพธ์ก่อนหน้า
    activeRequestTab.value = selectedPreset.method === 'GET' ? 'params' : 'body'
    apiResponse.value = null
    requestError.value = null
}

const addKeyValuePair = (targetArray) => {
    targetArray.push({ key: '', value: '' })
}

const removeKeyValuePair = (targetArray, index) => {
    targetArray.splice(index, 1)
}

const constructRequestUrl = () => {
    const cleanBase = requestBaseUrl.value.replace(/\/$/, '')
    const cleanEnd = requestEndpoint.value.replace(/^\//, '')
    let fullUrl = `${cleanBase}/${cleanEnd}`

    // เพิ่ม Query Parameters
    const validParams = requestQueryParams.value.filter((param) => param.key.trim() !== '')
    if (validParams.length > 0) {
        const queryString = new URLSearchParams(
            validParams.map((param) => [param.key, param.value]),
        ).toString()
        fullUrl += `?${queryString}`
    }
    return fullUrl
}

const executeApiRequest = async () => {
    isRequestLoading.value = true
    requestError.value = null
    apiResponse.value = null

    // แปลง Array ของ Headers เป็น Object
    const headersObject = {}
    requestHeaders.value.forEach((header) => {
        if (header.key.trim()) {
            headersObject[header.key.trim()] = header.value.trim()
        }
    })

    const fetchOptions = {
        method: requestMethod.value,
        headers: headersObject,
    }

    // เพิ่ม Body สำหรับ Method ที่รองรับ
    if (['POST', 'PUT', 'PATCH'].includes(requestMethod.value) && requestBody.value.trim()) {
        fetchOptions.body = requestBody.value
    }

    const startTime = performance.now()

    try {
        const response = await fetch(constructRequestUrl(), fetchOptions)
        const endTime = performance.now()

        let responseData
        const contentType = response.headers.get('content-type')

        if (contentType && contentType.includes('application/json')) {
            responseData = await response.json()
        } else {
            responseData = await response.text()
        }

        apiResponse.value = {
            statusCode: response.status,
            statusText: response.statusText,
            executionTime: (endTime - startTime).toFixed(0),
            data: responseData,
        }
    } catch (error) {
        requestError.value = error.message
    } finally {
        isRequestLoading.value = false
    }
}

const copyResponseToClipboard = () => {
    if (apiResponse.value) {
        const textToCopy =
            typeof apiResponse.value.data === 'object'
                ? JSON.stringify(apiResponse.value.data, null, 2)
                : apiResponse.value.data

        navigator.clipboard
            .writeText(textToCopy)
            .then(() => {
                CopyAlert({
                    title: 'สำเร็จแล้ว!',
                    text: 'คัดลอกข้อความสำเร็จแล้ว',
                    icon: 'success',
                })
            })
            .catch(() => {
                CopyAlert({
                    title: 'ไม่สำเร็จ!',
                    text: 'เกิดข้อผิดพลาดในการคัดลอกข้อความ',
                    icon: 'success',
                })
            })
    }
}

// ==========================================
// 4. SweetAlert
// ==========================================
const CopyAlert = ({ title, text, icon }) => {
    return Swal.fire({
        title: title,
        text: text,
        icon: icon,
        theme: 'bootstrap-5-light',
    })
}

// ==========================================
// 5. Computed Properties (คุณสมบัติคำนวณ)
// ==========================================
const formattedApiResponse = computed(() => {
    if (!apiResponse.value) return ''
    if (typeof apiResponse.value.data === 'object') {
        return JSON.stringify(apiResponse.value.data, null, 2)
    }
    return apiResponse.value.data
})
</script>

<template>
    <div
        class="container-fluid py-4 px-4 d-flex flex-column"
        style="max-width: 1400px; min-height: 100vh"
    >
        <!-- Header Section -->
        <header class="d-flex align-items-center mb-4">
            <div class="bg-primary text-white p-2 radius-md me-3">
                <i class="bi bi-lightning-charge-fill fs-4"></i>
            </div>
            <div>
                <h1 class="h4 mb-0 fw-bold text-dark">RESTful API Client</h1>
                <small class="text-muted"
                    >ทดสอบและตรวจสอบ API Endpoint อย่างรวดเร็วและทันสมัย</small
                >
            </div>
        </header>

        <!-- Main Content Area (ขยายเต็มพื้นที่ที่เหลือ) -->
        <main class="flex-grow-1">
            <div class="row g-4">
                <!-- Sidebar: API Presets -->
                <aside class="col-lg-3">
                    <div class="modern-card p-3 h-100">
                        <h6 class="fw-bold text-primary mb-3">
                            <i class="bi bi-collection me-2"></i>ตัวอย่าง API
                        </h6>
                        <div class="list-group list-group-flush">
                            <div
                                v-for="(preset, index) in apiPresets"
                                :key="index"
                                class="list-group-item list-group-item-action preset-item radius-md mb-2 p-3"
                                :class="{ active: activePresetIndex === index }"
                                @click="loadPreset(index)"
                            >
                                <div class="d-flex justify-content-between align-items-start">
                                    <span
                                        class="badge bg-primary bg-opacity-10 text-primary radius-md mb-1"
                                    >
                                        {{ preset.method }}
                                    </span>
                                    <small class="text-muted">{{ preset.category }}</small>
                                </div>
                                <div
                                    class="fw-semibold text-truncate text-primary"
                                    :title="preset.endpoint"
                                >
                                    {{ preset.presetName }}
                                </div>
                                <small class="text-muted" style="font-size: 0.75rem">{{
                                    preset.endpoint
                                }}</small>
                            </div>
                        </div>
                    </div>
                </aside>

                <!-- Request & Response Section -->
                <section class="col-lg-9">
                    <!-- Request Configuration Card -->
                    <div class="modern-card p-4 mb-4">
                        <!-- URL Bar -->
                        <div class="d-flex gap-2 mb-3 flex-wrap flex-md-nowrap">
                            <select
                                v-model="requestMethod"
                                class="form-select radius-md fw-bold text-primary"
                                style="min-width: 110px; max-width: 130px"
                            >
                                <option value="GET">GET</option>
                                <option value="POST">POST</option>
                                <option value="PUT">PUT</option>
                                <option value="PATCH">PATCH</option>
                                <option value="DELETE">DELETE</option>
                            </select>

                            <input
                                v-model="requestBaseUrl"
                                type="text"
                                class="form-control radius-md"
                                placeholder="Base URL (เช่น https://jsonplaceholder.typicode.com)"
                            />

                            <div class="input-group" style="max-width: 300px">
                                <span class="input-group-text bg-light border-end-0 text-muted"
                                    >/</span
                                >
                                <input
                                    v-model="requestEndpoint"
                                    type="text"
                                    class="form-control border-start-0 radius-md"
                                    placeholder="Endpoint (เช่น /posts/1)"
                                />
                            </div>

                            <button
                                class="btn btn-primary radius-md px-4 d-flex align-items-center gap-2"
                                @click="executeApiRequest"
                                :disabled="isRequestLoading"
                            >
                                <span
                                    v-if="isRequestLoading"
                                    class="spinner-border spinner-border-sm"
                                ></span>
                                <i v-else class="bi bi-send-fill"></i>
                                <span>ส่งคำขอ</span>
                            </button>
                        </div>

                        <!-- Request Tabs -->
                        <ul class="nav nav-pills mb-3 gap-2">
                            <li class="nav-item">
                                <button
                                    class="nav-link radius-md px-3"
                                    :class="{
                                        active: activeRequestTab === 'params',
                                        'bg-primary text-white': activeRequestTab === 'params',
                                    }"
                                    @click="activeRequestTab = 'params'"
                                >
                                    <i class="bi bi-sliders me-1"></i> Params
                                </button>
                            </li>
                            <li class="nav-item">
                                <button
                                    class="nav-link radius-md px-3"
                                    :class="{
                                        active: activeRequestTab === 'headers',
                                        'bg-primary text-white': activeRequestTab === 'headers',
                                    }"
                                    @click="activeRequestTab = 'headers'"
                                >
                                    <i class="bi bi-card-heading me-1"></i> Headers
                                </button>
                            </li>
                            <li
                                class="nav-item"
                                v-if="['POST', 'PUT', 'PATCH'].includes(requestMethod)"
                            >
                                <button
                                    class="nav-link radius-md px-3"
                                    :class="{
                                        active: activeRequestTab === 'body',
                                        'bg-primary text-white': activeRequestTab === 'body',
                                    }"
                                    @click="activeRequestTab = 'body'"
                                >
                                    <i class="bi bi-code-square me-1"></i> Body (JSON)
                                </button>
                            </li>
                        </ul>

                        <!-- Tab Contents -->
                        <div class="tab-content">
                            <!-- Params Tab -->
                            <div v-if="activeRequestTab === 'params'">
                                <div
                                    v-for="(param, index) in requestQueryParams"
                                    :key="index"
                                    class="row g-2 mb-2 align-items-center"
                                >
                                    <div class="col-5">
                                        <input
                                            v-model="param.key"
                                            class="form-control radius-md"
                                            placeholder="Key (เช่น id)"
                                        />
                                    </div>
                                    <div class="col-6">
                                        <input
                                            v-model="param.value"
                                            class="form-control radius-md"
                                            placeholder="Value (เช่น 1)"
                                        />
                                    </div>
                                    <div class="col-1">
                                        <button
                                            class="btn btn-outline-primary w-100 radius-md"
                                            @click="removeKeyValuePair(requestQueryParams, index)"
                                            title="ลบ"
                                        >
                                            <i class="bi bi-x-lg"></i>
                                        </button>
                                    </div>
                                </div>
                                <button
                                    class="btn btn-primary btn-sm radius-md mt-2"
                                    @click="addKeyValuePair(requestQueryParams)"
                                >
                                    <i class="bi bi-plus-lg me-1"></i> เพิ่ม Parameter
                                </button>
                            </div>

                            <!-- Headers Tab -->
                            <div v-if="activeRequestTab === 'headers'">
                                <div
                                    v-for="(header, index) in requestHeaders"
                                    :key="index"
                                    class="row g-2 mb-2 align-items-center"
                                >
                                    <div class="col-5">
                                        <input
                                            v-model="header.key"
                                            class="form-control radius-md"
                                            placeholder="Key (เช่น Content-Type)"
                                        />
                                    </div>
                                    <div class="col-6">
                                        <input
                                            v-model="header.value"
                                            class="form-control radius-md"
                                            placeholder="Value (เช่น application/json)"
                                        />
                                    </div>
                                    <div class="col-1">
                                        <button
                                            class="btn btn-outline-primary w-100 radius-md"
                                            @click="removeKeyValuePair(requestHeaders, index)"
                                            title="ลบ"
                                        >
                                            <i class="bi bi-x-lg"></i>
                                        </button>
                                    </div>
                                </div>
                                <button
                                    class="btn btn-primary btn-sm radius-md mt-2"
                                    @click="addKeyValuePair(requestHeaders)"
                                >
                                    <i class="bi bi-plus-lg me-1"></i> เพิ่ม Header
                                </button>
                            </div>

                            <!-- Body Tab -->
                            <div v-if="activeRequestTab === 'body'">
                                <textarea
                                    v-model="requestBody"
                                    class="form-control radius-md font-monospace"
                                    rows="8"
                                    placeholder='{ "key": "value" }'
                                ></textarea>
                            </div>
                        </div>
                    </div>

                    <!-- Response Section -->
                    <div v-if="apiResponse || requestError" class="modern-card p-4">
                        <div class="d-flex justify-content-between align-items-center mb-3">
                            <h6 class="fw-bold mb-0 text-primary">
                                <i class="bi bi-arrow-return-left me-2"></i>Response
                            </h6>
                            <div v-if="apiResponse" class="d-flex gap-2">
                                <span
                                    class="badge bg-success bg-opacity-10 text-success radius-md px-3 py-2"
                                >
                                    {{ apiResponse.statusCode }} {{ apiResponse.statusText }}
                                </span>
                                <span
                                    class="badge bg-primary bg-opacity-10 text-primary radius-md px-3 py-2"
                                >
                                    <i class="bi bi-clock me-1"></i
                                    >{{ apiResponse.executionTime }} ms
                                </span>
                                <button
                                    class="btn btn-outline-primary btn-sm radius-md"
                                    type="button"
                                    @click="copyResponseToClipboard"
                                    title="คัดลอก"
                                >
                                    <i class="bi bi-clipboard"></i>
                                </button>
                            </div>
                        </div>

                        <div
                            v-if="requestError"
                            class="alert alert-danger radius-md d-flex align-items-center"
                        >
                            <i class="bi bi-exclamation-triangle-fill fs-4 me-3"></i>
                            <div>
                                <strong>เกิดข้อผิดพลาด:</strong> {{ requestError }}
                                <div class="small text-muted mt-1">
                                    *อาจเกิดจากข้อจำกัด CORS ของเบราว์เซอร์ หรือ URL ไม่ถูกต้อง
                                </div>
                            </div>
                        </div>

                        <div v-if="apiResponse" class="code-block">
                            <pre>{{ formattedApiResponse }}</pre>
                        </div>
                    </div>
                </section>
            </div>
        </main>

        <!-- Footer Component -->
        <footer class="mt-5 py-4 border-top text-center text-muted">
            <div class="container">
                <small class="d-block mb-1">
                    <strong>Modern RESTful API Client</strong>
                </small>
                <small class="d-block">
                    Developed with Vue.js 3, Bootstrap 5 & Bootstrap Icons
                </small>
                <small class="d-block mt-2 text-black-50" style="font-size: 0.75rem">
                    &copy;
                    <span id="copyright-year">{{ new Date().getFullYear() }}</span> PluemStudio
                    (MewChan). Open Source (MIT License).
                </small>
            </div>
        </footer>
    </div>
</template>

<style scoped>
/* ฟอนต์สำหรับ Code Block */
.font-monospace {
    font-family: 'JetBrains Mono', 'Fira Code', 'Courier New', monospace;
}
</style>
