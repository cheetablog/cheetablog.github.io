<script setup lang="ts">
import { ref, computed } from 'vue'

const props = withDefaults(defineProps<{
    aspectRatio?: number | string;
    showImmediately?: boolean;
}>(),
{
    aspectRatio: 16 / 9,
    showImmediately: false
})

const internalAspectRatio = ref();
const isShow = ref(props.showImmediately)

const handleShow = () => {
    isShow.value = true
}

const handleLoadMetadata = ({ target }: { target: HTMLVideoElement }) => {
    handleShow();

    internalAspectRatio.value = Number(target.videoWidth) / Number(target.videoHeight);
}

const aspectRatio = computed(() => {
    return internalAspectRatio.value || Number(props.aspectRatio)
})

const style = computed(() => {    
    return {
        aspectRatio: aspectRatio.value
    }
})
</script>

<template>
    <div
        :style="style"
        :data-show="isShow"
        class="external-video-wrapper"
    >
        <video
            v-bind="$attrs"
            class="external-video"
            @loadedmetadata="handleLoadMetadata"
            :style="style"
        >
        </video>
    </div>
</template>

<style>
.external-video-wrapper {
    background-color: var(--vp-c-gray-1);
    border-radius: 11px;
    width: 100%;
}

.external-video {
    opacity: 0;
    transition: opacity .2s ease;
    border-radius: 10px;
    width: 100%;
}

.external-video-wrapper[data-show=true] .external-video {
    opacity: 1;
}
</style>