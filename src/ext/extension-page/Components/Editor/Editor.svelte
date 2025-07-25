<script>
	import { items, log, v4state } from "../../store.js";
	import Loader from "@shared/Components/Loader.svelte";
	import IconButton from "@shared/Components/IconButton.svelte";
	import Tag from "@shared/Components/Tag.svelte";
	import { formatDate, downloadToFile } from "@ext/utils.js";
	import { sendNativeMessage } from "@ext/native.js";
	import CodeMirror, {
		cmChanged,
		cmGetInstance,
		cmSetSavedCode,
	} from "./CodeMirror.svelte";
	import iconDownload from "@shared/img/icon-download.svg?raw";
	import iconTrash from "@shared/img/icon-trash.svg?raw";
	import iconSync from "@shared/img/icon-sync.svg?raw";
	import iconSidebar from "@shared/img/icon-sidebar.svg?raw";

	export let sidebarHidden;
	export let sidebarSwitch;

	// the data the populates editor elements
	let canUpdate;
	let name;
	let type;
	let lastModified;
	let remote;
	let temp;

	// binds to codemirror component, allows accessing of component's exported functions
	let codemirror;

	// unique discard/save disabled states
	let discardDisabled = true;
	let saveDisabled = true;

	$: disabled = !$v4state.includes("ready");

	$: activeItem = $items.find((a) => a.active);

	$: if (activeItem) {
		if (!cmGetInstance()) codemirror.init();
		// when there active item present, load content into editor
		lastModified = formatDate(activeItem.lastModified);
		name = activeItem.name;
		remote = activeItem.remote;
		temp = activeItem.temp;
		type = activeItem.request ? "request" : activeItem.type;
		canUpdate = activeItem.canUpdate;
		// on load if temp item, disabled discard and enable save, if not disable both
		if (temp) {
			toggleButtons(true, false);
		} else {
			toggleButtons(true, true);
		}
	}

	async function save() {
		if (
			// save already in progress
			!$v4state.includes("ready") ||
			// code hasn't changed and the active script in not a temp
			(!cmChanged() && !activeItem.temp)
		)
			return;
		v4state.add("saving");
		// send the current script data for saving
		const message = {
			name: "PAGE_SAVE",
			item: activeItem,
			content: codemirror.getValue(),
		};
		const response = await sendNativeMessage(message);
		if (response.error) {
			log.add(response.error, "error", true);
		} else {
			// overwrite item in items store
			items.update((i) => {
				const index = i.findIndex((a) => a.active);
				const d = i[index].disabled !== undefined ? i[index].disabled : false;
				const t = i[index].type;
				const visible =
					i[index].visible !== undefined ? i[index].visible : true;
				i[index] = response;
				i[index].active = true;
				i[index].disabled = d;
				i[index].type = t;
				i[index].visible = visible;
				return i;
			});
			// set the newly saved file contents in codemirror instance
			cmSetSavedCode(response.content);
			// refresh session rules
			browser.runtime.sendMessage({ name: "REFRESH_DNR_RULES" });
			// refresh context-menu scripts
			browser.runtime.sendMessage({ name: "REFRESH_CONTEXT_MENU_SCRIPTS" });
		}
		v4state.remove("saving");
	}

	const discard = () => codemirror.discardChanges();

	function download() {
		downloadToFile(activeItem.filename, codemirror.getValue());
	}

	async function update() {
		v4state.add("updating");
		// get the editor's current contents
		const message = { name: "PAGE_UPDATE", content: codemirror.getValue() };
		const response = await sendNativeMessage(message);
		if (response.error) {
			log.add(response.error, "error", true);
		} else if (response.info) {
			log.add(response.info, "info", true);
		} else {
			cmGetInstance().setValue(response.content);
			toggleButtons(false, false);
			log.add("Successfully updated code, review it and save!", "warn", true);
		}
		v4state.remove("updating");
	}

	async function abort() {
		const response = await sendNativeMessage({ name: "CANCEL_REQUESTS" });
		if (response) v4state.remove("updating");
	}

	async function trash() {
		const m = "Are you sure you want to trash this file?";
		const temporary = activeItem.temp;
		if (!window.confirm(m)) return;
		v4state.add("trashing");
		// since temporary files are not yet saved to the file system, can be trashed immediately
		if (temporary) {
			// can only trash an active script, update items filtering out the active one
			items.update((i) => i.filter((a) => !a.active));
		} else {
			const message = { name: "PAGE_TRASH", item: activeItem };
			const response = await sendNativeMessage(message);
			if (response.error) {
				log.add(response.error, "error", true);
			} else {
				items.update((i) => i.filter((a) => !a.active));
				log.add(`Successfully trashed ${activeItem.filename}`, "info", false);
			}
		}
		v4state.remove("trashing");
	}

	function toggleButtons(discardButtonDisabled, saveButtonDisabled) {
		discardDisabled = discardButtonDisabled;
		saveDisabled = saveButtonDisabled;
	}

	function handleMessage(e) {
		const n = e.name;
		// temp files keep save button enabled until save
		if (n === "enableButtons" && !temp) {
			toggleButtons(false, false);
		} else if (n === "disableButtons" && !temp) {
			toggleButtons(true, true);
		}
	}
</script>

<div class="editor">
	{#if $v4state.includes("editor-loading") || $v4state.includes("fetching")}
		<Loader />
	{/if}
	{#if !activeItem}
		<div class="editor__empty">No Item Selected</div>
	{/if}
	<div class="editor__header">
		{#if sidebarHidden}
			<div class="sidebar-switch">
				<IconButton
					icon={iconSidebar}
					on:click={sidebarSwitch}
					title="Show sidebar"
				/>
			</div>
		{/if}
		<div class="editor__header__content">
			<div>
				<Tag {type} />
				<div class="editor__title truncate">{name}</div>
			</div>
			<div class="editor__status">
				<div class="truncate">
					{#if $v4state.includes("saving")}
						Saving...
					{:else if $v4state.includes("trashing")}
						（◞‸◟）
					{:else if $v4state.includes("updating")}
						Updating code, <button class="link" on:click={abort}>
							cancel request
						</button>
					{:else if remote}
						Code was <span class="info" title={remote}>remotely fetched</span>,
						check carefully before saving!
					{:else if temp}
						Ready for code!
					{:else}
						Last modified: {lastModified}
					{/if}
				</div>
			</div>
		</div>
		<div class="editor__header__buttons">
			<IconButton
				icon={iconSync}
				on:click={update}
				title="Check for updates"
				disabled={disabled || !canUpdate}
			/>
			<IconButton
				icon={iconDownload}
				on:click={download}
				title="Download file"
				{disabled}
			/>
			<IconButton
				icon={iconTrash}
				on:click={trash}
				title="Trash file"
				{disabled}
			/>
		</div>
	</div>
	<div class="editor__code">
		<CodeMirror bind:this={codemirror} {handleMessage} saveHandler={save} />
	</div>
	<div class="editor__footer">
		<button on:click={discard} disabled={disabled || discardDisabled}>
			Discard
		</button>
		<button on:click={save} disabled={disabled || saveDisabled}>Save</button>
	</div>
</div>

<style>
	.editor {
		background-color: var(--color-bg-primary);
		display: flex;
		flex-direction: column;
		flex-grow: 1;
		overflow: hidden;
		position: relative;
		width: calc(100svw - var(--sidebar-width));
	}

	.info {
		border-bottom: 1px dotted var(--text-color-disabled);
		cursor: help;
	}

	.editor__empty {
		background-color: inherit;
		color: var(--text-color-disabled);
		font: var(--text-large);
		letter-spacing: var(--letter-spacing-large);
		display: flex;
		place-content: center;
		place-items: center;
		position: absolute;
		inset: 0;
		z-index: 10;
	}

	.editor__header {
		align-items: center;
		display: flex;
		flex-shrink: 0;
		padding: 1rem;
		gap: 1rem;
	}

	.sidebar-switch {
		scale: 1.5;
	}

	.editor__header__content {
		align-items: center;
		display: flex;
		flex-grow: 1;
		flex-wrap: wrap;
		font: var(--text-large);
		min-width: 0; /* needed for .truncate on .title */
	}

	/* element needed so that .truncate works on .title */
	.editor__header__content > div {
		align-items: center;
		display: flex;
		max-width: 100%;
		padding-right: 1rem;
	}

	.editor__title {
		font: var(--text-large);
		letter-spacing: var(--letter-spacing-large);
	}

	.editor__status {
		color: var(--text-color-disabled);
		flex-basis: 100%;
		font: var(--text-small);
		letter-spacing: var(--letter-spacing-small);
	}

	.editor__code {
		display: flex;
		flex-basis: 100%;
		flex-direction: column;
		flex-grow: 1;
		overflow: hidden;
		position: relative;
	}

	.editor__header__buttons {
		display: flex;
		margin-right: 1rem;
	}

	:global(.editor__header__buttons > button:nth-of-type(2)) {
		margin: 0 1.5rem;
	}

	.editor__footer {
		background: var(--editor-blurred-background);
		border-radius: var(--border-radius);
		bottom: 0.25rem;
		padding: 0.5rem 1rem;
		position: absolute;
		right: 0.25rem;
		z-index: 2;
		backdrop-filter: blur(3px);
		-webkit-backdrop-filter: blur(3px);
	}

	.editor__footer button {
		background-color: transparent;
		color: var(--text-color-primary);
		font: var(--text-medium);
		font-weight: bold;
		letter-spacing: var(--letter-spacing-medium);
		line-height: 1.5rem;
	}

	.editor__footer button:hover {
		text-decoration: underline;
	}

	.editor__footer button:nth-child(2) {
		color: var(--color-blue);
		margin-left: 2rem;
	}
</style>
