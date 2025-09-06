<script lang="ts">
	import Papa from 'papaparse';
	import _ from 'lodash';

	// Types
	interface CSVRow {
		[key: string]: string | number | null;
	}

	interface FieldMappings {
		[standardField: string]: string[];
	}

	// State using Svelte 5 runes
	let file = $state<File | null>(null);
	let originalData = $state<CSVRow[]>([]);
	let processedData = $state<CSVRow[]>([]);
	let fieldMappings = $state<FieldMappings>({});
	let processing = $state(false);
	let step = $state<'upload' | 'mapping' | 'preview'>('upload');
	let fileInput = $state<HTMLInputElement>();

	// All accepted CRM fields with their pattern variations
	const fieldPatterns: { [key: string]: RegExp[] } = {
		'First Name': [
			/^(first.?name|fname|contact.?first|given.?name|contact[_\s]?[0-9]*[_\s]?first)$/i
		],
		'Last Name': [
			/^(last.?name|lname|contact.?last|surname|family.?name|contact[_\s]?[0-9]*[_\s]?last)$/i
		],
		Company: [/^(company|organization|org|business|employer|contact[_\s]?[0-9]*[_\s]?company)$/i],
		'Job Title': [/^(job.?title|title|position|role|occupation|contact[_\s]?[0-9]*[_\s]?title)$/i],
		'Home Phone': [
			/^(home.?phone|home.?tel|residential.?phone|contact[_\s]?[0-9]*[_\s]?home.?phone)$/i
		],
		'Business Phone': [
			/^(business.?phone|work.?phone|office.?phone|company.?phone|contact[_\s]?[0-9]*[_\s]?business.?phone)$/i
		],
		'Mobile Phone': [
			/^(mobile|cell|cellular|mobile.?phone|cell.?phone|contact[_\s]?[0-9]*[_\s]?mobile)$/i
		],
		'Home Fax': [/^(home.?fax|residential.?fax|contact[_\s]?[0-9]*[_\s]?home.?fax)$/i],
		'E-mail Address': [
			/^(email|e.?mail|email.?address|contact.?email|contact[_\s]?[0-9]*[_\s]?email)$/i
		],
		'Web Page': [/^(website|web.?page|url|web.?site|homepage|contact[_\s]?[0-9]*[_\s]?website)$/i],
		Birthday: [/^(birthday|birth.?date|dob|date.?of.?birth|contact[_\s]?[0-9]*[_\s]?birthday)$/i],
		Anniversary: [/^(anniversary|wedding.?anniversary|contact[_\s]?[0-9]*[_\s]?anniversary)$/i],
		'Home Sale Anniversary': [
			/^(home.?sale.?anniversary|purchase.?date|contact[_\s]?[0-9]*[_\s]?home.?sale)$/i
		],
		Spouse: [/^(spouse|partner|husband|wife|contact[_\s]?[0-9]*[_\s]?spouse)$/i],
		'Spouse Business Phone': [
			/^(spouse.?business.?phone|spouse.?work.?phone|partner.?business.?phone)$/i
		],
		'Spouse Mobile Phone': [/^(spouse.?mobile|spouse.?cell|partner.?mobile|partner.?cell)$/i],
		'Spouse Email Address': [/^(spouse.?email|spouse.?e.?mail|partner.?email)$/i],
		'Spouse Birthday': [/^(spouse.?birthday|spouse.?birth.?date|partner.?birthday)$/i],
		Categories: [/^(categories|category|tags|groups|contact[_\s]?[0-9]*[_\s]?category)$/i],
		Source: [/^(source|lead.?source|referral|contact[_\s]?[0-9]*[_\s]?source)$/i],
		'Home Street': [
			/^(home.?street|home.?address|residential.?street|contact[_\s]?[0-9]*[_\s]?home.?street)$/i
		],
		Suite: [/^(suite|apt|apartment|unit|contact[_\s]?[0-9]*[_\s]?suite)$/i],
		'Home Address PO Box': [/^(home.?po.?box|home.?box|residential.?po.?box)$/i],
		'Home City': [/^(home.?city|residential.?city|contact[_\s]?[0-9]*[_\s]?home.?city)$/i],
		'Home State': [
			/^(home.?state|residential.?state|home.?province|contact[_\s]?[0-9]*[_\s]?home.?state)$/i
		],
		'Home Postal Code': [
			/^(home.?zip|home.?postal|residential.?zip|contact[_\s]?[0-9]*[_\s]?home.?zip)$/i
		],
		County: [/^(county|region|contact[_\s]?[0-9]*[_\s]?county)$/i],
		'Business Street': [/^(business.?street|work.?street|office.?street|company.?street)$/i],
		'Business City': [/^(business.?city|work.?city|office.?city|company.?city)$/i],
		'Business State': [/^(business.?state|work.?state|office.?state|company.?state)$/i],
		'Business Postal Code': [/^(business.?zip|work.?zip|office.?zip|company.?zip)$/i],
		'E-mail 3 Address': [/^(email.?3|third.?email|e.?mail.?3|contact[_\s]?[0-9]*[_\s]?email.?3)$/i],
		'Other E-mail': [/^(other.?email|alt.?email|alternative.?email|secondary.?email)$/i],
		'Extra E-mails': [/^(extra.?emails|additional.?emails|more.?emails)$/i],
		'Other Street': [/^(other.?street|alt.?street|alternative.?street|mailing.?street)$/i],
		'Other City': [/^(other.?city|alt.?city|alternative.?city|mailing.?city)$/i],
		'Other State': [/^(other.?state|alt.?state|alternative.?state|mailing.?state)$/i],
		'Other Postal Code': [/^(other.?zip|alt.?zip|alternative.?zip|mailing.?zip)$/i],
		'Other Phone': [/^(other.?phone|alt.?phone|alternative.?phone|additional.?phone)$/i],
		'Business Fax': [/^(business.?fax|work.?fax|office.?fax|company.?fax)$/i],
		'Child 1': [/^(child.?1|first.?child|kid.?1|son.?1|daughter.?1)$/i],
		'Child 1 Birthday': [/^(child.?1.?birthday|first.?child.?birthday|kid.?1.?birthday)$/i],
		'Child 2': [/^(child.?2|second.?child|kid.?2|son.?2|daughter.?2)$/i],
		'Child 2 Birthday': [/^(child.?2.?birthday|second.?child.?birthday|kid.?2.?birthday)$/i],
		'Child 3': [/^(child.?3|third.?child|kid.?3|son.?3|daughter.?3)$/i],
		'Child 3 Birthday': [/^(child.?3.?birthday|third.?child.?birthday|kid.?3.?birthday)$/i],
		'Child 4': [/^(child.?4|fourth.?child|kid.?4|son.?4|daughter.?4)$/i],
		'Child 4 Birthday': [/^(child.?4.?birthday|fourth.?child.?birthday|kid.?4.?birthday)$/i],
		'Child 5': [/^(child.?5|fifth.?child|kid.?5|son.?5|daughter.?5)$/i],
		'Child 5 Birthday': [/^(child.?5.?birthday|fifth.?child.?birthday|kid.?5.?birthday)$/i],
		Country: [/^(country|nation|contact[_\s]?[0-9]*[_\s]?country)$/i],
		Subdivision: [
			/^(subdivision|district|area|neighborhood|contact[_\s]?[0-9]*[_\s]?subdivision)$/i
		],
		'Last Contact Date': [/^(last.?contact|last.?contacted|contact.?date|last.?call)$/i],
		Twitter: [/^(twitter|@|social|contact[_\s]?[0-9]*[_\s]?twitter)$/i],
		Rank: [/^(rank|priority|level|grade|contact[_\s]?[0-9]*[_\s]?rank)$/i],
		'Contact Status': [/^(status|contact.?status|lead.?status|customer.?status)$/i],
		Notes: [/^(notes|comments|remarks|description|contact[_\s]?[0-9]*[_\s]?notes)$/i],
		'Date Met': [/^(date.?met|first.?met|meeting.?date|introduction.?date)$/i],
		'Assigned To': [/^(assigned.?to|owner|salesperson|agent|rep)$/i],
		'Important Dates': [/^(important.?dates|key.?dates|special.?dates)$/i],
		'Extra Details': [/^(extra.?details|additional.?info|misc|miscellaneous)$/i],
		'Custom Fields': [/^(custom|custom.?fields|additional.?fields|extra.?fields)$/i]
	};

	// Derived values using Svelte 5 runes
	const stepIndex = $derived({ upload: 0, mapping: 1, preview: 2 }[step]);

	function detectFieldMapping(headers: string[]): FieldMappings {
		const mappings: FieldMappings = {};
		const standardFields = Object.keys(fieldPatterns);

		headers.forEach((header) => {
			const cleanHeader = header.trim();

			// First check for exact matches (for perfect CRM exports)
			if (standardFields.includes(cleanHeader)) {
				if (!mappings[cleanHeader]) {
					mappings[cleanHeader] = [];
				}
				mappings[cleanHeader].push(cleanHeader);
				return;
			}

			// Then check pattern matching for messy data
			Object.entries(fieldPatterns).forEach(([standardField, patterns]) => {
				patterns.forEach((pattern) => {
					if (pattern.test(cleanHeader)) {
						if (!mappings[standardField]) {
							mappings[standardField] = [];
						}
						if (!mappings[standardField].includes(cleanHeader)) {
							mappings[standardField].push(cleanHeader);
						}
					}
				});
			});
		});

		return mappings;
	}

	function processContacts(data: CSVRow[]): CSVRow[] {
		const processedRows: CSVRow[] = [];

		data.forEach((row) => {
			// Process each row as a single contact record - don't separate spouses!
			const cleanedContact: CSVRow = {};

			// Map all fields directly - keep spouse data as spouse fields
			Object.entries(fieldMappings).forEach(([standardField, sourceFields]) => {
				sourceFields.forEach((sourceField) => {
					const value = row[sourceField];
					if (value !== null && value !== undefined && value !== '') {
						// Use the exact standard field name from your CRM
						cleanedContact[standardField] = value;
					}
				});
			});

			// Only add if we have meaningful contact data
			if (Object.keys(cleanedContact).length > 0) {
				processedRows.push(cleanedContact);
			}
		});

		return processedRows;
	}

	function handleFileUpload(event: Event) {
		const target = event.target as HTMLInputElement;
		const uploadedFile = target.files?.[0];

		if (!uploadedFile) {
			console.log('No file selected');
			return;
		}

		console.log('File selected:', uploadedFile.name);
		file = uploadedFile;
		processing = true;

		Papa.parse(uploadedFile, {
			header: true,
			skipEmptyLines: true,
			transformHeader: (header: string) => header.trim(),
			complete: (results) => {
				console.log('CSV parsed successfully:', results.data.length, 'rows');
				originalData = results.data as CSVRow[];

				if (originalData.length > 0) {
					const headers = Object.keys(originalData[0] || {});
					console.log('Headers detected:', headers);
					fieldMappings = detectFieldMapping(headers);
					console.log('Field mappings:', fieldMappings);
					step = 'mapping';
				} else {
					console.error('No data found in CSV');
				}

				processing = false;
			},
			error: (error) => {
				console.error('CSV parsing error:', error);
				processing = false;
			}
		});
	}

	function handleProcessData() {
		console.log('Processing data...');
		processing = true;

		try {
			processedData = processContacts(originalData);
			console.log('Data processed:', processedData.length, 'contacts');
			step = 'preview';
		} catch (error) {
			console.error('Processing error:', error);
		}

		processing = false;
	}

	function downloadProcessedCSV() {
		if (!file || processedData.length === 0) return;

		const csv = Papa.unparse(processedData);
		const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
		const link = document.createElement('a');
		const url = URL.createObjectURL(blob);
		link.setAttribute('href', url);
		link.setAttribute('download', `processed_${file.name}`);
		link.style.visibility = 'hidden';
		document.body.appendChild(link);
		link.click();
		document.body.removeChild(link);
		URL.revokeObjectURL(url);
	}

	function resetProcessor() {
		file = null;
		originalData = [];
		processedData = [];
		fieldMappings = {};
		step = 'upload';
		if (fileInput) {
			fileInput.value = '';
		}
	}

	function goToMapping() {
		step = 'mapping';
	}
</script>

<div class="min-h-screen bg-gray-50 p-8">
	<div class="mx-auto max-w-6xl">
		<div class="mb-8 text-center">
			<h1 class="mb-2 text-3xl font-bold text-gray-900">CSV Contact Processor</h1>
			<p class="text-gray-600">Clean and standardize your contact CSV files for CRM import</p>
		</div>

		<!-- Progress Steps -->
		<div class="mb-8 flex justify-center">
			<div class="flex items-center space-x-4">
				{#each ['upload', 'mapping', 'preview'] as stepName, index}
					<div class="flex items-center">
						<div
							class="flex h-8 w-8 items-center justify-center rounded-full text-sm font-medium {step ===
							stepName
								? 'bg-blue-600 text-white'
								: stepIndex > index
									? 'bg-green-600 text-white'
									: 'bg-gray-300 text-gray-600'}"
						>
							{#if stepIndex > index}
								✓
							{:else}
								{index + 1}
							{/if}
						</div>
						{#if index < 2}
							<div class="mx-2 h-0.5 w-12 bg-gray-300"></div>
						{/if}
					</div>
				{/each}
			</div>
		</div>

		<!-- Upload Step -->
		{#if step === 'upload'}
			<div class="rounded-lg bg-white p-8 text-center shadow">
				<div class="mx-auto mb-4 h-16 w-16 text-gray-400">
					<svg
						fill="none"
						stroke="currentColor"
						viewBox="0 0 24 24"
						xmlns="http://www.w3.org/2000/svg"
					>
						<path
							stroke-linecap="round"
							stroke-linejoin="round"
							stroke-width="2"
							d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12"
						></path>
					</svg>
				</div>
				<h2 class="mb-4 text-xl font-semibold">Upload Your CSV File</h2>
				<p class="mb-6 text-gray-600">
					Upload a CSV file with contact data. We'll automatically detect and clean the fields.
				</p>
				<input
					bind:this={fileInput}
					type="file"
					accept=".csv"
					onchange={handleFileUpload}
					class="block w-full cursor-pointer text-sm text-gray-500 file:mr-4 file:rounded-full file:border-0 file:bg-blue-50 file:px-4 file:py-2 file:text-sm file:font-semibold file:text-blue-700 hover:file:bg-blue-100"
				/>
				{#if processing}
					<div class="mt-4 flex items-center justify-center text-blue-600">
						<div class="mr-2 h-4 w-4 animate-spin rounded-full border-b-2 border-blue-600"></div>
						Processing file...
					</div>
				{/if}
			</div>
		{/if}

		<!-- Mapping Step -->
		{#if step === 'mapping'}
			<div class="rounded-lg bg-white p-8 shadow">
				<h2 class="mb-4 flex items-center text-xl font-semibold">
					<svg class="mr-2 h-5 w-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
						<path
							stroke-linecap="round"
							stroke-linejoin="round"
							stroke-width="2"
							d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"
						></path>
					</svg>
					Field Mapping Detection
				</h2>
				<p class="mb-6 text-gray-600">
					We've automatically detected these field mappings from your CSV ({originalData.length} rows):
				</p>

				<div class="mb-6 max-h-96 space-y-4 overflow-y-auto">
					{#each Object.entries(fieldMappings) as [standardField, sourceFields]}
						<div class="rounded-lg border p-4">
							<div class="mb-2 font-medium text-gray-900">{standardField}</div>
							<div class="flex flex-wrap gap-2">
								{#each sourceFields as field}
									<span class="rounded-full bg-blue-100 px-3 py-1 text-sm text-blue-800">
										{field}
									</span>
								{/each}
							</div>
						</div>
					{/each}
				</div>

				{#if Object.keys(fieldMappings).length === 0}
					<div class="py-8 text-center text-gray-500">
						<p>No field mappings detected. Your CSV headers might not match our patterns.</p>
						<p class="mt-2 text-sm">Check the console for detected headers.</p>
					</div>
				{/if}

				<div class="flex justify-between">
					<button
						onclick={resetProcessor}
						class="rounded-lg border border-gray-300 px-4 py-2 text-gray-600 hover:bg-gray-50"
					>
						Start Over
					</button>
					<button
						onclick={handleProcessData}
						class="rounded-lg bg-blue-600 px-6 py-2 text-white hover:bg-blue-700 disabled:bg-gray-400"
						disabled={processing || Object.keys(fieldMappings).length === 0}
					>
						{processing ? 'Processing...' : 'Process Data'}
					</button>
				</div>
			</div>
		{/if}

		<!-- Preview Step -->
		{#if step === 'preview'}
			<div class="rounded-lg bg-white p-8 shadow">
				<div class="mb-6 flex items-center justify-between">
					<h2 class="flex items-center text-xl font-semibold">
						<svg class="mr-2 h-5 w-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
							<path
								stroke-linecap="round"
								stroke-linejoin="round"
								stroke-width="2"
								d="M12 4.354a4 4 0 110 5.292M15 21H3v-1a6 6 0 0112 0v1zm0 0h6v-1a6 6 0 00-9-5.197m13.5-9a2.5 2.5 0 11-5 0 2.5 2.5 0 015 0z"
							></path>
						</svg>
						Processed Data Preview
					</h2>
					<div class="text-sm text-gray-600">
						{originalData.length} rows → {processedData.length} contacts
					</div>
				</div>

				{#if processedData.length > 0}
					<div class="mb-6 max-h-96 overflow-auto rounded-lg border">
						<table class="w-full text-sm">
							<thead class="sticky top-0 bg-gray-50">
								<tr>
									{#each Object.keys(processedData[0] || {}) as field}
										<th class="p-3 text-left font-medium whitespace-nowrap text-gray-900">
											{field}
										</th>
									{/each}
								</tr>
							</thead>
							<tbody>
								{#each processedData.slice(0, 10) as row, index}
									<tr class="border-t">
										{#each Object.values(row) as value}
											<td class="p-3 whitespace-nowrap text-gray-600">
												{String(value || '')}
											</td>
										{/each}
									</tr>
								{/each}
							</tbody>
						</table>
					</div>

					{#if processedData.length > 10}
						<p class="mb-6 text-sm text-gray-600">
							Showing first 10 rows of {processedData.length} total contacts
						</p>
					{/if}
				{:else}
					<div class="py-8 text-center text-gray-500">
						<p>No processed data to display.</p>
					</div>
				{/if}

				<div class="flex justify-between">
					<button
						onclick={goToMapping}
						class="rounded-lg border border-gray-300 px-4 py-2 text-gray-600 hover:bg-gray-50"
					>
						Back to Mapping
					</button>
					<button
						onclick={downloadProcessedCSV}
						class="flex items-center rounded-lg bg-green-600 px-6 py-2 text-white hover:bg-green-700 disabled:bg-gray-400"
						disabled={processedData.length === 0}
					>
						<svg class="mr-2 h-4 w-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
							<path
								stroke-linecap="round"
								stroke-linejoin="round"
								stroke-width="2"
								d="M12 10v6m0 0l-3-3m3 3l3-3M3 17V7a2 2 0 012-2h6l2 2h6a2 2 0 012 2v8a2 2 0 01-2-2z"
							></path>
						</svg>
						Download Clean CSV
					</button>
				</div>
			</div>
		{/if}
	</div>
</div>
