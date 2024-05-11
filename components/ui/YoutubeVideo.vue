<script setup lang="ts">
defineProps<{
    src: string
}>();
</script>

<template>
    <iframe
        width="100%"
        height="368"
        :src="src"
        title="YouTube video player"
        frameborder="0"
        allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
        referrerpolicy="strict-origin-when-cross-origin"
        allowfullscreen>
    </iframe>
</template>