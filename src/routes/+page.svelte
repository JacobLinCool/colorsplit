<script lang="ts">
	import { onMount } from 'svelte';
	import * as pdfjsLib from 'pdfjs-dist';
	import { PDFDocument } from 'pdf-lib';
	import { m } from '$lib/paraglide/messages.js';

	// State variables
	let files: FileList | null = $state(null);
	let colorThreshold = $state(50);
	let insertBlankPages = $state(false);
	let processing = $state(false);
	let progress = $state('');
	let results: { colorPages: number; bwPages: number } = $state({ colorPages: 0, bwPages: 0 });

	onMount(async () => {
		// Set up PDF.js worker
		pdfjsLib.GlobalWorkerOptions.workerSrc = `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/${pdfjsLib.version}/pdf.worker.min.mjs`;
	});

	// Improved color detection function with luminance-aware processing
	function detectColor(imageData: ImageData, threshold: number): boolean {
		const data = imageData.data;
		let colorPixels = 0;
		let sampledPixels = 0;
		let brightPixels = 0;
		let darkPixels = 0;

		// Sample every 10th pixel for performance
		for (let i = 0; i < data.length; i += 40) {
			// 40 = 4 * 10 (RGBA * skip)
			const r = data[i];
			const g = data[i + 1];
			const b = data[i + 2];
			const a = data[i + 3];

			// Skip transparent pixels
			if (a < 128) continue;

			sampledPixels++;

			// Calculate luminance using standard perceptual weights
			const luminance = 0.299 * r + 0.587 * g + 0.114 * b;
			
			// Track brightness distribution for adaptive processing
			if (luminance > 200) brightPixels++;
			else if (luminance < 55) darkPixels++;

			// Check if pixel has significant color variation
			const max = Math.max(r, g, b);
			const min = Math.min(r, g, b);
			const saturation = max === 0 ? 0 : (max - min) / max;

			// Calculate chroma (absolute color intensity)
			const chroma = max - min;
			
			// Adaptive color detection based on luminance conditions
			let isColorPixel = false;
			
			if (luminance < 55) {
				// Low light conditions: be more sensitive to any color variation
				// Use lower threshold and consider both saturation and chroma
				isColorPixel = (saturation * 100 > threshold * 0.7) || (chroma > 8);
			} else if (luminance > 200) {
				// Bright conditions: be more strict but consider subtle colors
				// Bright areas might wash out colors, so check both metrics
				isColorPixel = (saturation * 100 > threshold * 0.8) || (chroma > 10);
			} else {
				// Normal conditions: use standard saturation with chroma backup
				isColorPixel = (saturation * 100 > threshold) || 
							  (chroma > 12 && saturation > 0.1);
			}

			if (isColorPixel) {
				colorPixels++;
			}
		}

		if (sampledPixels === 0) return false;

		const colorPercentage = (colorPixels / sampledPixels) * 100;
		const brightRatio = brightPixels / sampledPixels;
		const darkRatio = darkPixels / sampledPixels;
		
		// Adaptive final threshold based on lighting conditions
		let finalThreshold = 0.01; // Base threshold
		
		// In very bright or very dark images, lower the threshold
		// as color detection becomes more challenging
		if (brightRatio > 0.3 || darkRatio > 0.3) {
			finalThreshold = 0.005; // More sensitive in challenging lighting
		}
		
		return colorPercentage > finalThreshold;
	}

	// Process PDFs
	async function processPDFs() {
		if (!files || files.length === 0) return;

		processing = true;
		progress = m.progress_starting();
		results = { colorPages: 0, bwPages: 0 };

		const colorPDF = await PDFDocument.create();
		const bwPDF = await PDFDocument.create();

		try {
			for (let fileIndex = 0; fileIndex < files.length; fileIndex++) {
				const file = files[fileIndex];
				progress = m.progress_processing_file({
					filename: file.name,
					current: fileIndex + 1,
					total: files.length
				});

				let arrayBuffer: ArrayBuffer;
				try {
					arrayBuffer = await file.arrayBuffer();
				} catch (error) {
					console.error(`Error reading file ${file.name}:`, error);
					progress = m.error_reading_file({ filename: file.name });
					continue;
				}

				let sourcePDF: PDFDocument;
				try {
					sourcePDF = await PDFDocument.load(arrayBuffer);
				} catch (error) {
					console.error(`Error loading PDF ${file.name}:`, error);
					progress = m.error_invalid_pdf({ filename: file.name });
					continue;
				}

				const pageCount = sourcePDF.getPageCount();

				// Load PDF with pdf.js for rendering
				let pdfDoc: any;
				try {
					const loadingTask = pdfjsLib.getDocument(arrayBuffer);
					pdfDoc = await loadingTask.promise;
				} catch (error) {
					console.error(`Error loading PDF with pdf.js ${file.name}:`, error);
					progress = m.error_processing_file({ filename: file.name });
					continue;
				}

				// Add blank page between documents if option is enabled
				if (insertBlankPages && fileIndex > 0) {
					colorPDF.addPage();
					bwPDF.addPage();
				}

				for (let pageIndex = 0; pageIndex < pageCount; pageIndex++) {
					progress = m.progress_processing_page({
						filename: file.name,
						current: pageIndex + 1,
						total: pageCount
					});

					let isColorPage = false;
					try {
						// Render page to canvas for color detection
						const page = await pdfDoc.getPage(pageIndex + 1);
						const viewport = page.getViewport({ scale: 0.2 }); // Lower scale for faster processing, and larger color area

						const canvas = document.createElement('canvas');
						const context = canvas.getContext('2d')!;
						canvas.height = viewport.height;
						canvas.width = viewport.width;

						await page.render({
							canvasContext: context,
							viewport: viewport
						}).promise;

						const imageData = context.getImageData(0, 0, canvas.width, canvas.height);
						isColorPage = detectColor(imageData, colorThreshold);
					} catch (error) {
						console.error(`Error processing page ${pageIndex + 1} of ${file.name}:`, error);
						// Default to B&W if color detection fails
						isColorPage = false;
					}

					try {
						// Copy page to appropriate PDF
						const [copiedPage] = await (isColorPage ? colorPDF : bwPDF).copyPages(sourcePDF, [
							pageIndex
						]);
						(isColorPage ? colorPDF : bwPDF).addPage(copiedPage);

						if (isColorPage) {
							results.colorPages++;
						} else {
							results.bwPages++;
						}
					} catch (error) {
						console.error(`Error copying page ${pageIndex + 1} of ${file.name}:`, error);
						progress = m.error_copying_page({ page: pageIndex + 1, filename: file.name });
						continue;
					}
				}
			}

			progress = m.progress_generating();

			// Generate and download PDFs
			let colorPdfBytes: Uint8Array | null = null;
			let bwPdfBytes: Uint8Array | null = null;

			try {
				if (results.colorPages > 0) {
					colorPdfBytes = await colorPDF.save();
				}
				if (results.bwPages > 0) {
					bwPdfBytes = await bwPDF.save();
				}
			} catch (error) {
				console.error('Error generating PDFs:', error);
				progress = m.error_generating_pdfs();
				return;
			}

			// Download color PDF
			if (colorPdfBytes && results.colorPages > 0) {
				try {
					const colorBlob = new Blob([colorPdfBytes], { type: 'application/pdf' });
					const colorUrl = URL.createObjectURL(colorBlob);
					const colorLink = document.createElement('a');
					colorLink.href = colorUrl;
					colorLink.download = 'color_pages.pdf';
					colorLink.click();
					URL.revokeObjectURL(colorUrl);
				} catch (error) {
					console.error('Error downloading color PDF:', error);
					progress = m.error_downloading_color();
				}
			}

			// Download B&W PDF
			if (bwPdfBytes && results.bwPages > 0) {
				try {
					const bwBlob = new Blob([bwPdfBytes], { type: 'application/pdf' });
					const bwUrl = URL.createObjectURL(bwBlob);
					const bwLink = document.createElement('a');
					bwLink.href = bwUrl;
					bwLink.download = 'bw_pages.pdf';
					bwLink.click();
					URL.revokeObjectURL(bwUrl);
				} catch (error) {
					console.error('Error downloading B&W PDF:', error);
					progress = m.error_downloading_bw();
				}
			}

			if (results.colorPages === 0 && results.bwPages === 0) {
				progress = m.error_no_pages();
			} else {
				progress = m.progress_complete();
			}
		} catch (error) {
			console.error('Unexpected error during PDF processing:', error);
			progress = m.error_unexpected();
		} finally {
			processing = false;
		}
	}

	function handleFileChange(event: Event) {
		const target = event.target as HTMLInputElement;
		files = target.files;
	}
</script>

<div class="min-h-screen bg-gray-50 py-8">
	<div class="mx-auto max-w-4xl px-4">
		<div class="rounded-lg bg-white p-8 shadow-lg">
			<h1 class="mb-8 text-3xl font-bold text-gray-900">{m.app_title()}</h1>
			<p class="mb-8 text-gray-600">
				{m.app_description()}
			</p>

			<!-- File Upload -->
			<div class="mb-8">
				<label for="pdf-files" class="mb-2 block text-sm font-medium text-gray-700">
					{m.select_pdf_files()}
				</label>
				<input
					type="file"
					id="pdf-files"
					multiple
					accept=".pdf"
					onchange={handleFileChange}
					class="block w-full rounded-md border border-gray-300 px-3 py-2 text-sm file:mr-4 file:rounded-md file:border-0 file:bg-blue-50 file:px-4 file:py-2 file:text-sm file:font-medium file:text-blue-700 hover:file:bg-blue-100 focus:border-blue-500 focus:ring-1 focus:ring-blue-500 focus:outline-none"
				/>
				{#if files && files.length > 0}
					<p class="mt-2 text-sm text-gray-600">
						{m.files_selected({ count: files.length, plural: files.length > 1 ? 's' : '' })}
					</p>
				{/if}
			</div>

			<!-- Options -->
			<div class="mb-8 grid gap-6 md:grid-cols-2">
				<div>
					<label for="threshold" class="mb-2 block text-sm font-medium text-gray-700">
						{m.color_threshold({ value: colorThreshold })}
					</label>
					<input
						type="range"
						id="threshold"
						min="10"
						max="100"
						bind:value={colorThreshold}
						class="w-full"
					/>
					<div class="mt-1 flex justify-between text-xs text-gray-500">
						<span>{m.less_sensitive()}</span>
						<span>{m.more_sensitive()}</span>
					</div>
				</div>

				<div class="flex items-center">
					<input
						type="checkbox"
						id="blank-pages"
						bind:checked={insertBlankPages}
						class="h-4 w-4 rounded border-gray-300 text-blue-600 focus:ring-blue-500"
					/>
					<label for="blank-pages" class="ml-2 text-sm text-gray-700">
						{m.insert_blank_pages()}
					</label>
				</div>
			</div>

			<!-- Process Button -->
			<button
				onclick={processPDFs}
				disabled={!files || files.length === 0 || processing}
				class="w-full rounded-md bg-blue-600 px-4 py-3 font-medium text-white hover:bg-blue-700 focus:ring-2 focus:ring-blue-500 focus:ring-offset-2 focus:outline-none disabled:cursor-not-allowed disabled:bg-gray-300"
			>
				{processing ? m.processing() : m.split_pdfs()}
			</button>

			<!-- Progress -->
			{#if processing || progress}
				<div class="mt-6">
					<div class="rounded-md bg-blue-50 p-4">
						<div class="flex items-center space-x-3">
							{#if processing}
								<div class="h-5 w-5">
									<svg
										class="h-5 w-5 animate-spin text-blue-600"
										xmlns="http://www.w3.org/2000/svg"
										fill="none"
										viewBox="0 0 24 24"
									>
										<circle
											class="opacity-25"
											cx="12"
											cy="12"
											r="10"
											stroke="currentColor"
											stroke-width="4"
										></circle>
										<path
											class="opacity-75"
											fill="currentColor"
											d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
										></path>
									</svg>
								</div>
							{:else}
								<div class="h-5 w-5">
									<svg
										class="h-5 w-5 text-blue-600"
										xmlns="http://www.w3.org/2000/svg"
										fill="none"
										viewBox="0 0 24 24"
										stroke="currentColor"
									>
										<path
											stroke-linecap="round"
											stroke-linejoin="round"
											stroke-width="2"
											d="M5 13l4 4L19 7"
										/>
									</svg>
								</div>
							{/if}
							<p class="flex-1 text-sm text-blue-800">{progress}</p>
						</div>
						{#if processing}
							<div class="mt-3">
								<div class="h-2 w-full rounded-full bg-blue-200">
									<div
										class="h-2 rounded-full bg-blue-600 transition-all duration-300"
										style="width: {Math.min((results.colorPages + results.bwPages) * 10, 100)}%"
									></div>
								</div>
							</div>
						{/if}
					</div>
				</div>
			{/if}

			<!-- Results -->
			{#if results.colorPages > 0 || results.bwPages > 0}
				<div class="mt-6 grid gap-4 md:grid-cols-2">
					<div class="rounded-md bg-green-50 p-4">
						<h3 class="text-lg font-medium text-green-800">{m.results()}</h3>
						<p class="text-sm text-green-700">{m.color_pages({ count: results.colorPages })}</p>
						<p class="text-sm text-green-700">{m.bw_pages({ count: results.bwPages })}</p>
					</div>
					<div class="rounded-md bg-yellow-50 p-4">
						<h3 class="text-lg font-medium text-yellow-800">{m.downloads()}</h3>
						<p class="text-sm text-yellow-700">
							{results.colorPages > 0 ? m.color_pdf_downloaded() : m.no_color_pages()}
						</p>
						<p class="text-sm text-yellow-700">
							{results.bwPages > 0 ? m.bw_pdf_downloaded() : m.no_bw_pages()}
						</p>
					</div>
				</div>
			{/if}

			<!-- Instructions -->
			<div class="mt-8 space-y-6">
				<div class="rounded-md bg-gray-50 p-4">
					<h3 class="mb-3 text-lg font-medium text-gray-900">{m.how_it_works()}</h3>
					<ul class="space-y-2 text-sm text-gray-700">
						<li class="flex items-start space-x-2">
							<span
								class="mt-0.5 flex h-5 w-5 flex-shrink-0 items-center justify-center rounded-full bg-blue-100 text-xs font-medium text-blue-800"
								>1</span
							>
							<span>{m.step_1()}</span>
						</li>
						<li class="flex items-start space-x-2">
							<span
								class="mt-0.5 flex h-5 w-5 flex-shrink-0 items-center justify-center rounded-full bg-blue-100 text-xs font-medium text-blue-800"
								>2</span
							>
							<span>{m.step_2()}</span>
						</li>
						<li class="flex items-start space-x-2">
							<span
								class="mt-0.5 flex h-5 w-5 flex-shrink-0 items-center justify-center rounded-full bg-blue-100 text-xs font-medium text-blue-800"
								>3</span
							>
							<span>{m.step_3()}</span>
						</li>
						<li class="flex items-start space-x-2">
							<span
								class="mt-0.5 flex h-5 w-5 flex-shrink-0 items-center justify-center rounded-full bg-blue-100 text-xs font-medium text-blue-800"
								>4</span
							>
							<span>{m.step_4()}</span>
						</li>
						<li class="flex items-start space-x-2">
							<span
								class="mt-0.5 flex h-5 w-5 flex-shrink-0 items-center justify-center rounded-full bg-blue-100 text-xs font-medium text-blue-800"
								>5</span
							>
							<span>{m.step_5()}</span>
						</li>
					</ul>
				</div>

				<div class="rounded-md bg-green-50 p-4">
					<h3 class="mb-2 text-lg font-medium text-green-900">{m.privacy_security()}</h3>
					<p class="text-sm text-green-800">
						{m.privacy_local()}<br />
						{m.privacy_no_upload()}<br />
						{m.privacy_offline()}
					</p>
				</div>

				<div class="rounded-md bg-amber-50 p-4">
					<h3 class="mb-2 text-lg font-medium text-amber-900">{m.tips_title()}</h3>
					<ul class="space-y-1 text-sm text-amber-800">
						<li>{m.tip_subtle()}</li>
						<li>{m.tip_bold()}</li>
						<li>{m.tip_large()}</li>
						<li>{m.tip_incorrect()}</li>
					</ul>
				</div>
			</div>
		</div>
	</div>
</div>
