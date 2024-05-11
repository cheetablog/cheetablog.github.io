<script setup lang="ts">
import YoutubeVideo from './ui/YoutubeVideo.vue';
import DetailsLazy from './ui/DetailsLazy.vue';

defineProps<{summary: string}>();
</script>

<template>
    <DetailsLazy :summary="summary">
        <YoutubeVideo v-bind="$attrs" />
    </DetailsLazy>
</template>