// Copyright (C) 2023 Storj Labs, Inc.
// See LICENSE for copying information.

<template>
    <v-app-bar :elevation="0">
        <v-app-bar-nav-icon
            v-if="showNavDrawerButton"
            variant="text"
            color="default"
            class="ml-1"
            size="x-small"
            density="comfortable"
            @click.stop="appStore.toggleNavigationDrawer()"
        />

        <v-app-bar-title class="mx-1 flex-initial" :class="{ 'ml-4': !showNavDrawerButton }">
            <router-link to="/projects">
                <v-img
                    v-if="theme.global.current.value.dark"
                    src="@poc/assets/logo-dark.svg"
                    width="120"
                    alt="Storj Logo"
                />
                <v-img
                    v-else
                    src="@poc/assets/logo.svg"
                    width="120"
                    alt="Storj Logo"
                />
            </router-link>
        </v-app-bar-title>

        <template #append>
            <v-menu offset-y width="200" class="rounded-xl">
                <template #activator="{ props: activatorProps }">
                    <!-- Theme Toggle Light/Dark Mode -->
                    <v-btn-toggle
                        v-model="activeTheme"
                        mandatory
                        border
                        inset
                        density="comfortable"
                        class="pa-1"
                    >
                        <v-tooltip text="Light Theme" location="bottom">
                            <template #activator="{ props: darkProps }">
                                <v-btn
                                    v-bind="darkProps"
                                    icon="mdi-weather-sunny"
                                    size="small"
                                    rounded="xl"
                                    aria-label="Toggle Light Theme"
                                    @click="toggleTheme('light')"
                                />
                            </template>
                        </v-tooltip>

                        <v-tooltip text="Dark Theme" location="bottom">
                            <template #activator="{ props: lightProps }">
                                <v-btn
                                    v-bind="lightProps"
                                    icon="mdi-weather-night"
                                    size="small"
                                    rounded="xl"
                                    aria-label="Toggle Dark Theme"
                                    @click="toggleTheme('dark')"
                                />
                            </template>
                        </v-tooltip>
                    </v-btn-toggle>

                    <!-- My Account Dropdown Button -->
                    <v-btn
                        v-bind="activatorProps"
                        variant="outlined"
                        color="default"
                        class="ml-4 font-weight-medium"
                        density="comfortable"
                    >
                        <template #append>
                            <img src="@poc/assets/icon-dropdown.svg" alt="Account Dropdown">
                        </template>
                        My Account
                    </v-btn>
                </template>

                <!-- My Account Menu -->
                <v-list class="px-2">
                    <v-list-item class="py-2 rounded-lg">
                        <template #prepend>
                            <img src="@poc/assets/icon-satellite.svg" alt="Region">
                        </template>
                        <v-list-item-title class="text-body-2 ml-3">Region</v-list-item-title>
                        <v-list-item-subtitle class="ml-3">
                            North America 1
                        </v-list-item-subtitle>
                    </v-list-item>

                    <v-divider class="my-2" />

                    <v-list-item link class="my-1 rounded-lg">
                        <template #prepend>
                            <img src="@poc/assets/icon-upgrade.svg" alt="Upgrade">
                        </template>
                        <v-list-item-title class="text-body-2 ml-3">
                            Upgrade
                        </v-list-item-title>
                    </v-list-item>

                    <v-list-item link class="my-1 rounded-lg" router-link to="/account/billing">
                        <template #prepend>
                            <img src="@poc/assets/icon-card.svg" alt="Billing">
                        </template>
                        <v-list-item-title class="text-body-2 ml-3">
                            Billing
                        </v-list-item-title>
                    </v-list-item>

                    <v-list-item link class="my-1 rounded-lg" router-link to="/account/settings">
                        <template #prepend>
                            <img src="@poc/assets/icon-settings.svg" alt="Account Settings">
                        </template>
                        <v-list-item-title class="text-body-2 ml-3">
                            Settings
                        </v-list-item-title>
                    </v-list-item>
                    <v-list-item class="rounded-lg" link @click="onLogout">
                        <template #prepend>
                            <img src="@poc/assets/icon-logout.svg" alt="Log Out">
                        </template>
                        <v-list-item-title class="text-body-2 ml-3">
                            Sign Out
                        </v-list-item-title>
                    </v-list-item>
                </v-list>
            </v-menu>
        </template>
    </v-app-bar>
</template>

<script setup lang="ts">
import { ref, watch } from 'vue';
import { useRouter } from 'vue-router';
import { useTheme } from 'vuetify';
import {
    VAppBar,
    VAppBarNavIcon,
    VAppBarTitle,
    VImg,
    VMenu,
    VBtnToggle,
    VTooltip,
    VBtn,
    VList,
    VListItem,
    VListItemTitle,
    VListItemSubtitle,
    VDivider,
} from 'vuetify/components';

import { useAppStore } from '@poc/store/appStore';
import { AuthHttpApi } from '@/api/auth';
import { RouteConfig } from '@/types/router';
import { AnalyticsEvent } from '@/utils/constants/analyticsEventNames';
import { useNotify } from '@/utils/hooks';
import { useUsersStore } from '@/store/modules/usersStore';
import { useProjectMembersStore } from '@/store/modules/projectMembersStore';
import { useBillingStore } from '@/store/modules/billingStore';
import { useAccessGrantsStore } from '@/store/modules/accessGrantsStore';
import { useBucketsStore } from '@/store/modules/bucketsStore';
import { useProjectsStore } from '@/store/modules/projectsStore';
import { useNotificationsStore } from '@/store/modules/notificationsStore';
import { useObjectBrowserStore } from '@/store/modules/objectBrowserStore';
import { useAnalyticsStore } from '@/store/modules/analyticsStore';

const activeTheme = ref<number>(0);
const theme = useTheme();

const analyticsStore = useAnalyticsStore();
const bucketsStore = useBucketsStore();
const appStore = useAppStore();
const agStore = useAccessGrantsStore();
const pmStore = useProjectMembersStore();
const billingStore = useBillingStore();
const usersStore = useUsersStore();
const notificationsStore = useNotificationsStore();
const projectsStore = useProjectsStore();
const obStore = useObjectBrowserStore();

const router = useRouter();
const notify = useNotify();

const auth: AuthHttpApi = new AuthHttpApi();

const props = withDefaults(defineProps<{
    showNavDrawerButton: boolean;
}>(), {
    showNavDrawerButton: false,
});

function toggleTheme(newTheme: string): void {
    if ((newTheme === 'dark' && theme.global.current.value.dark) || (newTheme === 'light' && !theme.global.current.value.dark)) {
        return;
    }
    theme.global.name.value = newTheme;
    localStorage.setItem('theme', newTheme);  // Store the selected theme in localStorage
}

watch(() => theme.global.current.value.dark, (newVal: boolean) => {
    activeTheme.value = newVal ? 1 : 0;
});

// Check for stored theme in localStorage. If none, default to 'light'
toggleTheme(localStorage.getItem('theme') || 'light');
activeTheme.value = theme.global.current.value.dark ? 1 : 0;

/**
 * Logs out user and navigates to login page.
 */
async function onLogout(): Promise<void> {
    await Promise.all([
        pmStore.clear(),
        projectsStore.clear(),
        usersStore.clear(),
        agStore.stopWorker(),
        agStore.clear(),
        notificationsStore.clear(),
        bucketsStore.clear(),
        appStore.clear(),
        billingStore.clear(),
        obStore.clear(),
    ]);

    try {
        analyticsStore.eventTriggered(AnalyticsEvent.LOGOUT_CLICKED);
        await auth.logout();
    } catch (error) {
        notify.error(error.message, null);
    }

    analyticsStore.pageVisit(RouteConfig.Login.path);
    await router.push(RouteConfig.Login.path);
    // TODO this reload will be unnecessary once vuetify poc has its own login and/or becomes the primary app
    location.reload();
}

</script>

<style scoped lang="scss">
.flex-initial {
    flex: initial;
}
</style>
