<script setup lang="ts">
import { computed } from 'vue';
import ExternalVideo from './ui/ExternalVideo.vue';

const props = withDefaults(defineProps<{
    src: string;
    caption: string;
    autoplay?: boolean;
    preload?: "auto" | "metadata" | "none";
    aspectRatio?: number | string;
}>(), {
    preload: "metadata"
});

const internalSrc = computed(() => {
    return props.src + "#t=0.1"
})
</script>

<template>
    <ExternalVideo
        :src="internalSrc"
        :autoplay="autoplay"
        :preload="preload"
        :aspectRatio="aspectRatio"
        width="100%"
        controls
        playsinline="true"
        muted
        loop
    />
    <span class="caption">{{caption}}</span>
</template>

<style>
.caption {
    font-size: 0.7em;
    font-style: italic;
}
</style>