<script setup lang="ts">
import { ref } from 'vue';
import MediaPlaceholder from './MediaPlaceholder.vue';

defineProps<{
    src: string
}>();

const mediaPlaceholderRef = ref<{ show: () => void } | null>(null)

const show = () => {
    mediaPlaceholderRef.value?.show()
}
</script>

<template>
<MediaPlaceholder
    ref="mediaPlaceholderRef"
    class="youtube-video-wrapper"
>
    <iframe
        width="100%"
        height="368"
        :src="src"
        title="YouTube video player"
        frameborder="0"
        allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
        referrerpolicy="strict-origin-when-cross-origin"
        allowfullscreen
        class="youtube-video-component"
        @load="show"
    >
    </iframe>
</MediaPlaceholder>
</template>

<style>
.youtube-video-wrapper {
    height: 368px;
}
</style>