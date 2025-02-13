// Copyright (C) 2023 Storj Labs, Inc.
// See LICENSE for copying information.

<template>
    <div v-if="isLoading" class="loading-overlay active">
        <div class="load" />
        <LoaderImage class="loading-icon" />
    </div>
    <div v-else class="all-dashboard">
        <div class="all-dashboard__bars">
            <BetaSatBar v-if="isBetaSatellite" />
            <MFARecoveryCodeBar v-if="showMFARecoveryCodeBar" :open-generate-modal="generateNewMFARecoveryCodes" />
        </div>

        <heading class="all-dashboard__heading" />

        <div class="all-dashboard__content" :class="{ 'no-x-padding': isMyProjectsPage }">
            <div class="all-dashboard__content__divider" />

            <router-view />

            <limit-warning-modal
                v-if="isHundredLimitModalShown && !isLoading"
                severity="critical"
                :on-close="() => setIsHundredLimitModalShown(false)"
                :title="limitState.hundredModalTitle"
                :limit-type="limitState.hundredModalLimitType"
                :on-upgrade="togglePMModal"
            />
            <limit-warning-modal
                v-if="isEightyLimitModalShown && !isLoading"
                severity="warning"
                :on-close="() => setIsEightyLimitModalShown(false)"
                :title="limitState.eightyModalTitle"
                :limit-type="limitState.eightyModalLimitType"
                :on-upgrade="togglePMModal"
            />
            <AllModals />
        </div>
        <InactivityModal
            v-if="inactivityModalShown"
            :on-continue="() => refreshSession(true)"
            :on-logout="handleInactive"
            :on-close="closeInactivityModal"
            :initial-seconds="inactivityModalTime / 1000"
        />
        <SessionExpiredModal v-if="sessionExpiredModalShown" :on-redirect="redirectToLogin" />
    </div>
</template>

<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue';
import { useRoute, useRouter } from 'vue-router';

import { MODALS } from '@/utils/constants/appStatePopUps';
import { User } from '@/types/users';
import {
    AnalyticsErrorEventSource,
} from '@/utils/constants/analyticsEventNames';
import { useNotify } from '@/utils/hooks';
import { RouteConfig } from '@/types/router';
import { ErrorUnauthorized } from '@/api/errors/ErrorUnauthorized';
import { FetchState } from '@/utils/constants/fetchStateEnum';
import { LocalData } from '@/utils/localData';
import { CouponType } from '@/types/coupons';
import { AuthHttpApi } from '@/api/auth';
import Heading from '@/views/all-dashboard/components/Heading.vue';
import { useABTestingStore } from '@/store/modules/abTestingStore';
import { useUsersStore } from '@/store/modules/usersStore';
import { useProjectMembersStore } from '@/store/modules/projectMembersStore';
import { useBillingStore } from '@/store/modules/billingStore';
import { useAppStore } from '@/store/modules/appStore';
import { useAccessGrantsStore } from '@/store/modules/accessGrantsStore';
import { useBucketsStore } from '@/store/modules/bucketsStore';
import { useProjectsStore } from '@/store/modules/projectsStore';
import { useNotificationsStore } from '@/store/modules/notificationsStore';
import { useObjectBrowserStore } from '@/store/modules/objectBrowserStore';
import { useConfigStore } from '@/store/modules/configStore';
import { useAnalyticsStore } from '@/store/modules/analyticsStore';

import InactivityModal from '@/components/modals/InactivityModal.vue';
import SessionExpiredModal from '@/components/modals/SessionExpiredModal.vue';
import BetaSatBar from '@/components/infoBars/BetaSatBar.vue';
import MFARecoveryCodeBar from '@/components/infoBars/MFARecoveryCodeBar.vue';
import AllModals from '@/components/modals/AllModals.vue';
import LimitWarningModal from '@/components/modals/LimitWarningModal.vue';

import LoaderImage from '@/../static/images/common/loadIcon.svg';

const router = useRouter();
const route = useRoute();
const notify = useNotify();

const analyticsStore = useAnalyticsStore();
const configStore = useConfigStore();
const bucketsStore = useBucketsStore();
const pmStore = useProjectMembersStore();
const usersStore = useUsersStore();
const abTestingStore = useABTestingStore();
const billingStore = useBillingStore();
const agStore = useAccessGrantsStore();
const appStore = useAppStore();
const projectsStore = useProjectsStore();
const notificationsStore = useNotificationsStore();
const obStore = useObjectBrowserStore();

const auth: AuthHttpApi = new AuthHttpApi();

const inactivityModalTime = 60000;
// Minimum number of recovery codes before the recovery code warning bar is shown.
const recoveryCodeWarningThreshold = 4;

const inactivityTimerId = ref<ReturnType<typeof setTimeout> | null>(null);
const sessionRefreshTimerId = ref<ReturnType<typeof setTimeout> | null>(null);
const debugTimerId = ref<ReturnType<typeof setTimeout> | null>(null);
const debugTimerText = ref<string>('');
const resetActivityEvents: string[] = ['keypress', 'mousemove', 'mousedown', 'touchmove'];
const inactivityModalShown = ref<boolean>(false);
const sessionExpiredModalShown = ref<boolean>(false);
const isSessionActive = ref<boolean>(false);
const isSessionRefreshing = ref<boolean>(false);
const isHundredLimitModalShown = ref<boolean>(false);
const isEightyLimitModalShown = ref<boolean>(false);

/**
 * Returns the session duration from the store.
 */
const sessionDuration = computed((): number => {
    const duration =  (usersStore.state.settings.sessionDuration?.fullSeconds || configStore.state.config.inactivityTimerDuration) * 1000;
    const maxTimeout = 2.1427e+9; // 24.8 days https://developer.mozilla.org/en-US/docs/Web/API/setTimeout#maximum_delay_value
    if (duration > maxTimeout) {
        return maxTimeout;
    }
    return duration;
});

const isMyProjectsPage = computed((): boolean => {
    return route.path === RouteConfig.AllProjectsDashboard.path;
});

/**
 * Returns the session refresh interval from the store.
 */
const sessionRefreshInterval = computed((): number => {
    return sessionDuration.value / 2;
});

/**
 * Indicates whether to display the session timer for debugging.
 */
const debugTimerShown = computed((): boolean => {
    return configStore.state.config.inactivityTimerViewerEnabled;
});

/**
 * Indicates if account was frozen due to billing issues.
 */
const isAccountFrozen = computed((): boolean => {
    return usersStore.state.user.freezeStatus.frozen;
});

/**
 * Returns all needed information for limit modal when bandwidth or storage close to limits.
 */
 type LimitedState = {
    eightyIsShown: boolean;
    hundredIsShown: boolean;
    eightyLabel: string;
    eightyModalLimitType: string;
    eightyModalTitle: string;
    hundredLabel: string;
    hundredModalTitle: string;
    hundredModalLimitType: string;
}

const limitState = computed((): LimitedState => {
    const result: LimitedState = {
        eightyIsShown: false,
        hundredIsShown: false,
        eightyLabel: '',
        eightyModalLimitType: '',
        eightyModalTitle: '',
        hundredLabel: '',
        hundredModalTitle: '',
        hundredModalLimitType: '',
    };

    if (usersStore.state.user.paidTier || isAccountFrozen.value) {
        return result;
    }

    const currentLimits = projectsStore.state.currentLimits;

    const limitTypeArr = [
        { name: 'egress', usedPercent: Math.round(currentLimits.bandwidthUsed * 100 / currentLimits.bandwidthLimit) },
        { name: 'storage', usedPercent: Math.round(currentLimits.storageUsed * 100 / currentLimits.storageLimit) },
        { name: 'segment', usedPercent: Math.round(currentLimits.segmentUsed * 100 / currentLimits.segmentLimit) },
    ];

    const hundredPercent: string[] = [];
    const eightyPercent: string[] = [];

    limitTypeArr.forEach((limitType) => {
        if (limitType.usedPercent >= 80) {
            if (limitType.usedPercent >= 100) {
                hundredPercent.push(limitType.name);
            } else {
                eightyPercent.push(limitType.name);
            }
        }
    });

    if (eightyPercent.length !== 0) {
        result.eightyIsShown = true;

        const eightyPercentString = eightyPercent.join(' and ');

        result.eightyLabel = `You've used 80% of your ${eightyPercentString} limit. Avoid interrupting your usage by upgrading your account.`;
        result.eightyModalTitle = `80% ${eightyPercentString} limit used`;
        result.eightyModalLimitType = eightyPercentString;
    }

    if (hundredPercent.length !== 0) {
        result.hundredIsShown = true;

        const hundredPercentString = hundredPercent.join(' and ');

        result.hundredLabel = `URGENT: You’ve reached the ${hundredPercentString} limit for your project. Upgrade to avoid any service interruptions.`;
        result.hundredModalTitle = `URGENT: You’ve reached the ${hundredPercentString} limit for your project.`;
        result.hundredModalLimitType = hundredPercentString;
    }

    return result;
});

/**
 * Indicates if satellite is in beta.
 */
const isBetaSatellite = computed((): boolean => {
    return configStore.state.config.isBetaSatellite;
});

/**
 * Indicates if loading screen is active.
 */
const isLoading = computed((): boolean => {
    return appStore.state.fetchState === FetchState.LOADING;
});

/**
 * Indicates whether the MFA recovery code warning bar should be shown.
 */
const showMFARecoveryCodeBar = computed((): boolean => {
    const user: User = usersStore.state.user;
    return user.isMFAEnabled && user.mfaRecoveryCodeCount < recoveryCodeWarningThreshold;
});

function setIsEightyLimitModalShown(value: boolean): void {
    isEightyLimitModalShown.value = value;
}

function setIsHundredLimitModalShown(value: boolean): void {
    isHundredLimitModalShown.value = value;
}

/**
 * Redirects to log in screen.
 */
function redirectToLogin(): void {
    analyticsStore.pageVisit(RouteConfig.Login.path);
    router.push(RouteConfig.Login.path);

    sessionExpiredModalShown.value = false;
}

/**
 * Clears pinia stores and timers.
 */
async function clearStoreAndTimers(): Promise<void> {
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
        abTestingStore.reset(),
        obStore.clear(),
    ]);

    resetActivityEvents.forEach((eventName: string) => {
        document.removeEventListener(eventName, onSessionActivity);
    });
    clearSessionTimers();
    inactivityModalShown.value = false;
    sessionExpiredModalShown.value = true;
}

/**
 * Performs logout and cleans event listeners and session timers.
 */
async function handleInactive(): Promise<void> {
    await clearStoreAndTimers();

    try {
        await auth.logout();
    } catch (error) {
        if (error instanceof ErrorUnauthorized) return;

        notify.notifyError(error, AnalyticsErrorEventSource.OVERALL_SESSION_EXPIRED_ERROR);
    }
}

/**
 * Generates new MFA recovery codes and toggles popup visibility.
 */
async function generateNewMFARecoveryCodes(): Promise<void> {
    try {
        await usersStore.generateUserMFARecoveryCodes();
        toggleMFARecoveryModal();
    } catch (error) {
        notify.notifyError(error, AnalyticsErrorEventSource.ALL_PROJECT_DASHBOARD);
    }
}

/**
 * Toggles MFA recovery modal visibility.
 */
function toggleMFARecoveryModal(): void {
    appStore.updateActiveModal(MODALS.mfaRecovery);
}

/**
 * Disables session inactivity modal visibility.
 */
function closeInactivityModal(): void {
    inactivityModalShown.value = false;
}

/**
 * Opens add payment method modal.
 */
function togglePMModal(): void {
    isHundredLimitModalShown.value = false;
    isEightyLimitModalShown.value = false;

    if (!usersStore.state.user.paidTier) {
        appStore.updateActiveModal(MODALS.upgradeAccount);
    }
}

/**
 * Clears timers associated with session refreshing and inactivity.
 */
function clearSessionTimers(): void {
    [inactivityTimerId.value, sessionRefreshTimerId.value, debugTimerId.value].forEach(id => {
        if (id !== null) clearTimeout(id);
    });
}

/**
 * Adds DOM event listeners and starts session timers.
 */
function setupSessionTimers(): void {
    if (!configStore.state.config.inactivityTimerEnabled) return;

    const expiresAt = LocalData.getSessionExpirationDate();

    if (expiresAt) {
        resetActivityEvents.forEach((eventName: string) => {
            document.addEventListener(eventName, onSessionActivity, false);
        });

        if (expiresAt.getTime() - sessionDuration.value + sessionRefreshInterval.value < Date.now()) {
            refreshSession();
        }

        restartSessionTimers();
    }
}

/**
 * Restarts timers associated with session refreshing and inactivity.
 */
function restartSessionTimers(): void {
    sessionRefreshTimerId.value = setTimeout(async () => {
        sessionRefreshTimerId.value = null;
        if (isSessionActive.value) {
            await refreshSession();
        }
    }, sessionRefreshInterval.value);

    inactivityTimerId.value = setTimeout(async () => {
        if (obStore.uploadingLength) {
            await refreshSession();
            return;
        }

        if (isSessionActive.value) return;
        inactivityModalShown.value = true;
        inactivityTimerId.value = setTimeout(async () => {
            await clearStoreAndTimers();
            notify.notify('Your session was timed out.');
        }, inactivityModalTime);
    }, sessionDuration.value - inactivityModalTime);

    if (!debugTimerShown.value) return;

    const debugTimer = () => {
        const expiresAt = LocalData.getSessionExpirationDate();

        if (expiresAt) {
            const ms = Math.max(0, expiresAt.getTime() - Date.now());
            const secs = Math.floor(ms / 1000) % 60;

            debugTimerText.value = `${Math.floor(ms / 60000)}:${(secs < 10 ? '0' : '') + secs}`;

            if (ms > 1000) {
                debugTimerId.value = setTimeout(debugTimer, 1000);
            }
        }
    };

    debugTimer();
}

/**
 * Refreshes session and resets session timers.
 * @param manual - whether the user manually refreshed session. i.e.: clicked "Stay Logged In".
 */
async function refreshSession(manual = false): Promise<void> {
    isSessionRefreshing.value = true;

    try {
        LocalData.setSessionExpirationDate(await auth.refreshSession());
    } catch (error) {
        error.message = (error instanceof ErrorUnauthorized) ? 'Your session was timed out.' : error.message;
        notify.notifyError(error, AnalyticsErrorEventSource.ALL_PROJECT_DASHBOARD);
        await handleInactive();
        isSessionRefreshing.value = false;
        return;
    }

    clearSessionTimers();
    restartSessionTimers();
    inactivityModalShown.value = false;
    isSessionActive.value = false;
    isSessionRefreshing.value = false;

    if (manual && !usersStore.state.settings.sessionDuration) {
        appStore.updateActiveModal(MODALS.editSessionTimeout);
    }
}

/**
 * Resets inactivity timer and refreshes session if necessary.
 */
async function onSessionActivity(): Promise<void> {
    if (inactivityModalShown.value || isSessionActive.value) return;

    if (sessionRefreshTimerId.value === null && !isSessionRefreshing.value) {
        await refreshSession();
    }

    isSessionActive.value = true;
}

/**
 * Lifecycle hook after initial render.
 * Pre-fetches user`s and project information.
 */
onMounted(async () => {
    usersStore.$onAction(({ name, after, args }) => {
        if (name === 'clear') clearSessionTimers();
        else if (name === 'updateSettings') {
            if (args[0].sessionDuration && args[0].sessionDuration !== usersStore.state.settings.sessionDuration?.nanoseconds) {
                after((_) => refreshSession());
            }
        }
    });

    try {
        await Promise.all([
            usersStore.getUser(),
            abTestingStore.fetchValues(),
            usersStore.getSettings(),
        ]);

        setupSessionTimers();
    } catch (error) {
        if (!(error instanceof ErrorUnauthorized)) {
            appStore.changeState(FetchState.ERROR);
            notify.notifyError(error, AnalyticsErrorEventSource.ALL_PROJECT_DASHBOARD);
        }

        setTimeout(async () => await router.push(RouteConfig.Login.path), 1000);

        return;
    }

    try {
        agStore.stopWorker();
        await agStore.startWorker();
    } catch (error) {
        notify.error(`Unable to set access grants wizard. ${error.message}`, AnalyticsErrorEventSource.ALL_PROJECT_DASHBOARD);
    }

    try {
        const couponType = await billingStore.setupAccount();
        if (couponType === CouponType.NoCoupon) {
            notify.error(`The coupon code was invalid, and could not be applied to your account`, AnalyticsErrorEventSource.ALL_PROJECT_DASHBOARD);
        }

        if (couponType === CouponType.SignupCoupon) {
            notify.success(`The coupon code was added successfully`);
        }
    } catch (error) {
        error.message = `Unable to setup account. ${error.message}`;
        notify.notifyError(error, AnalyticsErrorEventSource.ALL_PROJECT_DASHBOARD);
    }

    try {
        await billingStore.getCreditCards();
    } catch (error) {
        error.message = `Unable to get credit cards. ${error.message}`;
        notify.notifyError(error, AnalyticsErrorEventSource.ALL_PROJECT_DASHBOARD);
    }

    try {
        await projectsStore.getUserInvitations();
    } catch (error) {
        error.message = `Unable to get project invitations. ${error.message}`;
        notify.notifyError(error, AnalyticsErrorEventSource.ALL_PROJECT_DASHBOARD);
    }

    try {
        await projectsStore.getProjects();
    } catch (error) {
        return;
    }

    appStore.changeState(FetchState.LOADED);

    if (usersStore.shouldOnboard && !appStore.state.hasShownPricingPlan) {
        appStore.setHasShownPricingPlan(true);
        // if the user is not legible for a pricing plan, they'll automatically be
        // navigated back to all projects dashboard.
        analyticsStore.pageVisit(RouteConfig.OnboardingTour.with(RouteConfig.PricingPlanStep).path);
        await router.push(RouteConfig.OnboardingTour.with(RouteConfig.PricingPlanStep).path);
    }
});

onBeforeUnmount(() => {
    clearSessionTimers();
    resetActivityEvents.forEach((eventName: string) => {
        document.removeEventListener(eventName, onSessionActivity);
    });
});
</script>

<style scoped lang="scss">
@keyframes rotate {

    from {
        transform: rotate(0deg);
    }

    to {
        transform: rotate(360deg);
    }
}

.no-x-padding {
    padding-left: 0 !important;
    padding-right: 0 !important;
}

.all-dashboard {
    box-sizing: border-box;
    overflow-y: auto;
    width: 100%;
    height: 100%;
    background: var(--c-grey-1);

    &__bars {
        display: contents;
        position: fixed;
        width: 100%;
        top: 0;
        z-index: 1000;
    }

    &__heading {
        margin: 50px auto 0;
        padding: 0 20px;
        max-width: 1200px;
        box-sizing: border-box;

        @media screen and (width <= 500px) {
            margin-top: 0;
            padding: 0;
        }
    }

    &__content {
        padding: 0 20px 50px;
        margin: 0 auto;
        max-width: 1200px;
        box-sizing: border-box;

        &__divider {
            margin: 20px 0;
            border: 0.5px solid var(--c-grey-2);

            @media screen and (width <= 500px) {
                display: none;
            }
        }
    }
}

.load {
    width: 90px;
    height: 90px;
    margin: auto 0;
    border: solid 3px var(--c-blue-3);
    border-radius: 50%;
    border-right-color: transparent;
    border-bottom-color: transparent;
    border-left-color: transparent;
    transition: all 0.5s ease-in;
    animation-name: rotate;
    animation-duration: 1.2s;
    animation-iteration-count: infinite;
    animation-timing-function: linear;
}

.loading-overlay {
    display: flex;
    justify-content: center;
    align-items: center;
    position: absolute;
    inset: 0;
    background-color: var(--c-white);
    visibility: hidden;
    opacity: 0;
    transition: all 0.5s linear;
}

.loading-overlay.active {
    visibility: visible;
    opacity: 1;
}

.loading-icon {
    position: absolute;
    inset: 0;
    margin: auto;
}

:deep(div.account-area-container) {
    padding: 0;
}
</style>
