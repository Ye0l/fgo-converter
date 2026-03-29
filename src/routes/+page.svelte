<script>
	import pako from 'pako';

	let url = $state(
		'https://link.chaldea.center/laplace/share?data=GH4sIANpNyWkA_9VXUW_aMBD-L37OJDsECLytbNWQ2MvGnipUuclBLBwnjR00VPW_784JFJVC2dRGnYRwztx3392X8yU8sFyZq1rplI1HvV7AZFnuzDgO2H0N1rHxA1O0E3Hc5CJgZSYtsDFegYF8-03ajI2ZuOUx57dxlPAkEewxYEXpVGEsBcjVKnPbSSaVYWNX1RCwVFl5p-HrBgxyLKW2uFkWyrirerlElKm1DphVea2lg89q7-RZf-Der7LxQi4HMiciI3PYQfOtdSqZFCn4FPbWFKsJBQ-Yhg1oLIRTsuZagU5_bhxS3zwwu3HkN-QR5-SqcuUmRU25RpjVWmk9Tck1Ho6iPg-Gg7CHCyrU7_NF6zHbkAdS-Q_uunQfVJA123gdNS4jZJFufV3UlBFxZuWBkfi08SbwXuTNGWV0VYFc7xTFPQrXR4Xua1WKgzvXgPQRRHsAlm_rsiwqN9-WKBZbVgpMygK2qqQhTVrtH4P_TJi2Y1plLhKmhXhljoQxhYGzsghManRel0E8QCWCQTyM_ILma7o0UT9yx5wQBiu4k8m6Lp-fK6xI9M_rFIrhiIQKhW8dWkX_Eqkw8F9LdbJp-HNtmtlyomP4xR3jw9DXAiEpaFjhQCMOlxISR6VZga8QS6ug1DKB75DfQTU1Kfz2P92IgC8IL5N20qLAruH1GiGxJU7eauavDsYyRt1CNZfVCtArbEfrzkZfyrrIST80XFaBzQp6PJBsj9T259lEp2xhB2xhp0qKi9jEm7OJTtnC92GTzuHoYUF7sR8_Pkgiq9QPHGXnX56OfKXwHUHqpw10aw5y2A75A3R4OVocoz-JHbydxee4F11LJF4vsk37hRr3YH4xuHcM_ggCnTj7Yads730au6npXefY6UYO_6GdxNu204KIJ0g3939SmheAP2JGBAd8DQAA&questId=94098801&phase=1&enemyHash=1_0800_84c0cc1'
	);
	let decodedData = $state(null);
	let fgaCommand = $state('');
	let isError = $state(false);

	function handleConvert() {
		try {
			isError = false;
			const urlObj = new URL(url.trim());
			let dataParam = urlObj.searchParams.get('data');

			if (!dataParam) {
				isError = true;
				decodedData = null;
				return;
			}

			if (dataParam.startsWith('G') || dataParam.startsWith('D') || dataParam.startsWith('Z')) {
				dataParam = dataParam.substring(1);
			}

			let base64 = dataParam.replace(/-/g, '+').replace(/_/g, '/');

			while (base64.length % 4 !== 0) {
				base64 += '=';
			}

			const binaryString = atob(base64);
			const len = binaryString.length;
			const bytes = new Uint8Array(len);
			for (let i = 0; i < len; i++) {
				bytes[i] = binaryString.charCodeAt(i);
			}
			const decompressed = pako.inflate(bytes);
			const jsonString = new TextDecoder().decode(new Uint8Array(decompressed));

			decodedData = JSON.parse(jsonString);
			fgaCommand = convertToFga(decodedData.actions);
		} catch (err) {
			console.error('handleConvert:', err);
			isError = true;
			decodedData = null;
		}
	}

	function copyToClipboard() {
		if (!decodedData?.actions) return;

		// 텍스트로 변환 (들여쓰기 2칸 적용)
		const text = JSON.stringify(decodedData.actions, null, 2);

		// 브라우저 클립보드 API 호출
		navigator.clipboard
			.writeText(text)
			.then(() => {
				alert('데이터가 복사되었습니다');
			})
			.catch((err) => {
				console.error('복사 실패:', err);
			});
	}

	function convertToFga(actions) {
		if (!actions) return '';

		let result = '';

		actions.forEach((action) => {
			if (action.type === 'skill') {
				// 마스터 스킬 (svt가 undefined인 경우)
				if (action.svt === undefined) {
					const masterSkills = ['j', 'k', 'l'];
					result += masterSkills[action.skill];
				}
				// 서번트 스킬
				else {
					// svt 0: a,b,c / svt 1: d,e,f / svt 2: g,h,i
					const skillMap = [
						['a', 'b', 'c'], // svt 0
						['d', 'e', 'f'], // svt 1
						['g', 'h', 'i'] // svt 2
					];
					result += skillMap[action.svt][action.skill];
				}

				// 타겟팅 정보가 있다면 추가 (FGA는 스킬 뒤에 숫자를 붙임. 예: a1)
				// playerTarget은 0,1,2로 오므로 FGA용 1,2,3으로 변환
				if (action.options?.playerTarget !== undefined) {
					result += action.options.playerTarget + 1;
				}
			} else if (action.type === 'attack') {
				// 보구 사용 여부 확인
				action.attacks.forEach((atk) => {
					if (atk.isTD) {
						// svt 0: 4 / svt 1: 5 / svt 2: 6
						result += atk.svt + 4;
					}
				});
				// 턴 종료 문자 추가
				result += ',#,';
			}
		});

		return result;
	}
</script>

<div class="min-h-screen bg-gray-100 p-10 md:p-10">
	<div class="mx-auto max-w-3xl rounded-2xl bg-white p-5 shadow-lg">
		<h1 class="mb-2 text-3xl font-bold text-gray-800">FGO 커맨드 변환기</h1>
		<div class="text-2x1 mb-6 text-gray-500">
			칼데아(Chaldea) 앱의 공유 URL을 FGA용 텍스트로 변환합니다.
		</div>

		<div class="flex flex-col gap-3">
			<input
				type="text"
				bind:value={url}
				placeholder="https://link.chaldea.center/laplace/share?data=..."
				class="w-full rounded-lg border border-gray-300 p-3 outline-none focus:ring-2 focus:ring-blue-500"
			/>
			<button
				onclick={handleConvert}
				class="w-full rounded-lg bg-blue-600 py-3 font-bold text-white transition-colors hover:bg-blue-700 active:bg-blue-900"
			>
				변환하기
			</button>
		</div>

		{#if isError}
			<div class="mt-4 rounded-lg bg-red-50 p-4 text-red-600">⚠️ 올바른 링크인지 확인해주세요.</div>
		{/if}

		{#if decodedData}
			<div class="mt-4 space-y-4">
				<div class="relative rounded-lg bg-gray-900 p-4">
					<div class="mb-2 flex items-center justify-between">
						<h3 class="text-sm font-bold text-gray-400">압축 해제된 전체 JSON 데이터</h3>

						<button
							onclick={copyToClipboard}
							class="rounded bg-gray-700 px-2 py-1 text-xs text-gray-200 transition-colors hover:bg-gray-600"
						>
							복사하기 📑
						</button>
					</div>

					<div class="max-h-1600 overflow-auto text-xs text-green-400">
						<pre>{JSON.stringify(decodedData?.actions, null, 2)}</pre>
					</div>
				</div>
			</div>
		{/if}
		{#if fgaCommand}
			<div class="mt-6 rounded-xl border-2 border-blue-200 bg-blue-50 p-5">
				<h2 class="mb-2 text-lg font-bold text-blue-800">FGA 커맨드 결과 🚀</h2>
				<div class="flex items-center gap-2">
					<code class="flex-1 rounded border bg-white p-3 font-mono break-all text-blue-600">
						{fgaCommand}
					</code>
					<button
						onclick={() => {
							navigator.clipboard.writeText(fgaCommand);
							alert('커맨드가 복사되었습니다!');
						}}
						class="rounded-lg bg-blue-600 px-4 py-3 font-bold text-white hover:bg-blue-700"
					>
						복사
					</button>
				</div>
			</div>
		{/if}
	</div>
</div>

<style>
</style>
