// Copyright (C) 2023 Storj Labs, Inc.
// See LICENSE for copying information.

<template>
    <v-card :title="title" :subtitle="subtitle" variant="flat" :border="true" rounded="xlg">
        <v-card-text>
            <v-chip rounded color="success" variant="outlined" class="font-weight-bold">{{ data }}</v-chip>
        </v-card-text>
    </v-card>
</template>

<script setup lang="ts">
import { VCard, VCardText, VChip } from 'vuetify/components';

const props = defineProps<{
    title: string;
    subtitle: string;
    data: string;
}>();
</script>
