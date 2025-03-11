<script lang="ts">
	import { fade } from "svelte/transition";
	import Highlighter from "../components/Highlighter.svelte";
	import Papa from "papaparse";

	// State variables
	let url = $state("https://a7dc-111-94-113-192.ngrok-free.app");
	let path = $state(
		"/api/chat?passphrase=eJMZBmAL6bNZtMLWHpiQwAksDvIfEaaqt5QZ&config_id=1&app_id=3",
	);
	let method: "GET" | "POST" | "PUT" | "DELETE" | "PATCH" | "HEAD" =
		$state("POST");
	let headers: { key: string; value: string }[] = $state([
		{
			key: "Authorization",
			value: "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2NjczNmNmZTZmZDJhYzBmMmMzZTZmYzMiLCJuYW1lIjoiTU9OSUtBIiwiZW1haWwiOiJtb25pa2FAZnhtd2ViLmNvbSIsInJvbGUiOiJBZG1pbmlzdHJhdG9yIiwiY29uZGl0aW9uIjoiQzEsQzIsQzMiLCJnZW5kZXIiOiJNYWxlIiwic3RhdHVzIjoiQWN0aXZlIiwiZXhwIjoxNzcwMDk1NjY4LCJpYXQiOjE3Mzg1NTk2NDd9.mXqQ9zQpb-gUlOVsiqmtc9gMrHePq5LAMcK4OeUc4ak",
		},
		{ key: "Content-Type", value: "application/json" },
	]);
	let prompt = $state("There is a dead mouse near me...");
	let expressionModel = $state("gpt-4o-mini");
	let body = $derived({
		prompt,
		chat_history: JSON.stringify([
			{
				ai: "Hey there! 😊 What did you have for breakfast or lunch today? I'm super curious!",
			},
			{ user: "what is sin 90?" },
			{ ai: "Sin 90 is 1! That's a nice little fact to remember." },
		]),
		use_audio: "false",
		use_expression: "true",
		activity: "icebreaker",
		activity_count: 3,
		expression_model: expressionModel,
	});
	let bodyStr = $derived(JSON.stringify(body, null, "\t"));

	// Response storage
	let database: {
		timeTaken: number;
		statusCode: number;
		body: any;
		headers: Record<string, string>;
	}[] = $state([]);
	let error = $state<string | null>(null);
	let isLoading = $state(false);
	let tabPreview = $state(0);
	let runner = $state(1);
	let delay = $state(0);

	// Derived statistics
	let totalTests = $derived(database.length);
	let totalSuccess = $derived(
		database.filter((v) => v.statusCode === 200).length,
	);
	let totalFailed = $derived(
		database.filter((v) => v.statusCode !== 200).length,
	);
	let averageTime = $derived.by(() => {
		if (!database.length) return 0;

		let sum = 0;
		database.forEach((v) => (sum += v.timeTaken));

		return sum / database.length;
	});
	let expressions = $derived(
		database.map((v) => v.body?.data?.expression?.expression),
	);

	// Utility functions
	function sleep(ms: number) {
		return new Promise((resolve) => setTimeout(resolve, ms));
	}

	function addHeader() {
		headers = [...headers, { key: "", value: "" }];
	}

	function removeHeader(index: number) {
		headers = headers.filter((_, i) => i !== index);
	}

	async function sendRequest() {
		error = null;
		isLoading = true;
		const startTime = performance.now();

		try {
			const headersObj = Object.fromEntries(
				headers
					.filter((h) => h.key && h.value)
					.map((h) => [h.key, h.value]),
			);

			const options: RequestInit = {
				method,
				headers: headersObj,
				body:
					method !== "GET" && method !== "HEAD"
						? JSON.stringify(body)
						: undefined,
			};

			const res = await fetch(url + path, options);
			const timeTaken = performance.now() - startTime;

			const contentType = res.headers.get("content-type") || "";
			const responseBody = contentType.includes("application/json")
				? await res.json()
				: await res.text();

			database.push({
				headers: Object.fromEntries(res.headers.entries()),
				body: responseBody,
				statusCode: res.status,
				timeTaken,
			});
		} catch (err) {
			error = (err as Error).message;
		} finally {
			isLoading = false;
		}
	}

	async function handleSendRequest(e: Event) {
		e.preventDefault();
		database = [];
		for (let i = 0; i < runner; i++) {
			await sendRequest();
			await sleep(delay);
		}
	}

	function exportToCSV() {
		if (database.length === 0) return;

		const csv = Papa.unparse(
			database.map((v, idx) => ({
				No: idx + 1,
				"Time Taken (ms)": v.timeTaken,
				"Status Code": v.statusCode,
				Prompt: prompt,
				Response: v.body?.data?.text,
				Expression: v.body?.data?.expression.expression,
				Model: v.body?.data?.model,
				"Raw Response Body": JSON.stringify(v.body),
			})),
		);

		const blob = new Blob([csv], { type: "text/csv" });
		const a = document.createElement("a");
		a.href = URL.createObjectURL(blob);
		a.download = "test-results.csv";
		document.body.appendChild(a);
		a.click();
		document.body.removeChild(a);
	}
</script>

<div class="flex my-14">
	<main class="px-10 w-10/12">
		<h1 class="font-bold text-3xl">
			Expression Consistency Tests (Adjusted for PR <a
				href="https://github.com/fxmgithub/2024-fxm-ai-influencer/pull/177"
				class="underline"
				target="_blank">#117</a
			>)
		</h1>
		<p class="text-sm mt-1 max-w-4xl">
			Created by <a class="underline" href="https://github.com/resqiar"
				>@resqiar</a
			>
			adjusted to work with a local ngrok tunnel. Since the current PR does
			not provide a public test URL, I set up my own to evaluate the quality
			of my changes. Feel free to change the settings.
		</p>

		<form onsubmit={handleSendRequest} class="mt-6">
			<select class="select max-w-[100px]" bind:value={method}>
				<option value="GET">GET</option>
				<option value="POST" selected>POST</option>
				<option value="PUT">PUT</option>
				<option value="DELETE">DELETE</option>
			</select>

			<label class="input font-bold w-xs">
				Target URL
				<input
					type="text"
					class="font-normal"
					bind:value={url}
					required
				/>
			</label>

			<label class="input font-bold w-2xl">
				Path
				<input
					type="text"
					class="font-normal"
					bind:value={path}
					required
				/>
			</label>

			<fieldset class="mt-4">
				<legend class="font-bold m">Headers</legend>
				<div class="mt-4 flex flex-col gap-2">
					{#each headers as header, index}
						<div class="flex gap-2">
							<input
								type="text"
								class="input"
								placeholder="Key (e.g., Authorization)"
								bind:value={header.key}
							/>
							<input
								type="text"
								class="input"
								placeholder="Value"
								bind:value={header.value}
							/>
							<button
								type="button"
								class="btn btn-error"
								onclick={() => removeHeader(index)}
								>Remove</button
							>
						</div>
					{/each}
				</div>
				<button
					type="button"
					class="my-2 btn btn-ghost font-normal px-2 btn-sm"
					onclick={addHeader}>Add New Header</button
				>
			</fieldset>

			{#if method !== "GET" && method !== "HEAD"}
				<div class="flex items-center gap-4">
					<fieldset class="fieldset">
						<legend class="fieldset-legend text-lg"
							>Insert Prompt Here</legend
						>
						<textarea
							bind:value={prompt}
							class="textarea h-48 min-w-md"
							placeholder="Enter Prompt"
						></textarea>
					</fieldset>

					<fieldset class="fieldset">
						<legend class="fieldset-legend font-normal"
							>Body Preview</legend
						>
						<textarea
							value={bodyStr}
							class="textarea h-48 min-w-md text-wrap"
							disabled
						></textarea>
					</fieldset>
				</div>
			{/if}

			<div class="mt-6 flex items-end gap-4">
				<fieldset class="fieldset">
					<legend class="fieldset-legend font-normal"
						>Expression Model</legend
					>
					<input
						type="text"
						class="input"
						placeholder="Value"
						bind:value={expressionModel}
					/>
				</fieldset>

				<fieldset class="fieldset">
					<legend class="fieldset-legend">Run how many times?</legend>
					<select class="select max-w-[200px]" bind:value={runner}>
						<option value={1} selected>1x</option>
						<option value={3}>3x</option>
						<option value={5}>5x</option>
						<option value={10}>10x</option>
						<option value={20}>20x</option>
					</select>
				</fieldset>

				<fieldset class="fieldset">
					<legend class="fieldset-legend">Delayed?</legend>
					<select class="select max-w-[200px]" bind:value={delay}>
						<option value={0} selected>0s (No delay)</option>
						<option value={1000}>1s</option>
						<option value={5000}>5s</option>
						<option value={10000}>10s</option>
					</select>
				</fieldset>

				<div class="flex items-center gap-4">
					<button
						type="submit"
						class="btn flex gap-2 items-center btn-primary mb-1"
						disabled={isLoading}
					>
						Send Request
						<svg
							xmlns="http://www.w3.org/2000/svg"
							viewBox="0 0 24 24"
							fill="currentColor"
							class="size-5"
						>
							<path
								d="M3.478 2.404a.75.75 0 0 0-.926.941l2.432 7.905H13.5a.75.75 0 0 1 0 1.5H4.984l-2.432 7.905a.75.75 0 0 0 .926.94 60.519 60.519 0 0 0 18.445-8.986.75.75 0 0 0 0-1.218A60.517 60.517 0 0 0 3.478 2.404Z"
							/>
						</svg>
					</button>

					<button
						type="button"
						onclick={exportToCSV}
						class="btn flex gap-2 items-center btn-secondary mb-1"
						disabled={!database.length || isLoading}
					>
						Export Data as CSV
						<svg
							xmlns="http://www.w3.org/2000/svg"
							viewBox="0 0 24 24"
							fill="currentColor"
							class="size-5"
						>
							<path
								fill-rule="evenodd"
								d="M12 2.25a.75.75 0 0 1 .75.75v11.69l3.22-3.22a.75.75 0 1 1 1.06 1.06l-4.5 4.5a.75.75 0 0 1-1.06 0l-4.5-4.5a.75.75 0 1 1 1.06-1.06l3.22 3.22V3a.75.75 0 0 1 .75-.75Zm-9 13.5a.75.75 0 0 1 .75.75v2.25a1.5 1.5 0 0 0 1.5 1.5h13.5a1.5 1.5 0 0 0 1.5-1.5V16.5a.75.75 0 0 1 1.5 0v2.25a3 3 0 0 1-3 3H5.25a3 3 0 0 1-3-3V16.5a.75.75 0 0 1 .75-.75Z"
								clip-rule="evenodd"
							/>
						</svg>
					</button>

					{#if isLoading}
						<div class="flex items-center gap-2">
							<div
								class="flex animate-spin items-center justify-center gap-1"
							>
								<div
									class="h-1 w-1 rounded-full bg-white"
								></div>
								<div
									class="h-2 w-2 rounded-full bg-yellow-200"
								></div>
								<div
									class="h-1 w-1 rounded-full bg-white"
								></div>
							</div>
						</div>
					{/if}
				</div>
			</div>
		</form>

		{#if database.length > 1}
			<div class="tabs tabs-box w-fit mt-12">
				{#each database as _, idx}
					<input
						type="radio"
						name={`Response ${idx + 1}`}
						class="tab"
						aria-label={`Response ${idx + 1}`}
						checked={tabPreview === idx ? true : false}
						onchange={() => (tabPreview = idx)}
					/>
				{/each}
			</div>
		{/if}

		<section class="mt-6">
			{#if error}
				<p>Error: {error}</p>
			{/if}

			{#if database[tabPreview]}
				<div transition:fade>
					<h2 class="font-bold">API Response</h2>
					<p>
						Status Code:
						<span class="font-bold">
							{database[tabPreview].statusCode}
						</span>
					</p>
					<p>
						Time Taken: <span class="font-bold"
							>{database[tabPreview].timeTaken.toFixed(0)}</span
						>
					</p>

					<h3 class="font-bold text-lg mt-4">Response Headers</h3>
					<div class="mt-2">
						<div id="code-preview" class="mockup-code bg-[#011627]">
							<Highlighter
								code={JSON.stringify(
									database[tabPreview].headers,
									null,
									2,
								)}
							/>
						</div>
					</div>

					<h3 class="font-bold text-lg mt-4">Response Body</h3>
					<div class="mt-2">
						<div id="code-preview" class="mockup-code bg-[#011627]">
							<Highlighter
								code={JSON.stringify(
									database[tabPreview].body,
									null,
									2,
								)}
							/>
						</div>
					</div>
				</div>
			{/if}
		</section>
	</main>

	<div class="w-2/12 mr-12">
		<div class="card bg-gray-700 shadow-xl">
			<div class="card-body">
				<h1 class="font-bold text-lg">Summary of The Test</h1>
				<p class="text-sm">
					Total Tests: <span class="font-bold">{totalTests}</span>
				</p>
				<p class="text-sm">
					Total Success (200): <span class="font-bold"
						>{totalSuccess}</span
					>
				</p>
				<p class="text-sm">
					Total Failure (non 200): <span class="font-bold"
						>{totalFailed}</span
					>
				</p>
				<p class="text-sm">
					Average Response Time: <span class="font-bold"
						>{averageTime.toFixed(0)}ms</span
					>
				</p>

				<ol class="list-decimal">
					<p class="text-sm">List of Expressions:</p>
					{#each expressions as item}
						<li class="text-sm font-bold ml-4">{item}</li>
					{/each}
				</ol>
			</div>
		</div>
	</div>
</div>
