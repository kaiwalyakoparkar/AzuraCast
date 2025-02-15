<template>
    <audio
        v-if="isPlaying"
        ref="$audio"
        :title="title"
    />
</template>

<script setup lang="ts">
import getLogarithmicVolume from '~/functions/getLogarithmicVolume';
import Hls from 'hls.js';
import {usePlayerStore} from "~/store";
import {nextTick, onMounted, onScopeDispose, ref, toRef, watch} from "vue";
import {storeToRefs} from "pinia";

const props = defineProps({
    title: {
        type: String,
        default: null
    },
    volume: {
        type: Number,
        default: 55
    },
    isMuted: {
        type: Boolean,
        default: false
    }
});

const $audio = ref(null);
const hls = ref(null);
const duration = ref(0);
const currentTime = ref(0);

const store = usePlayerStore();
const {isPlaying, current} = storeToRefs(store);

const bc = ref<BroadcastChannel | null>(null);

watch(toRef(props, 'volume'), (newVol) => {
    if ($audio.value !== null) {
        $audio.value.volume = getLogarithmicVolume(newVol);
    }
});

watch(toRef(props, 'isMuted'), (newMuted) => {
    if ($audio.value !== null) {
        $audio.value.muted = newMuted;
    }
});

const stop = () => {
    if ($audio.value !== null) {
        $audio.value.pause();
        $audio.value.src = '';
    }

    if (hls.value !== null) {
        hls.value.destroy();
        hls.value = null;
    }

    duration.value = 0;
    currentTime.value = 0;

    store.stopPlaying();
};

const play = () => {
    if (isPlaying.value) {
        stop();
        nextTick(() => {
            play();
        });
        return;
    }

    store.startPlaying();

    nextTick(() => {
        // Handle audio errors.
        $audio.value.onerror = (e) => {
            if (e.target.error.code === e.target.error.MEDIA_ERR_NETWORK && $audio.value.src !== '') {
                console.log('Network interrupted stream. Automatically reconnecting shortly...');
                setTimeout(() => {
                    play();
                }, 5000);
            }
        };

        $audio.value.onended = () => {
            stop();
        };

        $audio.value.ontimeupdate = () => {
            const audioDuration = $audio.value?.duration ?? 0;
            duration.value = (audioDuration !== Infinity && !isNaN(audioDuration)) ? audioDuration : 0;

            currentTime.value = $audio.value?.currentTime ?? null;
        };

        $audio.value.volume = getLogarithmicVolume(props.volume);
        $audio.value.muted = props.isMuted;

        if (current.value.isHls) {
            // HLS playback support
            if (Hls.isSupported()) {
                hls.value = new Hls();
                hls.value.loadSource(current.value.url);
                hls.value.attachMedia($audio.value);
            } else if ($audio.value.canPlayType('application/vnd.apple.mpegurl')) {
                $audio.value.src = current.value.url;
            } else {
                console.log('Your browser does not support HLS.');
            }
        } else {
            // Standard streams
            $audio.value.src = current.value.url;

            // Firefox caches the downloaded stream, this causes playback issues.
            // Giving the browser a new url on each start bypasses the old cache/buffer
            if (navigator.userAgent.includes("Firefox")) {
                $audio.value.src += "?refresh=" + Date.now();
            }
        }

        $audio.value.load();
        $audio.value.play();

        if (bc.value) {
            bc.value.postMessage('played');
        }
    });
};

watch(current, (newCurrent) => {
    if (newCurrent.url === null) {
        stop();
    } else {
        play();
    }
});

const getCurrentTime = () => currentTime.value;
const getDuration = () => duration.value;

const getProgress = () => {
    return (duration.value !== 0)
        ? +((currentTime.value / duration.value) * 100).toFixed(2)
        : 0;
};

const setProgress = (progress) => {
    if ($audio.value !== null) {
        $audio.value.currentTime = (progress / 100) * duration.value;
    }
};

onMounted(() => {
    // Allow pausing from the mobile metadata update.
    if ('mediaSession' in navigator) {
        navigator.mediaSession.setActionHandler('pause', () => {
            stop();
        });
    }

    if ('BroadcastChannel' in window) {
        bc.value = new BroadcastChannel('audio_player');
        bc.value.addEventListener('message', () => {
            stop();
        }, {passive: true});
    }
});

onScopeDispose(() => {
    if (bc.value) {
        bc.value.close()
    }
});

defineExpose({
    play,
    stop,
    getCurrentTime,
    getDuration,
    getProgress,
    setProgress
});
</script>
