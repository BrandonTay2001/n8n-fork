<script setup lang="ts">
import { ref, computed, watch, onBeforeMount, onBeforeUnmount, onMounted } from 'vue';
import type { RouteLocation, RouteLocationRaw } from 'vue-router';
import { useRouter, useRoute } from 'vue-router';
import WorkflowDetails from '@/components/MainHeader/WorkflowDetails.vue';
import TabBar from '@/components/MainHeader/TabBar.vue';
import {
	LOCAL_STORAGE_HIDE_GITHUB_STAR_BUTTON,
	MAIN_HEADER_TABS,
	PLACEHOLDER_EMPTY_WORKFLOW_ID,
	STICKY_NODE_TYPE,
	VIEWS,
	WORKFLOW_EVALUATION_EXPERIMENT,
} from '@/constants';
import { useI18n } from '@/composables/useI18n';
import { useNDVStore } from '@/stores/ndv.store';
import { useSourceControlStore } from '@/stores/sourceControl.store';
import { useUIStore } from '@/stores/ui.store';
import { useWorkflowsStore } from '@/stores/workflows.store';
import { useExecutionsStore } from '@/stores/executions.store';
import { useSettingsStore } from '@/stores/settings.store';
import { usePushConnection } from '@/composables/usePushConnection';
import { usePostHog } from '@/stores/posthog.store';

import GithubButton from 'vue-github-button';
import { useLocalStorage } from '@vueuse/core';

const router = useRouter();
const route = useRoute();
const locale = useI18n();
const pushConnection = usePushConnection({ router });
const ndvStore = useNDVStore();
const uiStore = useUIStore();
const sourceControlStore = useSourceControlStore();
const workflowsStore = useWorkflowsStore();
const executionsStore = useExecutionsStore();
const settingsStore = useSettingsStore();
const posthogStore = usePostHog();

const activeHeaderTab = ref(MAIN_HEADER_TABS.WORKFLOW);
const workflowToReturnTo = ref('');
const executionToReturnTo = ref('');
const dirtyState = ref(false);
const githubButtonHidden = useLocalStorage(LOCAL_STORAGE_HIDE_GITHUB_STAR_BUTTON, false);

const tabBarItems = computed(() => {
	const items = [
		{ value: MAIN_HEADER_TABS.WORKFLOW, label: locale.baseText('generic.editor') },
		{ value: MAIN_HEADER_TABS.EXECUTIONS, label: locale.baseText('generic.executions') },
	];

	if (posthogStore.isFeatureEnabled(WORKFLOW_EVALUATION_EXPERIMENT)) {
		items.push({
			value: MAIN_HEADER_TABS.TEST_DEFINITION,
			label: locale.baseText('generic.tests'),
		});
	}
	return items;
});

const activeNode = computed(() => ndvStore.activeNode);
const hideMenuBar = computed(() =>
	Boolean(activeNode.value && activeNode.value.type !== STICKY_NODE_TYPE),
);
const workflow = computed(() => workflowsStore.workflow);
const workflowId = computed(() =>
	String(router.currentRoute.value.params.name || workflowsStore.workflowId),
);
const onWorkflowPage = computed(() => !!(route.meta.nodeView || route.meta.keepWorkflowAlive));
const readOnly = computed(() => sourceControlStore.preferences.branchReadOnly);
const isEnterprise = computed(
	() => settingsStore.isQueueModeEnabled && settingsStore.isWorkerViewAvailable,
);
const showGitHubButton = computed(
	() => !isEnterprise.value && !settingsStore.settings.inE2ETests && !githubButtonHidden.value,
);

watch(route, (to, from) => {
	syncTabsWithRoute(to, from);
});

onBeforeMount(() => {
	pushConnection.initialize();
});

onBeforeUnmount(() => {
	pushConnection.terminate();
});

onMounted(async () => {
	dirtyState.value = uiStore.stateIsDirty;
	syncTabsWithRoute(route);
});

function syncTabsWithRoute(to: RouteLocation, from?: RouteLocation): void {
	if (to.matched.some((record) => record.name === VIEWS.TEST_DEFINITION)) {
		activeHeaderTab.value = MAIN_HEADER_TABS.TEST_DEFINITION;
	}
	if (
		to.name === VIEWS.EXECUTION_HOME ||
		to.name === VIEWS.WORKFLOW_EXECUTIONS ||
		to.name === VIEWS.EXECUTION_PREVIEW
	) {
		activeHeaderTab.value = MAIN_HEADER_TABS.EXECUTIONS;
	} else if (
		to.name === VIEWS.WORKFLOW ||
		to.name === VIEWS.NEW_WORKFLOW ||
		to.name === VIEWS.EXECUTION_DEBUG
	) {
		activeHeaderTab.value = MAIN_HEADER_TABS.WORKFLOW;
	}

	if (to.params.name !== 'new' && typeof to.params.name === 'string') {
		workflowToReturnTo.value = to.params.name;
	}

	if (
		from?.name === VIEWS.EXECUTION_PREVIEW &&
		to.params.name === from.params.name &&
		typeof from.params.executionId === 'string'
	) {
		executionToReturnTo.value = from.params.executionId;
	}
}

function onTabSelected(tab: MAIN_HEADER_TABS, event: MouseEvent) {
	const openInNewTab = event.ctrlKey || event.metaKey;

	switch (tab) {
		case MAIN_HEADER_TABS.WORKFLOW:
			void navigateToWorkflowView(openInNewTab);
			break;

		case MAIN_HEADER_TABS.EXECUTIONS:
			void navigateToExecutionsView(openInNewTab);
			break;

		case MAIN_HEADER_TABS.TEST_DEFINITION:
			activeHeaderTab.value = MAIN_HEADER_TABS.TEST_DEFINITION;
			void router.push({ name: VIEWS.TEST_DEFINITION });
			break;

		default:
			break;
	}
}

async function navigateToWorkflowView(openInNewTab: boolean) {
	let routeToNavigateTo: RouteLocationRaw;
	if (!['', 'new', PLACEHOLDER_EMPTY_WORKFLOW_ID].includes(workflowToReturnTo.value)) {
		routeToNavigateTo = {
			name: VIEWS.WORKFLOW,
			params: { name: workflowToReturnTo.value },
		};
	} else {
		routeToNavigateTo = { name: VIEWS.NEW_WORKFLOW };
	}

	if (openInNewTab) {
		const { href } = router.resolve(routeToNavigateTo);
		window.open(href, '_blank');
	} else if (route.name !== routeToNavigateTo.name) {
		if (route.name === VIEWS.NEW_WORKFLOW) {
			uiStore.stateIsDirty = dirtyState.value;
		}
		activeHeaderTab.value = MAIN_HEADER_TABS.WORKFLOW;
		await router.push(routeToNavigateTo);
	}
}

async function navigateToExecutionsView(openInNewTab: boolean) {
	const routeWorkflowId =
		workflowId.value === PLACEHOLDER_EMPTY_WORKFLOW_ID ? 'new' : workflowId.value;
	const executionToReturnToValue = executionsStore.activeExecution?.id || executionToReturnTo.value;
	const routeToNavigateTo: RouteLocationRaw = executionToReturnToValue
		? {
				name: VIEWS.EXECUTION_PREVIEW,
				params: { name: routeWorkflowId, executionId: executionToReturnToValue },
			}
		: {
				name: VIEWS.EXECUTION_HOME,
				params: { name: routeWorkflowId },
			};

	if (openInNewTab) {
		const { href } = router.resolve(routeToNavigateTo);
		window.open(href, '_blank');
	} else if (route.name !== routeToNavigateTo.name) {
		dirtyState.value = uiStore.stateIsDirty;
		workflowToReturnTo.value = workflowId.value;
		activeHeaderTab.value = MAIN_HEADER_TABS.EXECUTIONS;
		await router.push(routeToNavigateTo);
	}
}

function hideGithubButton() {
	githubButtonHidden.value = true;
}
</script>

<template>
	<div class="container">
		<div :class="{ 'main-header': true, expanded: !uiStore.sidebarMenuCollapsed }">
			<div v-show="!hideMenuBar" class="top-menu">
				<WorkflowDetails
					v-if="workflow?.name"
					:id="workflow.id"
					:tags="workflow.tags"
					:name="workflow.name"
					:meta="workflow.meta"
					:scopes="workflow.scopes"
					:active="workflow.active"
					:read-only="readOnly"
				/>
			</div>
		</div>
	</div>
</template>

<style lang="scss">
.container {
	display: flex;
	position: relative;
	width: 100%;
	align-items: center;
}

.main-header {
	background-color: var(--color-background-xlight);
	width: 100%;
	box-sizing: border-box;
}

.top-menu {
	position: relative;
	display: flex;
	align-items: center;
	font-size: 0.9em;
	font-weight: 400;
	padding-top: 15px;
	padding-bottom: 15px;
	margin-right: 20px;
}

.github-button {
	display: flex;
	position: relative;
	align-items: center;
	align-self: stretch;
	justify-content: center;
	min-width: 170px;
	padding-top: 2px;
	padding-left: var(--spacing-m);
	padding-right: var(--spacing-m);
	background-color: var(--color-background-xlight);
	border-bottom: var(--border-width-base) var(--border-style-base) var(--color-foreground-base);
	border-left: var(--border-width-base) var(--border-style-base) var(--color-foreground-base);
}

.close-github-button {
	display: none;
	position: absolute;
	right: 0;
	top: 0;
	transform: translate(50%, -46%);
	color: var(--color-foreground-xdark);
	background-color: var(--color-background-xlight);
	border-radius: 100%;
	cursor: pointer;

	&:hover {
		color: var(--prim-color-primary-shade-100);
	}
}
.github-button-container {
	position: relative;
}

.github-button:hover .close-github-button {
	display: block;
}
</style>
