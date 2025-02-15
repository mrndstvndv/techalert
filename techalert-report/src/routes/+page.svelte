<script lang="ts">
	import "../styles.css";
	import csulogo from "$lib/assets/csulogo.png";
	import { onMount } from "svelte";
	import { IssueReporter } from "$lib/types/IssueReporter";
	import Search from "$lib/components/Search.svelte";
	import Settings from "lucide-svelte/icons/settings";
	import CircleMinus from "lucide-svelte/icons/circle-minus";
	import { LazyStore } from "@tauri-apps/plugin-store";
	import Database from "@tauri-apps/plugin-sql";
	import ChipSwitch from "./_components/ChipSwitch.svelte";

	let store: LazyStore;

	let host = "localhost";
	let port = 8080;

	let components: { name: string; enabled: boolean }[] = [
		{ name: "Keyboard", enabled: false },
		{ name: "Mouse", enabled: false },
		{ name: "Monitor", enabled: false },
		{ name: "Power Supply", enabled: false },
	];

	let courses: string[] = [""];
	let course = "BSCS";
	let professors = [""];

	let students: Name[] = [];

	let first_name = "";
	let last_name = "";

	$: (async () => {
		if (db == null) return;

		let result = (await db.select(
			`SELECT t.first_name, t.last_name
FROM subject_table s
JOIN teacher_table t ON s.teacher_id = t.id
WHERE s.year = ? AND s.section = ? AND s.course = ?;`,
			[year, section, course],
		)) as Name[];
		professors = result.map((t) => `${t.first_name} ${t.last_name}`);

		students = (await db.select(
			`select first_name, last_name from student_table where year = ? and section = ? and course = ?;`,
			[year, section, course],
		)) as Name[];
	})();

	$: (async () => {
		let ln = students.find((item) => {
			return item.first_name==first_name
		})?.last_name

		if (ln == undefined) {
			return
		}

		last_name = ln
	})();

	let form: HTMLFormElement;
	let issueReporter: IssueReporter;

	let db: Database | null = null;

	let year = "1";
	let section: "A";

	onMount(async () => {
		store = new LazyStore("settings.json");

		host = (await store.get("host")) ?? host;
		port = (await store.get("port")) ?? port;

		issueReporter = new IssueReporter(host, port);

		db = await Database.load(`mysql://root:@${host}/school`);
		let result = (await db.select(
			`select * from course_table`,
		)) as Course[];
		courses = result.map((e) => e.code);
	});

	function submitDialog(event: Event) {
		const target = event.target as HTMLFormElement;

		if (target === null) throw new Error("Event target is null");

		const formData = new FormData(target);

		host = (formData.get("host") as string) ?? host;
		port = Number.parseInt(formData.get("port") as string) ?? port;

		store?.set("host", host);
		store?.set("port", port);

		issueReporter.setServer(host, port);
		store.save();

		settingsDialogRef.close();
	}

	let settingsDialogRef: HTMLDialogElement;
	function onSettingsClick() {
		settingsDialogRef.showModal();
	}

	function onOtherInputEntered(event: KeyboardEvent) {
		let input = event.target as HTMLInputElement;

		if (event.key == "Enter") {
			event.preventDefault();

			const componentExists = components.some(
				(component) => component.name === input.value,
			);

			if (!componentExists) {
				components.push({
					name: input.value,
					enabled: true,
				});
			}

			components = components;
		}
	}
</script>

<div class="min-h-screen">
	<!-- Header -->
	<header class="bg-[#5C4033] p-4 grid grid-cols-[auto,1fr]">
		<div
			class="container mx-auto flex items-center gap-4 justify-center md:col-start-2"
		>
			<img src={csulogo} alt="University Logo" class="w-16 h-16" />
			<h1 class="text-white text-2xl md:text-3xl font-semibold">
				Cagayan State University
			</h1>
		</div>
		<div class="flex items-center justify-end md:col-start-3 col-start-2">
			<button
				on:click={onSettingsClick}
				class="hover:bg-[#4B3329] rounded-md p-2"
			>
				<Settings class="col-start-3" color="white" />
			</button>
		</div>
	</header>

	<!-- TODO: add functionality to the dialog -->
	<!-- svelte-ignore a11y-click-events-have-key-events -->
	<!-- svelte-ignore a11y-no-noninteractive-element-interactions -->
	<dialog
		on:click|self={() => settingsDialogRef.close()}
		bind:this={settingsDialogRef}
		class="p-6 rounded-lg border shadow-lg border-gray-300 backdrop:bg-background/80 backdrop:fixed backdrop:inset-0 backdrop:backdrop-blur-sm"
	>
		<h2 class="font-semibold text-lg">Endpoint</h2>
		<p class="text-sm mt-[6px]" style="color: hsl(240 3.8% 46.1%);">
			Set the endpoint for the issue reporting system.
		</p>
		<form class="py-4 grid gap-4" on:submit|preventDefault={submitDialog}>
			<div class="grid grid-cols-4 items-center gap-4">
				<label for="host" class="text-right text-sm font-medium"
					>Host</label
				>
				<input
					value={host}
					required
					class="col-span-3"
					type="text"
					name="host"
					id="host"
					placeholder="192.168.1.1"
					autocomplete="off"
				/>
			</div>
			<div class="grid grid-cols-4 items-center gap-4">
				<label for="port" class="text-right text-sm font-medium"
					>Port</label
				>
				<input
					value={port}
					required
					class="col-span-3"
					type="number"
					id="port"
					name="port"
					placeholder="8080"
					autocomplete="off"
				/>
			</div>

			<div class="flex justify-end mt-4">
				<button
					class="bg-green-700 text-white px-8 py-2 rounded-md hover:bg-green-800 transition-colors inline-flex items-center gap-2 text-center"
				>
					Save Changes
				</button>
			</div>
		</form>
	</dialog>

	<!-- Main Form -->
	<main class="container mx-auto p-4 md:p-8">
		<div class="bg-white rounded-lg shadow-lg p-6 md:p-8">
			<form
				class="grid md:grid-cols-2 grid-rows-[auto,1fr] gap-8"
				id="issue-form"
				bind:this={form}
				autocomplete="off"
			>
				<!-- Left Column - Input Fields -->
				<div class="space-y-5">
					<div class="grid md:grid-cols-3 gap-5">
						<div>
							<label
								for="course"
								class="block text-gray-700 font-semibold mb-2 text-sm"
								>Course</label
							>
							<Search
								items={courses}
								bind:value={course}
								classNames="w-full"
								name="course"
								id="course"
								{form}
								required
							/>
						</div>

						<div>
							<label
								for="year"
								class="block text-gray-700 font-semibold mb-2 text-sm"
								>Year</label
							>
							<select
								required
								class="w-full"
								name="year"
								id="year"
								bind:value={year}
							>
								<option value="1">1</option>
								<option value="2">2</option>
								<option value="3">3</option>
								<option value="4">4</option>
							</select>
						</div>

						<div>
							<label
								for="section"
								class="block text-gray-700 font-semibold mb-2 text-sm"
								>Section</label
							>
							<select
								required
								class="w-full"
								name="section"
								id="section"
								bind:value={section}
							>
								<option value="A">A</option>
								<option value="B">B</option>
								<option value="C">C</option>
								<option value="D">D</option>
							</select>
						</div>
					</div>

					<div class="grid md:grid-cols-2 gap-5">
						<div>
							<label
								for="first-name"
								class="block text-gray-700 font-semibold mb-2 text-sm"
								>First name</label
							>

							<!--
							<input
								required
								class="w-full"
								type="text"
								id="first-name"
								bind:value={first_name}
							/>
							-->

							<Search
								classNames="w-full"
								items={students.map((e) => e.first_name)}
								id="first-name"
								name="first-name"
								bind:value={first_name}
								{form}
								required
							/>
						</div>

						<div>
							<label
								for="last-name"
								class="block text-gray-700 font-semibold mb-2 text-sm"
								>Last name</label
							>

							<!--
							<input
								required
								class="w-full"
								type="text"
								id="last-name"
								bind:value={last_name}
							/>
							-->

							<Search
								classNames="w-full"
								items={students.map((e) => e.last_name)}
								id="last-name"
								name="last-name"
								bind:value={last_name}
								{form}
								required
							/>
						</div>
					</div>

					<div>
						<label
							for="professor"
							class="block text-gray-700 font-semibold mb-2 text-sm"
							>Professor</label
						>

						<!-- old proffessor input
						<input
							required
							class="w-full"
							type="text"
							name="professor"
							id="professor"
						/> -->

						<Search
							classNames="w-full"
							items={professors}
							name="professor"
							id="professor"
							value={professors[0]}
							{form}
							required
						/>
					</div>

					<div class="grid grid-cols-2 gap-5">
						<div>
							<label
								for="lab-room"
								class="block text-gray-700 font-semibold mb-2 text-sm"
								>Laboratory Room</label
							>
							<select
								class="w-full"
								name="lab-room"
								id="lab-room"
							>
								<option value="cla">CLA</option>
								<option value="clb">CLB</option>
								<option value="clc">CLC</option>
							</select>
						</div>

						<div>
							<label
								for="pc-number"
								class="block text-gray-700 font-semibold mb-2 text-sm"
								>PC number</label
							>
							<input
								required
								type="number"
								name="pc-number"
								id="pc-number"
								class="h-[40px] w-full"
							/>
						</div>
					</div>
				</div>

				<!-- Right Column - Concern Textarea -->
				<div class="flex flex-col gap-4">
					<div>
						<p
							class="block text-gray-700 font-semibold mb-2 text-sm"
						>
							Urgency
						</p>

						<ChipSwitch></ChipSwitch>

						<p
							class="mt-4 block text-gray-700 font-semibold mb-2 text-sm"
						>
							Issues
						</p>
						<ul class="list-none grid grid-cols-2">
							{#each components as component, index (component.name)}
								<div
									class="grid grid-cols-[1fr,auto] pr-4 group items-center"
								>
									<label
										for={component.name}
										class="flex items-center col-start-1"
									>
										<input
											checked={component.enabled}
											id={component.name}
											type="checkbox"
										/>
										<span class="ml-2"
											>{component.name}</span
										>
									</label>

									<button
										on:click|preventDefault={() => {
											if (index < 4) {
												return;
											}

											components.splice(index, 1);
											components = components;
										}}
										class="col-start-2 hover:bg-black/15 rounded-full p-2 invisible group-hover:visible"
									>
										<CircleMinus
											class="size-5"
											color="red"
										/>
									</button>
								</div>
							{/each}
							<!-- TODO: Add guard for when non of the items are selected-->
						</ul>

						<div
							class="py-3 flex text-sm items-center align-middle"
						>
							<div class="col-start-2 flex flex-col">
								<label class="pb-1" for="other-input"
									>Other</label
								>
								<input
									id="other-input"
									name="other-input"
									class="h-6 col-start-3"
									type="text"
									autocomplete="off"
									on:keydown={onOtherInputEntered}
								/>
							</div>
						</div>
					</div>

					<textarea
						placeholder="Type your concern here."
						class="w-full row-span-2 h-[200px] md:h-full p-4 border-[1.5px] border-green-600 rounded-lg resize-none focus:ring-2 focus:ring-green-500 focus:border-green-500"
					></textarea>
				</div>

				<div class="flex justify-end md:col-start-2 md:row-start-2">
					<button
						type="submit"
						class="bg-green-700 text-white px-8 py-2 rounded-md hover:bg-green-800 transition-colors inline-flex items-center gap-2 text-center"
					>
						Send
						<!-- <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none"
							stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"
							class="lucide lucide-send-horizontal">
							<path
								d="M3.714 3.048a.498.498 0 0 0-.683.627l2.843 7.627a2 2 0 0 1 0 1.396l-2.842 7.627a.498.498 0 0 0 .682.627l18-8.5a.5.5 0 0 0 0-.904z" />
							<path d="M6 12h16" />
						</svg> -->
					</button>
				</div>
			</form>
		</div>
	</main>
</div>

<style>
	label,
	h2,
	p {
		user-select: none;
	}

	input[type="text"],
	input[type="number"] {
		@apply border-[1.5px] border-gray-300 shadow-sm rounded-lg p-2 focus:outline-none focus:ring-[1.5px] focus:ring-green-500 focus:border-green-500;
	}

	input[type="checkbox"] {
		@apply w-4 h-4 text-green-600 border-gray-300 rounded focus:ring-blue-500 focus:ring-2;
	}

	button:hover {
		cursor: default;
	}

	select {
		@apply border-[1.5px] border-gray-300 shadow-sm rounded-lg p-2 focus:outline-none focus:ring-[1.5px] focus:ring-green-500 focus:border-green-500 bg-white;
	}
</style>
