<script setup lang="ts">
import { computed, ref, watch } from 'vue';

interface PdfJsPage {
    getViewport: (params: { scale: number }) => { width: number; height: number };
    render: (params: { canvasContext: CanvasRenderingContext2D; viewport: { width: number; height: number } }) => {
        promise: Promise<void>;
    };
    getTextContent: () => Promise<{ items: Array<{ str?: string }> }>;
}

interface PdfJsDocument {
    numPages: number;
    getPage: (page: number) => Promise<PdfJsPage>;
}

const props = defineProps<{
    src: string;
}>();

const PDF_JS_CDN_URL = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.10.38/pdf.min.mjs';
const PDF_JS_WORKER_CDN_URL = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.10.38/pdf.worker.min.mjs';

const canvas = ref<HTMLCanvasElement | null>(null);
const pdfDoc = ref<PdfJsDocument | null>(null);
const currentPage = ref(1);
const totalPages = ref(0);
const zoom = ref(1);
const searchQuery = ref('');
const matchedPages = ref<number[]>([]);
const currentMatchIndex = ref(0);
const loading = ref(false);
const loadError = ref(false);
const fallbackUrl = computed(() => `assets://${props.src}`);

async function loadPdfJs() {
    const pdfjsLib = await import(/* @vite-ignore */ PDF_JS_CDN_URL);
    if (pdfjsLib.GlobalWorkerOptions) {
        pdfjsLib.GlobalWorkerOptions.workerSrc = PDF_JS_WORKER_CDN_URL;
    }
    return pdfjsLib;
}

async function renderPage() {
    if (!pdfDoc.value || !canvas.value) return;

    const page = await pdfDoc.value.getPage(currentPage.value);
    const viewport = page.getViewport({ scale: zoom.value });
    const context = canvas.value.getContext('2d');
    if (!context) return;

    canvas.value.width = viewport.width;
    canvas.value.height = viewport.height;

    await page.render({ canvasContext: context, viewport }).promise;
}

async function runSearch() {
    if (!pdfDoc.value || !searchQuery.value.trim()) {
        matchedPages.value = [];
        currentMatchIndex.value = 0;
        return;
    }

    const query = searchQuery.value.trim().toLowerCase();
    const matches: number[] = [];

    for (let i = 1; i <= totalPages.value; i++) {
        const page = await pdfDoc.value.getPage(i);
        const content = await page.getTextContent();
        const text = content.items.map((item) => item.str || '').join(' ').toLowerCase();
        if (text.includes(query)) {
            matches.push(i);
        }
    }

    matchedPages.value = matches;
    currentMatchIndex.value = 0;

    if (matches.length > 0) {
        currentPage.value = matches[0];
        await renderPage();
    }
}

async function loadDocument() {
    loading.value = true;
    loadError.value = false;

    try {
        const pdfjsLib = await loadPdfJs();
        const doc = await pdfjsLib.getDocument(fallbackUrl.value).promise;
        pdfDoc.value = doc;
        totalPages.value = doc.numPages;
        currentPage.value = 1;
        zoom.value = 1;
        await renderPage();
        await runSearch();
    } catch (error) {
        loadError.value = true;
        pdfDoc.value = null;
        totalPages.value = 0;
    } finally {
        loading.value = false;
    }
}

async function nextPage() {
    if (currentPage.value >= totalPages.value) return;
    currentPage.value += 1;
    await renderPage();
}

async function previousPage() {
    if (currentPage.value <= 1) return;
    currentPage.value -= 1;
    await renderPage();
}

async function zoomIn() {
    zoom.value = Math.min(zoom.value + 0.1, 3);
    await renderPage();
}

async function zoomOut() {
    zoom.value = Math.max(zoom.value - 0.1, 0.5);
    await renderPage();
}

async function goToNextMatch() {
    if (matchedPages.value.length === 0) return;
    currentMatchIndex.value = (currentMatchIndex.value + 1) % matchedPages.value.length;
    currentPage.value = matchedPages.value[currentMatchIndex.value];
    await renderPage();
}

async function goToPreviousMatch() {
    if (matchedPages.value.length === 0) return;
    currentMatchIndex.value = (currentMatchIndex.value - 1 + matchedPages.value.length) % matchedPages.value.length;
    currentPage.value = matchedPages.value[currentMatchIndex.value];
    await renderPage();
}

watch(
    () => props.src,
    () => {
        loadDocument();
    },
    { immediate: true },
);
</script>

<template>
    <div class="pdf-preview">
        <div class="toolbar">
            <button type="button" class="btn" :disabled="currentPage <= 1 || loading" @click="previousPage">←</button>
            <span>{{ currentPage }} / {{ totalPages }}</span>
            <button type="button" class="btn" :disabled="currentPage >= totalPages || loading" @click="nextPage">→</button>

            <button type="button" class="btn" :disabled="loading" @click="zoomOut">-</button>
            <span>{{ Math.round(zoom * 100) }}%</span>
            <button type="button" class="btn" :disabled="loading" @click="zoomIn">+</button>

            <input v-model="searchQuery" type="text" placeholder="Rechercher dans le PDF" @keyup.enter="runSearch" />
            <button type="button" class="btn" :disabled="matchedPages.length === 0 || loading" @click="goToPreviousMatch">↑</button>
            <button type="button" class="btn" :disabled="matchedPages.length === 0 || loading" @click="goToNextMatch">↓</button>
            <span v-if="matchedPages.length > 0">{{ currentMatchIndex + 1 }} / {{ matchedPages.length }}</span>
            <button type="button" class="btn" :disabled="loading" @click="runSearch">Chercher</button>
        </div>

        <div v-if="loading" class="state">Chargement du PDF…</div>
        <div v-else-if="loadError" class="state">
            <p>Impossible de charger PDF.js, ouverture via fallback navigateur.</p>
            <a :href="fallbackUrl" target="_blank" rel="noopener noreferrer">Ouvrir le PDF</a>
        </div>
        <canvas v-else ref="canvas"></canvas>
    </div>
</template>

<style scoped lang="scss">
.pdf-preview {
    border: 1px solid var(--editor-blue-light, #dce8f4);
    border-radius: 4px;
    overflow: hidden;
}

.toolbar {
    align-items: center;
    background: var(--editor-bg-light, #f7f9fc);
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
    padding: 0.5rem;

    input {
        flex: 1;
        min-width: 180px;
    }
}

.btn {
    border: 1px solid var(--editor-blue-light, #dce8f4);
    border-radius: 3px;
    cursor: pointer;
    padding: 0.2rem 0.4rem;
}

.state {
    padding: 1rem;
}

canvas {
    display: block;
    margin: 0 auto;
    max-width: 100%;
}
</style>
