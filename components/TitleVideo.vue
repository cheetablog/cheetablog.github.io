<script setup lang="ts">
withDefaults(defineProps<{
    src: string;
    caption: string;
    autoplay: boolean;
    preload: "auto" | "metadata" | "none";
}>(), {
    preload: "metadata"
});
</script>

<template>
    <video
        :src="src"
        :autoplay="autoplay"
        :preload="preload"
        width="100%"
        controls
        playsinline="true"
        muted
        loop
    >
    </video>
    <span class="caption">{{caption}}</span>
</template>

<style>
.caption {
    font-size: 0.7em;
    font-style: italic;
}
</style>