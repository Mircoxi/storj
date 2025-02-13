// Copyright (C) 2023 Storj Labs, Inc.
// See LICENSE for copying information.

<template>
    <v-app>
        <div v-if="isLoading" class="d-flex align-center justify-center w-100 h-100">
            <v-progress-circular color="primary" indeterminate size="64" />
        </div>
        <template v-else>
            <default-bar show-nav-drawer-button />
            <ProjectNav v-if="appStore.state.isNavigationDrawerShown" />
            <default-view />
        </template>
    </v-app>
</template>

<script setup lang="ts">
import { onBeforeMount, ref, watch } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import { VApp, VProgressCircular } from 'vuetify/components';

import DefaultBar from './AppBar.vue';
import ProjectNav from './ProjectNav.vue';
import DefaultView from './View.vue';

import { RouteConfig } from '@/types/router';
import { Project } from '@/types/projects';
import { useBillingStore } from '@/store/modules/billingStore';
import { useUsersStore } from '@/store/modules/usersStore';
import { useABTestingStore } from '@/store/modules/abTestingStore';
import { useProjectsStore } from '@/store/modules/projectsStore';
import { useAppStore } from '@poc/store/appStore';
import { useAnalyticsStore } from '@/store/modules/analyticsStore';
import { useAccessGrantsStore } from '@/store/modules/accessGrantsStore';
import { AnalyticsErrorEventSource } from '@/utils/constants/analyticsEventNames';
import { useNotify } from '@/utils/hooks';

const router = useRouter();
const route = useRoute();
const notify = useNotify();

const analyticsStore = useAnalyticsStore();
const billingStore = useBillingStore();
const usersStore = useUsersStore();
const abTestingStore = useABTestingStore();
const projectsStore = useProjectsStore();
const appStore = useAppStore();
const agStore = useAccessGrantsStore();

const isLoading = ref<boolean>(true);

/**
 * Selects the project with the given ID, redirecting to the
 * all projects dashboard if no such project exists.
 */
async function selectProject(projectId: string): Promise<void> {
    isLoading.value = true;

    let projects: Project[];
    try {
        projects = await projectsStore.getProjects();
    } catch (_) {
        const path = '/projects';
        router.push(path);
        analyticsStore.pageVisit(path);
        return;
    }
    if (!projects.some(p => p.id === projectId)) {
        const path = '/projects';
        router.push(path);
        analyticsStore.pageVisit(path);
        return;
    }
    projectsStore.selectProject(projectId);

    isLoading.value = false;
}

watch(() => route.params.projectId, async newId => selectProject(newId as string));

/**
 * Lifecycle hook after initial render.
 * Pre-fetches user`s and project information.
 */
onBeforeMount(async () => {
    try {
        await Promise.all([
            usersStore.getUser(),
            abTestingStore.fetchValues(),
            usersStore.getSettings(),
        ]);
    } catch (error) {
        notify.notifyError(error, AnalyticsErrorEventSource.OVERALL_APP_WRAPPER_ERROR);
        setTimeout(async () => await router.push(RouteConfig.Login.path), 1000);

        return;
    }

    try {
        await billingStore.setupAccount();
    } catch (error) {
        error.message = `Unable to setup account. ${error.message}`;
        notify.notifyError(error, AnalyticsErrorEventSource.OVERALL_APP_WRAPPER_ERROR);
    }

    try {
        await billingStore.getCreditCards();
    } catch (error) {
        error.message = `Unable to get credit cards. ${error.message}`;
        notify.notifyError(error, AnalyticsErrorEventSource.OVERALL_APP_WRAPPER_ERROR);
    }

    selectProject(route.params.projectId as string);

    if (!agStore.state.accessGrantsWebWorker) await agStore.startWorker();
});
</script>
