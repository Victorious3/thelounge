<template>
	<form id="form" method="post" action="" @submit.prevent="onSubmit">
		<span id="upload-progressbar" />
		<span id="nick">{{ network.nick }}</span>
		<span
			id="input"
			ref="input"
			dir="auto"
			class="mousetrap"
			enterkeyhint="send"
			contenteditable="true"
			:data-placeholder="getInputPlaceholder(channel)"
			:aria-label="getInputPlaceholder(channel)"
			@input="setPendingMessage"
			@keydown.enter.shift.exact.prevent="onShiftEnter"
			@keydown.enter.exact.prevent="onSubmit"
			@blur="onBlur"
		></span>
		<span
			v-if="store.state.serverConfiguration?.fileUpload"
			id="upload-tooltip"
			class="tooltipped tooltipped-w tooltipped-no-touch"
			aria-label="Upload file"
			@click="openFileUpload"
		>
			<input
				id="upload-input"
				ref="uploadInput"
				type="file"
				aria-labelledby="upload"
				multiple
				@change="onUploadInputChange"
			/>
			<button
				id="upload"
				type="button"
				aria-label="Upload file"
				:disabled="!store.state.isConnected"
			/>
		</span>
		<span
			id="submit-tooltip"
			class="tooltipped tooltipped-w tooltipped-no-touch"
			aria-label="Send message"
		>
			<button
				id="submit"
				type="submit"
				aria-label="Send message"
				:disabled="!store.state.isConnected"
			/>
		</span>
	</form>
</template>

<script lang="ts">
import Mousetrap from "mousetrap";
import autocompletion from "../js/autocompletion";
import commands from "../js/commands/index";
import socket from "../js/socket";
import upload from "../js/upload";
import eventbus from "../js/eventbus";
import {watch, defineComponent, nextTick, onMounted, PropType, ref, onUnmounted} from "vue";
import type {ClientNetwork, ClientChan} from "../js/types";
import {useStore} from "../js/store";

const formattingHotkeys = {
	"mod+k": "\x03",
	"mod+b": "\x02",
	"mod+u": "\x1F",
	"mod+i": "\x1D",
	"mod+o": "\x0F",
	"mod+s": "\x1e",
	"mod+m": "\x11",
};

// Autocomplete bracket and quote characters like in a modern IDE
// For example, select `text`, press `[` key, and it becomes `[text]`
const bracketWraps = {
	'"': '"',
	"'": "'",
	"(": ")",
	"<": ">",
	"[": "]",
	"{": "}",
	"*": "*",
	"`": "`",
	"~": "~",
	_: "_",
};

function extractText(span: HTMLSpanElement): string {
	let text = "";
	for (const child of span.childNodes) {
		if ((child as HTMLElement).tagName == "BR") text += "\n";
		else text += child.textContent;
	}
	return text;
}

function wrapCursor(element: HTMLElement, before: string, after: string) {
	const sel = window.getSelection();
	if (!sel) return;
	const range = sel.getRangeAt(0);
	if (!range || document.activeElement !== element) return;

	const startNode = range.startContainer,
		startOffset = range.startOffset;

	const startTextNode = document.createTextNode(before);
	const endTextNode = document.createTextNode(after);

	const boundaryRange = range.cloneRange();
	boundaryRange.collapse(false);
	boundaryRange.insertNode(endTextNode);
	boundaryRange.setStart(startNode, startOffset);
	boundaryRange.collapse(true);
	boundaryRange.insertNode(startTextNode);

	// Reselect the original text
	range.setStartAfter(startTextNode);
	range.setEndBefore(endTextNode);
	sel.removeAllRanges();
	sel.addRange(range);
}

export default defineComponent({
	name: "ChatInput",
	props: {
		network: {type: Object as PropType<ClientNetwork>, required: true},
		channel: {type: Object as PropType<ClientChan>, required: true},
	},
	setup(props) {
		const store = useStore();
		const input = ref<HTMLSpanElement>();
		const uploadInput = ref<HTMLInputElement>();
		const autocompletionRef = ref<ReturnType<typeof autocompletion>>();

		const setPendingMessage = (e: Event) => {
			const element = e.target as HTMLSpanElement;
			const text = extractText(element);
			if (text.trim().length === 0) element.innerText = "";

			props.channel.pendingMessage = text;
			props.channel.inputHistoryPosition = 0;
		};

		const getInputPlaceholder = (channel: ClientChan) => {
			if (channel.type === "channel" || channel.type === "query") {
				return `Write to ${channel.name}`;
			}

			return "";
		};

		const onShiftEnter = () => {
			const selection = window.getSelection()!;
			if (document.activeElement === input.value) {
				const range = selection.getRangeAt(0)!;
				const br = document.createElement("br");
				range.deleteContents();

				const isSingleLine = input.value.querySelectorAll("br").length === 0;
				range.insertNode(br);

				// We need to add a second trailing br if we want to move the cursor past the last one
				if (isSingleLine && input.value.lastElementChild == br)
					input.value.appendChild(document.createElement("br"));

				range.setStartAfter(br);
				range.setEndAfter(br);
			}
		};

		const onSubmit = () => {
			if (!input.value) {
				return;
			}

			// Triggering click event opens the virtual keyboard on mobile
			// This can only be called from another interactive event (e.g. button click)
			input.value.click();
			input.value.focus();

			if (!store.state.isConnected) {
				return false;
			}

			const target = props.channel.id;
			const text = props.channel.pendingMessage;

			if (text.length === 0) {
				return false;
			}

			if (autocompletionRef.value) {
				if (store.state.isAutoCompleting) {
					autocompletionRef.value.commit();
					autocompletionRef.value.hide();
					return;
				} else {
					autocompletionRef.value.hide();
				}
			}

			props.channel.inputHistoryPosition = 0;
			props.channel.pendingMessage = "";
			input.value.innerText = "";

			// Store new message in history if last message isn't already equal
			if (props.channel.inputHistory[1] !== text) {
				props.channel.inputHistory.splice(1, 0, text);
			}

			// Limit input history to a 100 entries
			if (props.channel.inputHistory.length > 100) {
				props.channel.inputHistory.pop();
			}

			if (text[0] === "/") {
				const args = text.substring(1).split(" ");
				const cmd = args.shift()?.toLowerCase();

				if (!cmd) {
					return false;
				}

				if (
					Object.prototype.hasOwnProperty.call(commands, cmd) &&
					(commands[cmd] as any).input(args)
				) {
					return false;
				}
			}

			socket.emit("input", {target, text});
		};

		const onUploadInputChange = () => {
			if (!uploadInput.value || !uploadInput.value.files) {
				return;
			}

			const files = Array.from(uploadInput.value.files);
			upload.triggerUpload(files);
			uploadInput.value.value = ""; // Reset <input> element so you can upload the same file
		};

		const openFileUpload = () => {
			uploadInput.value?.click();
		};

		const blurInput = () => {
			input.value?.blur();
		};

		const onBlur = () => {
			if (autocompletionRef.value) {
				autocompletionRef.value.hide();
			}
		};

		watch(
			() => props.channel.id,
			() => {
				if (autocompletionRef.value) {
					autocompletionRef.value.hide();
				}
			}
		);

		onMounted(() => {
			eventbus.on("escapekey", blurInput);

			if (store.state.settings.autocomplete) {
				if (!input.value) {
					throw new Error("ChatInput autocomplete: input element is not available");
				}

				autocompletionRef.value = autocompletion(input.value);
			}

			const inputTrap = Mousetrap(input.value);

			inputTrap.bind(Object.keys(formattingHotkeys), function (e, key) {
				const modifier = formattingHotkeys[key];

				if (!e.target) {
					return;
				}

				const selection = window.getSelection()?.getRangeAt(0);
				let wrapEnd = "";
				if (
					document.activeElement === e.target &&
					selection?.startOffset === selection?.endOffset
				)
					wrapEnd = modifier;

				wrapCursor(e.target as HTMLElement, modifier, wrapEnd);

				return false;
			});

			inputTrap.bind(Object.keys(bracketWraps), function (e, key) {
				const selection = window.getSelection()?.getRangeAt(0);
				if (
					document.activeElement === e.target &&
					selection?.startOffset !== selection?.endOffset
				) {
					wrapCursor(e.target as HTMLElement, key, bracketWraps[key]);

					return false;
				}
			});

			inputTrap.bind(["up", "down"], (e, key) => {
				const range = window.getSelection()?.getRangeAt(0)!;

				if (
					store.state.isAutoCompleting ||
					range?.startOffset !== range?.endOffset ||
					!input.value
				) {
					return;
				}

				const childElements = Array.from(input.value.childNodes.values());
				const lastIndex = childElements.indexOf(range.endContainer as ChildNode);
				const onRow = childElements
					.slice(undefined, lastIndex)
					.reduce((sum, val) => sum + ((val as HTMLElement).tagName === "BR" ? 1 : 0), 0);
				const totalRows = input.value.querySelectorAll("br").length;

				const {channel} = props;

				if (channel.inputHistoryPosition === 0) {
					channel.inputHistory[channel.inputHistoryPosition] = channel.pendingMessage;
				}

				if (key === "up" && onRow === 0) {
					if (channel.inputHistoryPosition < channel.inputHistory.length - 1) {
						channel.inputHistoryPosition++;
					} else {
						return;
					}
				} else if (
					key === "down" &&
					channel.inputHistoryPosition > 0 &&
					onRow === totalRows
				) {
					channel.inputHistoryPosition--;
				} else {
					return;
				}

				channel.pendingMessage = channel.inputHistory[channel.inputHistoryPosition];
				input.value.innerText = channel.pendingMessage;
				range.setStartAfter(input.value.lastChild!);
				range.setEndAfter(input.value.lastChild!);

				return false;
			});

			if (store.state.serverConfiguration?.fileUpload) {
				upload.mounted();
			}
		});

		onUnmounted(() => {
			eventbus.off("escapekey", blurInput);

			if (autocompletionRef.value) {
				autocompletionRef.value.destroy();
				autocompletionRef.value = undefined;
			}

			upload.unmounted();
			upload.abort();
		});

		return {
			store,
			input,
			uploadInput,
			onUploadInputChange,
			openFileUpload,
			blurInput,
			onBlur,
			upload,
			getInputPlaceholder,
			onSubmit,
			onShiftEnter,
			setPendingMessage,
		};
	},
});
</script>
