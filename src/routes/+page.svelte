<script>
    import { createWorker } from 'tesseract.js';
    import { onMount } from 'svelte';

    let fileInput;
    let originalImageUrl = $state(null);
    let processedImageUrl = $state(null); // ★ Tesseract가 보는 화면(디버그용)
    let wavePreviews = $state([]);
    let isLoading = $state(false);
    let statusMessage = $state('이미지를 업로드해주세요.');

    async function handleFileChange(event) {
        const file = event.target.files[0];
        if (!file || !file.type.startsWith('image/')) return;

        wavePreviews = [];
        isLoading = true;
        statusMessage = 'OCR 엔진 로딩 중...';

        if (originalImageUrl) URL.revokeObjectURL(originalImageUrl);
        originalImageUrl = URL.createObjectURL(file);
        processedImageUrl = null; // 초기화

        try {
            await processImageWithOcr(file);
        } catch (error) {
            console.error('이미지 처리 실패:', error);
            statusMessage = '⚠️ 분석 중 에러가 발생했습니다.';
            isLoading = false;
        }
    }

    async function processImageWithOcr(imageFile) {
        statusMessage = '엔진 초기화 중... (kor+eng)';
        
        // 1. 한국어+영어 동시 사용 (Wave를 못 읽으면 '턴'을 읽도록 보험)
        const worker = await createWorker('kor+eng', 1, {
            logger: m => {
                if (m.status === 'recognizing text') {
                    statusMessage = `텍스트 분석 중... (${Math.round(m.progress * 100)}%)`;
                }
            }
        });

        await worker.setParameters({
            tessedit_pageseg_mode: '11', // Sparse text
        });

        // 2. 안전한 전처리 (부드러운 흑백 반전)
        statusMessage = '이미지 전처리 중 (흑백 반전)...';
        
        const img = new Image();
        img.src = originalImageUrl;
        await img.decode();

        const prepCanvas = document.createElement('canvas');
        const pCtx = prepCanvas.getContext('2d');
        
        // 해상도 3배 확대
        const scale = 3; 
        prepCanvas.width = img.width * scale;
        prepCanvas.height = img.height * scale;
        
        // Tesseract 최신 모델(LSTM)은 안티앨리어싱을 좋아하므로 부드럽게 유지
        pCtx.imageSmoothingEnabled = true; 
        pCtx.drawImage(img, 0, 0, prepCanvas.width, prepCanvas.height);

        const imgData = pCtx.getImageData(0, 0, prepCanvas.width, prepCanvas.height);
        const dataArr = imgData.data;
        
        // 부드러운 흑백(Grayscale) 변환 후 색상 반전
        // 밝은 글자는 어둡게(검은색), 어두운 배경은 밝게(흰색) 만듭니다.
        for (let i = 0; i < dataArr.length; i += 4) {
            const r = dataArr[i];
            const g = dataArr[i + 1];
            const b = dataArr[i + 2];
            
            // 흑백 휘도 계산
            const gray = (r * 0.299 + g * 0.587 + b * 0.114);
            const invertedGray = 255 - gray; // 색상 반전

            dataArr[i] = invertedGray;
            dataArr[i + 1] = invertedGray;
            dataArr[i + 2] = invertedGray;
        }
        pCtx.putImageData(imgData, 0, 0);

        // ★ 우리가 화면에서 볼 수 있도록 상태 변수에 저장
        processedImageUrl = prepCanvas.toDataURL('image/png');

        // 3. 이미지 인식 수행
        const result = await worker.recognize(processedImageUrl);
        const data = result.data;

        statusMessage = '영역 계산 중...';

        const waveBlocks = [];
        const lines = data.lines || [];

        console.log("=== 드디어 읽어낸 텍스트 ===");
        lines.forEach((line) => {
            const text = line.text.trim()
            if(text.length > 0) {
                console.log(text); 
            }

            const match = text.match(/(wave|웨이브|턴|vave|wove)\s*(\d+)/i);

            if (match) {
                waveBlocks.push({
                    text: text,
                    waveNum: match[2],
                    y0: line.bbox.y0 / scale, 
                    y1: line.bbox.y1 / scale  
                });
            }
        });

        // 겹치는 위치 제거 (Y좌표 차이가 20픽셀 이내면 같은 줄로 간주)
        const uniqueWaveBlocks = [];
        waveBlocks.sort((a, b) => a.y0 - b.y0).forEach((block) => {
            if (uniqueWaveBlocks.length === 0) {
                uniqueWaveBlocks.push(block);
            } else {
                const lastBlock = uniqueWaveBlocks[uniqueWaveBlocks.length - 1];
                if (block.y0 - lastBlock.y0 > 20) {
                    uniqueWaveBlocks.push(block);
                }
            }
        });

        if (uniqueWaveBlocks.length === 0) {
            statusMessage = '⚠️ 텍스트를 찾지 못했습니다. 우측의 "Tesseract가 보는 화면"에 글자가 잘 보이는지 확인해주세요!';
            await worker.terminate();
            isLoading = false;
            return;
        }

        // 4. 원본 이미지를 Canvas에 로드하여 최종 영역 절삭
        const finalCanvas = document.createElement('canvas');
        const fCtx = finalCanvas.getContext('2d');
        const paddingAbove = 20;

        for (let i = 0; i < uniqueWaveBlocks.length; i++) {
            const currentWave = uniqueWaveBlocks[i];
            const nextWave = uniqueWaveBlocks[i + 1];

            const sy = Math.max(0, currentWave.y0 - paddingAbove);
            const ey = nextWave ? Math.max(0, nextWave.y0 - paddingAbove) : img.height;
            const cropHeight = ey - sy;

            if (cropHeight < 50) continue;

            finalCanvas.width = img.width;
            finalCanvas.height = cropHeight;
            fCtx.drawImage(img, 0, sy, img.width, cropHeight, 0, 0, img.width, cropHeight);

            wavePreviews.push({
                title: `Wave/Turn ${currentWave.waveNum} (인식: ${currentWave.text})`,
                dataUrl: finalCanvas.toDataURL('image/png')
            });
        }

        await worker.terminate();
        statusMessage = `✅ 완료: ${wavePreviews.length}개의 영역을 분리했습니다.`;
        isLoading = false;
    }
</script>

<div class="min-h-screen bg-gray-100 p-4 md:p-10">
    <div class="mx-auto max-w-5xl rounded-2xl bg-white p-6 shadow-lg">
        <h1 class="mb-2 text-3xl font-bold text-gray-800">
            FGO 커맨드 변환기 <span class="text-sm font-normal text-gray-500">(IMG 기반)</span>
        </h1>
        <div class="mb-6 text-gray-500">
            칼데아(Chaldea) 앱의 공유 스크린샷 이미지(IMG)를 FGA용 텍스트로 변환합니다.
        </div>

        <div class="flex flex-col items-center gap-4 rounded-2xl border-2 border-dashed border-gray-300 bg-gray-50 p-6">
            <input
                type="file"
                accept="image/*"
                bind:this={fileInput}
                onchange={handleFileChange}
                class="hidden"
            />

            {#if originalImageUrl}
                <div class="flex flex-col md:flex-row gap-4 w-full justify-center">
                    <div class="relative flex flex-col items-center">
                        <span class="mb-2 text-sm font-bold text-gray-600">원본 이미지</span>
                        <img src={originalImageUrl} alt="원본" class="max-h-80 rounded-xl border shadow-md object-contain" />
                        {#if isLoading}
                            <div class="absolute inset-0 top-7 flex items-center justify-center rounded-xl bg-white/70">
                                <span class="animate-pulse font-bold text-blue-600">분석 중...</span>
                            </div>
                        {/if}
                    </div>

                    {#if processedImageUrl}
                        <div class="relative flex flex-col items-center">
                            <span class="mb-2 text-sm font-bold text-red-500">Tesseract가 읽는 화면 (디버그)</span>
                            <img src={processedImageUrl} alt="전처리" class="max-h-80 rounded-xl border-2 border-red-300 shadow-md object-contain" />
                        </div>
                    {/if}
                </div>
            {/if}

            <button
                onclick={() => fileInput.click()}
                class="mt-4 flex w-full max-w-md items-center justify-center gap-2 rounded-xl bg-blue-600 py-4 font-bold text-white transition-colors hover:bg-blue-700 active:bg-blue-900"
                disabled={isLoading}
            >
                {#if isLoading}
                    ⏳ 처리 중...
                {:else}
                    📷 스크린샷 업로드 및 분석
                {/if}
            </button>
            <p class="text-sm {isLoading ? 'font-medium text-blue-600' : 'text-gray-500'}">
                {statusMessage}
            </p>
        </div>

        {#if wavePreviews.length > 0}
            <div class="mt-10 space-y-8">
                <div class="flex items-center justify-between border-b pb-3">
                    <h2 class="text-2xl font-bold text-gray-800">웨이브 영역 미리보기</h2>
                </div>

                <div class="grid grid-cols-1 gap-6 lg:grid-cols-2">
                    {#each wavePreviews as preview, index}
                        <div class="rounded-2xl border border-gray-700 bg-gray-900 p-5 shadow-inner">
                            <div class="mb-3 flex items-center justify-between">
                                <h3 class="font-mono text-sm font-bold tracking-wider text-green-400">
                                    {preview.title}
                                </h3>
                            </div>
                            <div class="max-h-96 overflow-auto rounded-lg border border-gray-800 bg-black">
                                <img src={preview.dataUrl} alt={preview.title} class="w-full object-contain" />
                            </div>
                        </div>
                    {/each}
                </div>
            </div>
        {/if}
    </div>
</div>