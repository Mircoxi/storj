// Copyright (C) 2023 Storj Labs, Inc.
// See LICENSE for copying information.

<template>
    <v-app>
        <default-bar />
        <default-view />
    </v-app>
</template>

<script setup lang="ts">
import { VApp } from 'vuetify/components';
import { onBeforeMount } from 'vue';

import DefaultBar from './AppBar.vue';
import DefaultView from './View.vue';

import { useUsersStore } from '@/store/modules/usersStore';
import { AnalyticsErrorEventSource } from '@/utils/constants/analyticsEventNames';
import { useNotify } from '@/utils/hooks';

const usersStore = useUsersStore();
const notify = useNotify();

/**
 * Lifecycle hook after initial render.
 * Pre-fetches user`s and project information.
 */
onBeforeMount(async () => {
    try {
        await usersStore.getSettings();
    } catch (error) {
        notify.notifyError(error, AnalyticsErrorEventSource.ALL_PROJECT_DASHBOARD);
    }
});
</script>
