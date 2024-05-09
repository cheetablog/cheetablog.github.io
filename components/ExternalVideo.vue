<script setup lang="ts">
defineProps<{src: string}>();
</script>

<template>
    <video
        :src="src"
        width="100%"
        controls
        playsinline="true"
        preload="metadata"
        muted
        loop
    >
    </video>
</template>