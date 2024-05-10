<script setup lang="ts">
import ExternalVideo from './ExternalVideo.vue';

withDefaults(defineProps<{
    src: string;
    caption: string;
    autoplay?: boolean;
    preload?: "auto" | "metadata" | "none";
    aspectRatio?: number | string;
}>(), {
    preload: "metadata"
});
</script>

<template>
    <ExternalVideo
        :src="src"
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