<script setup lang="ts">
import ExternalVideo from './ui/ExternalVideo.vue';
import DetailsLazy from './ui/DetailsLazy.vue';

defineProps<{summary: string}>();
</script>

<template>
    <DetailsLazy :summary="summary">
        <ExternalVideo
            v-bind="$attrs"
            showImmediately
            autoplay
            width="100%"
            controls
            playsinline="true"
            muted
            loop
        />
    </DetailsLazy>
</template>