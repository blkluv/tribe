<template>
	<div
		:class="{
			chat: true,
			activated: activated,
		}"
	>
		<div class="chat-header d-flex flex-row" v-if="activated">
			<v-btn
				icon
				size="x-small"
				@click="setActivated(false)"
				data-cy="chat-deactivate"
				aria-label="close chat"
			>
				<v-icon>fa:fas fa-chevron-down</v-icon>
			</v-btn>
			<h4>{{ $t("chat.title") }}</h4>
		</div>
		<div ref="messages" @scroll="onScroll" class="messages d-flex flex-column flex-grow-1 mt-2">
			<div class="d-flex flex-grow-1"><!-- Spacer --></div>
			<transition-group name="message">
				<div class="message" v-for="(msg, index) in chatMessagePast" :key="index">
					<div class="from">{{ msg.from.name }}</div>
					<div class="text">
						<ProcessedText :text="msg.text" @link-click="e => $emit('link-click', e)" />
					</div>
				</div>
				<div
					class="message recent"
					v-for="(msg, index) in chatMessageRecent"
					:key="chatMessagePast.length + index"
				>
					<!-- FIXME: reduce duplicated code by moving this to another component, preferably in this same file.  -->
					<div class="from">{{ msg.from.name }}</div>
					<div class="text"><ProcessedText :text="msg.text" /></div>
				</div>
			</transition-group>
		</div>
		<div v-if="!stickToBottom" class="to-bottom">
			<v-btn size="x-small" icon @click="forceToBottom">
				<v-icon>fa:fas fa-angle-double-down</v-icon>
			</v-btn>
		</div>
		<Transition name="input" @after-enter="enforceStickToBottom">
			<div class="input-box" v-if="activated">
				<v-text-field
					variant="solo"
					density="compact"
					single-line
					:placeholder="$t('chat.type-here')"
					@keydown="onInputKeyDown"
					v-model="inputValue"
					autocomplete="off"
					ref="chatInput"
					@blur="deactivateOnBlur && setActivated(false)"
					data-cy="chat-input"
				/>
			</div>
		</Transition>
		<div class="manual-activate" v-if="!activated">
			<v-btn
				variant="text"
				icon
				size="x-small"
				@click="setActivated(true, true)"
				color="white"
				data-cy="chat-activate"
				aria-label="open chat"
			>
				<v-icon>fa:far fa-comment-alt</v-icon>
			</v-btn>
		</div>
	</div>
</template>

<script lang="ts">
import ProcessedText from "@/components/ProcessedText.vue";
import { defineComponent, onUpdated, ref, Ref, nextTick, onMounted, onUnmounted } from "vue";
import type { ChatMessage } from "ott-common/models/types";
import { useConnection } from "@/plugins/connection";
import { useRoomApi } from "@/util/roomapi";
import { ServerMessageChat } from "ott-common/models/messages";
import { useRoomKeyboardShortcuts } from "@/util/keyboard-shortcuts";
import { useSfx } from "@/plugins/sfx";

const MSG_SHOW_TIMEOUT = 20000;

const Chat = defineComponent({
	name: "Chat",
	components: {
		ProcessedText,
	},
	emits: ["link-click"],
	setup() {
		const connection = useConnection();
		const roomapi = useRoomApi(connection);

		const inputValue = ref("");
		const stickToBottom = ref(true);
		/**
		 * When chat is activated, all messages are shown. and the
		 * user can scroll through message history, type in chat, etc.
		 * When chat is NOT activated, when messages are received,
		 * they appear and fade away after `MSG_SHOW_TIMEOUT` ms.
		 */
		const activated = ref(false);
		const deactivateOnBlur = ref(false);
		/**
		 * All past chat messages. They are are no longer
		 * shown when deactivated.
		 */
		const chatMessagePast: Ref<ChatMessage[]> = ref([]);
		/**
		 * All recent chat messages that are currently shown when deactivated.
		 * They will fade away after `MSG_SHOW_TIMEOUT` ms, and moved into `chatMessagePast`.
		 */
		const chatMessageRecent: Ref<ChatMessage[]> = ref([]);
		const messages = ref();
		const chatInput: Ref<HTMLInputElement | undefined> = ref();

		const shortcuts = useRoomKeyboardShortcuts();
		onMounted(() => {
			connection.addMessageHandler("chat", onChatReceived);
			if (shortcuts) {
				shortcuts.bind({ code: "KeyT" }, () => setActivated(true, false));
			} else {
				console.warn("No keyboard shortcuts available");
			}
		});

		onUnmounted(() => {
			connection.removeMessageHandler("chat", onChatReceived);
		});

		function focusChatInput() {
			chatInput.value?.focus();
		}

		function isActivated(): boolean {
			return activated.value;
		}

		async function setActivated(value: boolean, manual = false): Promise<void> {
			activated.value = value;
			if (value) {
				if (manual) {
					deactivateOnBlur.value = false;
				} else {
					deactivateOnBlur.value = true;
				}
				await nextTick();
				focusChatInput();
			} else {
				chatInput.value?.blur();
				forceToBottom();
			}
		}

		const sfx = useSfx();
		function onChatReceived(msg: ServerMessageChat): void {
			chatMessageRecent.value.push(msg);
			setTimeout(expireChatMessage, MSG_SHOW_TIMEOUT);
			nextTick(enforceStickToBottom);
			sfx.play("pop");
		}

		function expireChatMessage() {
			chatMessagePast.value.push(chatMessageRecent.value.splice(0, 1)[0]);
		}

		/**
		 * Performs the necessary actions to enact the stickToBottom behavior.
		 */
		function enforceStickToBottom() {
			const div = messages.value as HTMLDivElement;
			if (stickToBottom.value) {
				div.scrollTop = div.scrollHeight;
			}
		}

		function onInputKeyDown(e: KeyboardEvent): void {
			if (e.key === "Enter") {
				e.preventDefault();
				if (inputValue.value.trim() !== "") {
					roomapi.chat(inputValue.value);
				}
				inputValue.value = "";
				stickToBottom.value = true;
				setActivated(false);
			} else if (e.key === "Escape") {
				e.preventDefault();
				setActivated(false);
			}
		}

		function onScroll() {
			const div = messages.value as HTMLDivElement;
			const distToBottom = div.scrollHeight - div.clientHeight - div.scrollTop;
			stickToBottom.value = distToBottom === 0;
		}

		function forceToBottom() {
			stickToBottom.value = true;
			enforceStickToBottom();
		}

		onUpdated(enforceStickToBottom);

		return {
			inputValue,
			stickToBottom,
			activated,
			chatMessagePast,
			chatMessageRecent,
			deactivateOnBlur,

			onInputKeyDown,
			onScroll,
			focusChatInput,
			isActivated,
			setActivated,
			onChatReceived,
			enforceStickToBottom,
			forceToBottom,

			messages,
			chatInput,
		};
	},
});

export default Chat;
</script>

<style lang="scss" scoped>
@import "../variables.scss";

$chat-message-bg: $background-color;

.chat {
	display: flex;
	flex-direction: column;
	margin: 4px;
	padding: 3px;
	transition: all 0.2 ease;
	pointer-events: none;
	height: 100%;

	&.activated {
		background: rgba(var(--v-theme-background), $alpha: 0.8);
		pointer-events: auto;
	}
}

.activated {
	.message {
		opacity: 1;

		&.recent {
			background: transparent;
			transition-duration: 0.2s;
		}
	}

	.messages {
		overflow-y: auto;
		pointer-events: auto;
	}
}

.chat-header {
	border-bottom: 1px solid #666;
}

.input-box {
	display: flex;
	justify-self: end;
	flex-shrink: 1;
	height: 40px;
}

.messages {
	overflow: hidden;
	pointer-events: none;

	flex-basis: 0;
	align-items: baseline;
}

.message {
	margin: 2px 0;
	padding: 4px;
	opacity: 0;
	transition: all 1s ease;

	&:first-child {
		margin-top: auto;
	}

	&.recent {
		opacity: 1;
		background: rgba(var(--v-theme-background), $alpha: 0.6);
	}

	.from,
	.text {
		display: inline;
		margin: 3px 5px;
		word-wrap: break-word;
		overflow-wrap: anywhere;
	}

	.from {
		font-weight: bold;
		margin-left: 0;
	}

	@media screen and (max-width: $sm-max) {
		font-size: 0.8em;
	}
}

.manual-activate {
	display: flex;
	align-self: flex-end;
	justify-self: end;
	pointer-events: auto;
}

.to-bottom {
	display: flex;
	justify-content: start;
	width: 100%;
	pointer-events: auto;
	margin: 6px 0;
	position: absolute;
	bottom: 42px;
	z-index: 100;
}

// Transition animation
.message-enter-active,
.message-leave-active {
	transition: all 0.2s;
}
.message-enter,
.message.leave-to {
	opacity: 0;
	transform: translateX(-30px) scaleY(0);
}
.message-move {
	transition: transform 0.2s;
}

.input-enter-active,
.input-leave-active {
	transition: all 0.2s ease;
}
.input-enter,
.input-leave-to {
	opacity: 0;
	transform: translateY(-30px) scaleY(0);
	height: 0;
}
</style>
