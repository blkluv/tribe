<template>
	<div class="direct">
		<video id="directplayer" preload="auto"></video>
	</div>
</template>

<script lang="ts">
import { defineComponent, onMounted, ref, watch, onBeforeUnmount, toRefs } from "vue";
import Plyr from "plyr";
import Hls from "hls.js";
import "plyr/src/sass/plyr.scss";
import { useStore } from "@/store";

export default defineComponent({
	name: "PlyrPlayer",
	props: {
		service: { type: String, required: true },
		videoUrl: { type: String, required: true },
		videoMime: { type: String, required: true },
		thumbnail: { type: String },
	},
	emits: [
		"apiready",
		"ready",
		"playing",
		"paused",
		"waiting",
		"buffering",
		"error",
		"end",
		"buffer-progress",
		"buffer-spans",
	],
	setup(props, { emit }) {
		const { videoUrl, videoMime, thumbnail } = toRefs(props);
		const videoElem = ref<HTMLVideoElement | undefined>();
		const player = ref<Plyr | undefined>();
		let hls: Hls | undefined = undefined;
		const store = useStore();

		function play() {
			if (!player.value) {
				console.error("player not ready");
				return;
			}
			return player.value.play();
		}

		function pause() {
			if (!player.value) {
				console.error("player not ready");
				return;
			}
			return player.value.pause();
		}

		function setVolume(volume: number) {
			if (!player.value) {
				console.error("player not ready");
				return;
			}
			player.value.volume = volume / 100;
		}

		function getPosition() {
			if (!player.value) {
				console.error("player not ready");
				return;
			}
			return player.value.currentTime;
		}

		function setPosition(position: number) {
			if (!player.value) {
				console.error("player not ready");
				return;
			}
			player.value.currentTime = position;
		}

		function isCaptionsSupported(): boolean {
			return ["direct", "hls"].includes(props.service);
		}

		function setCaptionsEnabled(enabled: boolean): void {
			if (hls) {
				hls.subtitleDisplay = enabled;
			} else {
				player.value?.toggleCaptions(enabled);
			}
		}

		function isCaptionsEnabled(): boolean {
			if (hls) {
				return hls.subtitleDisplay;
			} else {
				return player.value?.currentTrack !== -1;
			}
		}

		function getCaptionsTracks(): string[] {
			const tracks: string[] = [];
			for (let i = 0; i < (videoElem.value?.textTracks?.length ?? 0); i++) {
				const track = videoElem.value?.textTracks[i];
				if (!track || track.kind !== "captions") {
					continue;
				}
				tracks.push(track.language);
			}
			return tracks;
		}

		function setCaptionsTrack(track: string): void {
			if (!player.value) {
				console.error("player not ready");
				return;
			}
			console.log("PlyrPlayer: setCaptionsTrack:", track);
			if (hls) {
				hls.subtitleTrack = findTrackIdx(track);
			} else {
				player.value.currentTrack = findTrackIdx(track);
			}
		}

		function findTrackIdx(language: string): number {
			for (let i = 0; i < (videoElem.value?.textTracks?.length ?? 0); i++) {
				const track = videoElem.value?.textTracks[i];
				if (!track || track.kind !== "captions") {
					continue;
				}
				if (track.language === language) {
					return i;
				}
			}
			return 0;
		}

		function getAvailablePlaybackRates() {
			return [0.25, 0.5, 0.75, 1, 1.25, 1.5, 2];
		}

		async function getPlaybackRate(): Promise<number> {
			if (!player.value) {
				console.error("player not ready");
				return 1;
			}
			return player.value.speed;
		}

		async function setPlaybackRate(rate: number): Promise<void> {
			if (!player.value) {
				console.error("player not ready");
				return;
			}
			player.value.speed = rate;
		}

		onMounted(() => {
			videoElem.value = document.getElementById("directplayer") as HTMLVideoElement;
			player.value = new Plyr(videoElem.value, {
				controls: [],
				clickToPlay: false,
				keyboard: {
					focused: false,
					global: false,
				},
				disableContextMenu: false,
				fullscreen: {
					enabled: false,
				},
			});
			emit("apiready");

			player.value.on("ready", () => emit("ready"));
			player.value.on("ended", () => emit("end"));
			player.value.on("playing", () => emit("playing"));
			player.value.on("pause", () => emit("paused"));
			player.value.on("play", () => emit("waiting"));
			player.value.on("stalled", () => emit("buffering"));
			player.value.on("loadstart", () => emit("buffering"));
			player.value.on("canplay", () => {
				emit("ready");
				store.commit("captions/SET_AVAILABLE_TRACKS", {
					tracks: getCaptionsTracks(),
				});
			});
			player.value.on("progress", () => {
				if (!player.value) {
					return;
				}
				emit("buffer-progress", player.value.buffered);
			});
			player.value.on("error", err => {
				emit("error");
				console.error("PlyrPlayer: error:", err);
			});

			loadVideoSource();
		});
		onBeforeUnmount(() => {
			player.value?.destroy();
			hls?.destroy();
		});

		function loadVideoSource() {
			console.log("PlyrPlayer: loading video source:", videoUrl.value, videoMime.value);
			if (!player.value) {
				console.error("player not ready");
				return;
			}

			if (videoMime.value === "application/x-mpegURL") {
				if (!videoElem.value) {
					console.error("video element not ready");
					return;
				}
				// HACK: force the video element to be recreated...
				player.value.source = {
					type: "video",
					sources: [],
					poster: thumbnail.value,
				};
				videoElem.value = document.querySelector("video") as HTMLVideoElement;
				// ...so that we can use hls.js to change the video source
				hls = new Hls();
				hls.loadSource(videoUrl.value);
				hls.attachMedia(videoElem.value);
				hls.on(Hls.Events.MANIFEST_PARSED, () => {
					console.info("PlyrPlayer: hls.js manifest parsed");
					emit("ready");
					store.commit("captions/SET_AVAILABLE_TRACKS", {
						tracks: getCaptionsTracks(),
					});
				});
				hls.on(Hls.Events.ERROR, (event, data) => {
					console.error("PlyrPlayer: hls.js error:", event, data);
					console.error("PlyrPlayer: hls.js inner error:", data.error);
					emit("error");
				});
				hls.on(Hls.Events.INIT_PTS_FOUND, () => {
					console.info("PlyrPlayer: hls.js init pts found");
				});
				hls.on(Hls.Events.KEY_LOADING, () => {
					console.info("PlyrPlayer: hls.js key loading");
				});
				hls.on(Hls.Events.KEY_LOADED, () => {
					console.info("PlyrPlayer: hls.js key loaded");
				});
			} else {
				hls?.destroy();
				hls = undefined;
				player.value.source = {
					sources: [
						{
							src: videoUrl.value,
							type: videoMime.value,
						},
					],
					type: "video",
					poster: thumbnail.value,
				};
				videoElem.value = document.querySelector("video") as HTMLVideoElement;
			}
			// this is needed to get the player to keep playing after the previous video has ended
			player.value.play();

			if (videoElem.value) {
				videoElem.value.addEventListener("progress", () => {
					if (player.value) {
						emit("buffer-progress", player.value.buffered);
					}
					if (videoElem.value) {
						emit("buffer-spans", videoElem.value.buffered);
					}
				});
				videoElem.value.addEventListener("loadstart", () => {
					console.debug("PlyrPlayer: video loadstart");
					emit("buffering");
				});
				videoElem.value.addEventListener("waiting", () => {
					console.debug("PlyrPlayer: video waiting");
				});
				videoElem.value.addEventListener("stalled", () => {
					console.debug("PlyrPlayer: video stalled");
					emit("buffering");
				});
				videoElem.value.addEventListener("canplay", () => {
					console.debug("PlyrPlayer: video canplay");
				});
			} else {
				console.error("video element not present");
			}
		}

		watch(videoUrl, () => {
			console.log("PlyrPlayer: videoUrl changed");
			if (!player.value) {
				console.error("player not ready");
				return;
			}
			loadVideoSource();
		});

		return {
			player,
			play,
			pause,
			setVolume,
			getPosition,
			setPosition,
			isCaptionsSupported,
			setCaptionsEnabled,
			isCaptionsEnabled,
			getCaptionsTracks,
			setCaptionsTrack,
			getAvailablePlaybackRates,
			getPlaybackRate,
			setPlaybackRate,
		};
	},
});
</script>

<style lang="scss">
.direct,
.plyr {
	display: flex;
	align-items: center;
	justify-content: center;
	max-width: 100%;
	max-height: 100%;
	width: 100%;
	height: 100%;
}

.plyr__video-wrapper {
	max-width: 100%;
	max-height: 100%;
	width: 100%;
	height: 100%;
}

.direct video {
	display: block;
	width: 100%;
	height: 100%;
	object-fit: contain;
	object-position: 50% 50%;
}
</style>
