<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import MediaPlaceholder from './MediaPlaceholder.vue';

const props = withDefaults(defineProps<{
    aspectRatio?: number | string;
    showImmediately?: boolean;
}>(),
{
    aspectRatio: 16 / 9,
    showImmediately: false
})

const videoRef = ref<HTMLVideoElement | null>(null);
const internalAspectRatio = ref();

const mediaPlaceholderRef = ref<{ show: () => void } | null>(null)

const show = () => {
    mediaPlaceholderRef.value?.show()
}

const handleLoadMetadata = ({ target }: { target: HTMLVideoElement }) => {
    show();

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

onMounted(() => {
    if (videoRef.value?.preload === "none") {
        show() 
    }
})
</script>

<template>
    <MediaPlaceholder
        ref="mediaPlaceholderRef"
        :showImmediately="showImmediately"
        :style="style"
    >
        <video
            ref="videoRef"
            v-bind="$attrs"
            @loadedmetadata="handleLoadMetadata"
            :style="style"
        >
        </video>
    </MediaPlaceholder>
</template>
