<script lang="ts" setup>
import { computed, onMounted, watch, ref, onBeforeUnmount } from 'vue';
import ResourcesListLayout from '@/components/layouts/ResourcesListLayout.vue';
import type {
	Resource,
	BaseFilters,
	FolderResource,
	WorkflowResource,
} from '@/components/layouts/ResourcesListLayout.vue';
import WorkflowCard from '@/components/WorkflowCard.vue';
import WorkflowTagsDropdown from '@/components/WorkflowTagsDropdown.vue';
import {
	EASY_AI_WORKFLOW_EXPERIMENT,
	EnterpriseEditionFeature,
	VIEWS,
	DEFAULT_WORKFLOW_PAGE_SIZE,
	MODAL_CONFIRM,
	COMMUNITY_PLUS_ENROLLMENT_MODAL,
} from '@/constants';
import type {
	IUser,
	UserAction,
	WorkflowListResource,
	WorkflowListItem,
	FolderPathItem,
	FolderListItem,
} from '@/Interface';
import { useUIStore } from '@/stores/ui.store';
import { useSettingsStore } from '@/stores/settings.store';
import { useUsersStore } from '@/stores/users.store';
import { useWorkflowsStore } from '@/stores/workflows.store';
import { useSourceControlStore } from '@/stores/sourceControl.store';
import { useTagsStore } from '@/stores/tags.store';
import { useProjectsStore } from '@/stores/projects.store';
import { getResourcePermissions } from '@/permissions';
import { usePostHog } from '@/stores/posthog.store';
import { useDocumentTitle } from '@/composables/useDocumentTitle';
import { useI18n } from '@/composables/useI18n';
import { type LocationQueryRaw, useRoute, useRouter } from 'vue-router';
import { useTelemetry } from '@/composables/useTelemetry';
import {
	N8nCard,
	N8nHeading,
	N8nIcon,
	N8nInputLabel,
	N8nOption,
	N8nSelect,
	N8nText,
} from '@n8n/design-system';
import ProjectHeader from '@/components/Projects/ProjectHeader.vue';
import { getEasyAiWorkflowJson } from '@/utils/easyAiWorkflowUtils';
import { useDebounce } from '@/composables/useDebounce';
import { createEventBus } from '@n8n/utils/event-bus';
import type { PathItem } from '@n8n/design-system/components/N8nBreadcrumbs/Breadcrumbs.vue';
import { type ProjectSharingData, ProjectTypes } from '@/types/projects.types';
import { FOLDER_LIST_ITEM_ACTIONS } from '@/components/Folders/constants';
import { debounce } from 'lodash-es';
import { useMessage } from '@/composables/useMessage';
import { useToast } from '@/composables/useToast';
import { useFoldersStore } from '@/stores/folders.store';
import { useFolders } from '@/composables/useFolders';
import { useUsageStore } from '@/stores/usage.store';
import { useInsightsStore } from '@/features/insights/insights.store';
import InsightsSummary from '@/features/insights/components/InsightsSummary.vue';
import { useOverview } from '@/composables/useOverview';
import { PROJECT_ROOT } from 'n8n-workflow';

const SEARCH_DEBOUNCE_TIME = 300;
const FILTERS_DEBOUNCE_TIME = 100;

interface Filters extends BaseFilters {
	status: string | boolean;
	tags: string[];
}

const StatusFilter = {
	ACTIVE: 'active',
	DEACTIVATED: 'deactivated',
	ALL: '',
};

/** Maps sort values from the ResourcesListLayout component to values expected by workflows endpoint */
const WORKFLOWS_SORT_MAP = {
	lastUpdated: 'updatedAt:desc',
	lastCreated: 'createdAt:desc',
	nameAsc: 'name:asc',
	nameDesc: 'name:desc',
} as const;

const i18n = useI18n();
const route = useRoute();
const router = useRouter();
const message = useMessage();
const toast = useToast();
const folderHelpers = useFolders();

const sourceControlStore = useSourceControlStore();
const usersStore = useUsersStore();
const workflowsStore = useWorkflowsStore();
const settingsStore = useSettingsStore();
const posthogStore = usePostHog();
const projectsStore = useProjectsStore();
const telemetry = useTelemetry();
const uiStore = useUIStore();
const tagsStore = useTagsStore();
const foldersStore = useFoldersStore();
const usageStore = useUsageStore();
const insightsStore = useInsightsStore();

const documentTitle = useDocumentTitle();
const { callDebounced } = useDebounce();
const overview = useOverview();

const loading = ref(false);
const breadcrumbsLoading = ref(false);
const filters = ref<Filters>({
	search: '',
	homeProject: '',
	status: StatusFilter.ALL,
	tags: [],
});

const workflowListEventBus = createEventBus();

const workflowsAndFolders = ref<WorkflowListResource[]>([]);

const easyAICalloutVisible = ref(true);

const currentPage = ref(1);
const pageSize = ref(DEFAULT_WORKFLOW_PAGE_SIZE);
const currentSort = ref('updatedAt:desc');

const currentFolderId = ref<string | null>(null);

const showCardsBadge = ref(false);

/**
 * Folder actions
 * These can appear on the list header, and then they are applied to current folder
 * or on each folder card, and then they are applied to the clicked folder
 * 'onlyAvailableOn' is used to specify where the action should be available, if not specified it will be available on both
 */
const folderActions = computed<
	Array<UserAction & { onlyAvailableOn?: 'mainBreadcrumbs' | 'card' }>
>(() => [
	{
		label: i18n.baseText('generic.open'),
		value: FOLDER_LIST_ITEM_ACTIONS.OPEN,
		disabled: false,
		onlyAvailableOn: 'card',
	},
	{
		label: i18n.baseText('folders.actions.create'),
		value: FOLDER_LIST_ITEM_ACTIONS.CREATE,
		disabled: readOnlyEnv.value || !hasPermissionToCreateFolders.value,
	},
	{
		label: i18n.baseText('folders.actions.create.workflow'),
		value: FOLDER_LIST_ITEM_ACTIONS.CREATE_WORKFLOW,
		disabled: readOnlyEnv.value || !hasPermissionToCreateWorkflows.value,
	},
	{
		label: i18n.baseText('generic.rename'),
		value: FOLDER_LIST_ITEM_ACTIONS.RENAME,
		disabled: readOnlyEnv.value || !hasPermissionToUpdateFolders.value,
	},
	{
		label: i18n.baseText('folders.actions.moveToFolder'),
		value: FOLDER_LIST_ITEM_ACTIONS.MOVE,
		disabled: readOnlyEnv.value || !hasPermissionToUpdateFolders.value,
	},
	{
		label: i18n.baseText('generic.delete'),
		value: FOLDER_LIST_ITEM_ACTIONS.DELETE,
		disabled: readOnlyEnv.value || !hasPermissionToDeleteFolders.value,
	},
]);

const folderCardActions = computed(() =>
	folderActions.value.filter(
		(action) => !action.onlyAvailableOn || action.onlyAvailableOn === 'card',
	),
);

const mainBreadcrumbsActions = computed(() =>
	folderActions.value.filter(
		(action) => !action.onlyAvailableOn || action.onlyAvailableOn === 'mainBreadcrumbs',
	),
);

const readOnlyEnv = computed(() => sourceControlStore.preferences.branchReadOnly);
const isOverviewPage = computed(() => route.name === VIEWS.WORKFLOWS);
const currentUser = computed(() => usersStore.currentUser ?? ({} as IUser));
const isShareable = computed(
	() => settingsStore.isEnterpriseFeatureEnabled[EnterpriseEditionFeature.Sharing],
);

const foldersEnabled = computed(() => {
	return settingsStore.isFoldersFeatureEnabled;
});

const teamProjectsEnabled = computed(() => {
	return projectsStore.isTeamProjectFeatureEnabled;
});

const showFolders = computed(() => {
	return foldersEnabled.value && !isOverviewPage.value;
});

const currentFolder = computed(() => {
	return currentFolderId.value ? foldersStore.breadcrumbsCache[currentFolderId.value] : null;
});

const hasPermissionToCreateFolders = computed(() => {
	if (!currentProject.value) return false;
	return getResourcePermissions(currentProject.value.scopes).folder.create === true;
});

const hasPermissionToUpdateFolders = computed(() => {
	if (!currentProject.value) return false;
	return getResourcePermissions(currentProject.value.scopes).folder.update === true;
});

const hasPermissionToDeleteFolders = computed(() => {
	if (!currentProject.value) return false;
	return getResourcePermissions(currentProject.value.scopes).folder.delete === true;
});

const hasPermissionToCreateWorkflows = computed(() => {
	if (!currentProject.value) return false;
	return getResourcePermissions(currentProject.value.scopes).workflow.create === true;
});

const currentProject = computed(() => projectsStore.currentProject);

const projectName = computed(() => {
	if (currentProject.value?.type === ProjectTypes.Personal) {
		return i18n.baseText('projects.menu.personal');
	}
	return currentProject.value?.name;
});

const currentParentName = computed(() => {
	if (currentFolder.value) {
		return currentFolder.value.name;
	}
	return projectName.value;
});

const workflowListResources = computed<Resource[]>(() => {
	const resources: Resource[] = (workflowsAndFolders.value || []).map((resource) => {
		if (resource.resource === 'folder') {
			return {
				resourceType: 'folder',
				id: resource.id,
				name: resource.name,
				createdAt: resource.createdAt.toString(),
				updatedAt: resource.updatedAt.toString(),
				homeProject: resource.homeProject,
				sharedWithProjects: resource.sharedWithProjects,
				workflowCount: resource.workflowCount,
				subFolderCount: resource.subFolderCount,
				parentFolder: resource.parentFolder,
			} as FolderResource;
		} else {
			return {
				resourceType: 'workflow',
				id: resource.id,
				name: resource.name,
				active: resource.active ?? false,
				updatedAt: resource.updatedAt.toString(),
				createdAt: resource.createdAt.toString(),
				homeProject: resource.homeProject,
				scopes: resource.scopes,
				sharedWithProjects: resource.sharedWithProjects,
				readOnly: !getResourcePermissions(resource.scopes).workflow.update,
				tags: resource.tags,
				parentFolder: resource.parentFolder,
			} as WorkflowResource;
		}
	});
	return resources;
});

const statusFilterOptions = computed(() => [
	{
		label: i18n.baseText('workflows.filters.status.all'),
		value: StatusFilter.ALL,
	},
	{
		label: i18n.baseText('workflows.filters.status.active'),
		value: StatusFilter.ACTIVE,
	},
	{
		label: i18n.baseText('workflows.filters.status.deactivated'),
		value: StatusFilter.DEACTIVATED,
	},
]);

const showEasyAIWorkflowCallout = computed(() => {
	const isEasyAIWorkflowExperimentEnabled =
		posthogStore.getVariant(EASY_AI_WORKFLOW_EXPERIMENT.name) ===
		EASY_AI_WORKFLOW_EXPERIMENT.variant;
	const easyAIWorkflowOnboardingDone = usersStore.isEasyAIWorkflowOnboardingDone;
	return isEasyAIWorkflowExperimentEnabled && !easyAIWorkflowOnboardingDone;
});

const projectPermissions = computed(() => {
	return getResourcePermissions(
		projectsStore.currentProject?.scopes ?? projectsStore.personalProject?.scopes,
	);
});

const emptyListDescription = computed(() => {
	if (readOnlyEnv.value) {
		return i18n.baseText('workflows.empty.description.readOnlyEnv');
	} else if (!projectPermissions.value.workflow.create) {
		return i18n.baseText('workflows.empty.description.noPermission');
	} else {
		return i18n.baseText('workflows.empty.description');
	}
});

const hasFilters = computed(() => {
	return !!(
		filters.value.search ||
		filters.value.status !== StatusFilter.ALL ||
		filters.value.tags.length
	);
});

const isSelfHostedDeployment = computed(() => settingsStore.deploymentType === 'default');

const canUserRegisterCommunityPlus = computed(
	() => getResourcePermissions(usersStore.currentUser?.globalScopes).community.register,
);

const showRegisteredCommunityCTA = computed(
	() => isSelfHostedDeployment.value && !foldersEnabled.value && canUserRegisterCommunityPlus.value,
);

/**
 * WATCHERS, STORE SUBSCRIPTIONS AND EVENT BUS HANDLERS
 */

watch(
	() => route.params?.projectId,
	async () => {
		await initialize();
	},
);

watch(
	() => route.params?.folderId,
	async (newVal) => {
		currentFolderId.value = newVal as string;
		filters.value.search = '';
		await fetchWorkflows();
	},
);

sourceControlStore.$onAction(({ name, after }) => {
	if (name !== 'pullWorkfolder') return;
	after(async () => await initialize());
});

const onWorkflowDeleted = async () => {
	await Promise.all([
		fetchWorkflows(),
		foldersStore.fetchTotalWorkflowsAndFoldersCount(route.params.projectId as string | undefined),
	]);
};

const onFolderDeleted = async (payload: {
	folderId: string;
	workflowCount: number;
	folderCount: number;
}) => {
	const folderInfo = foldersStore.getCachedFolder(payload.folderId);
	foldersStore.deleteFoldersFromCache([payload.folderId, folderInfo?.parentFolder ?? '']);
	// If the deleted folder is the current folder, navigate to the parent folder
	await foldersStore.fetchTotalWorkflowsAndFoldersCount(
		route.params.projectId as string | undefined,
	);

	if (currentFolderId.value === payload.folderId) {
		void router.push({
			name: VIEWS.PROJECTS_FOLDERS,
			params: { projectId: route.params.projectId, folderId: folderInfo?.parentFolder ?? '' },
		});
	} else {
		await fetchWorkflows();
	}
	telemetry.track('User deleted folder', {
		folder_id: payload.folderId,
		deleted_sub_folders: payload.folderCount,
		deleted_sub_workflows: payload.workflowCount,
	});
};

/**
 * LIFE-CYCLE HOOKS
 */

onMounted(async () => {
	documentTitle.set(i18n.baseText('workflows.heading'));
	void usersStore.showPersonalizationSurvey();

	workflowListEventBus.on('resource-moved', fetchWorkflows);
	workflowListEventBus.on('workflow-duplicated', fetchWorkflows);
	workflowListEventBus.on('folder-deleted', onFolderDeleted);
	workflowListEventBus.on('folder-moved', moveFolder);
	workflowListEventBus.on('workflow-moved', onWorkflowMoved);
});

onBeforeUnmount(() => {
	workflowListEventBus.off('resource-moved', fetchWorkflows);
	workflowListEventBus.off('workflow-duplicated', fetchWorkflows);
	workflowListEventBus.off('folder-deleted', onFolderDeleted);
	workflowListEventBus.off('folder-moved', moveFolder);
	workflowListEventBus.off('workflow-moved', onWorkflowMoved);
});

/**
 * METHODS
 */

// Main component fetch methods
const initialize = async () => {
	loading.value = true;
	await setFiltersFromQueryString();

	currentFolderId.value = route.params.folderId as string | null;
	const [, resourcesPage] = await Promise.all([
		usersStore.fetchUsers(),
		fetchWorkflows(),
		workflowsStore.fetchActiveWorkflows(),
		usageStore.getLicenseInfo(),
		foldersStore.fetchTotalWorkflowsAndFoldersCount(route.params.projectId as string | undefined),
	]);
	breadcrumbsLoading.value = false;
	workflowsAndFolders.value = resourcesPage;
	loading.value = false;
};

/**
 * Fetches:
 * - Current workflows and folders page
 * - Total count of workflows and folders in the current project
 * - Path to the current folder (if not cached)
 */
const fetchWorkflows = async () => {
	// We debounce here so that fast enough fetches don't trigger
	// the placeholder graphics for a few milliseconds, which would cause a flicker
	const delayedLoading = debounce(() => {
		loading.value = true;
	}, 300);

	const routeProjectId = route.params?.projectId as string | undefined;
	const homeProjectFilter = filters.value.homeProject || undefined;
	const parentFolder = (route.params?.folderId as string) || undefined;

	const tags = filters.value.tags.length
		? filters.value.tags.map((tagId) => tagsStore.tagsById[tagId]?.name)
		: [];
	const activeFilter =
		filters.value.status === StatusFilter.ALL
			? undefined
			: filters.value.status === StatusFilter.ACTIVE;

	// Only fetch folders if showFolders is enabled and there are not tags or active filter applied
	const fetchFolders = showFolders.value && !tags.length && activeFilter === undefined;

	try {
		const fetchedResources = await workflowsStore.fetchWorkflowsPage(
			routeProjectId ?? homeProjectFilter,
			currentPage.value,
			pageSize.value,
			currentSort.value,
			{
				name: filters.value.search || undefined,
				active: activeFilter,
				tags: tags.length ? tags : undefined,
				parentFolderId:
					parentFolder ??
					(isOverviewPage.value ? undefined : filters?.value.search ? undefined : PROJECT_ROOT), // Sending 0 will only show one level of folders
			},
			fetchFolders,
		);

		foldersStore.cacheFolders(
			fetchedResources
				.filter((resource) => resource.resource === 'folder')
				.map((r) => ({ id: r.id, name: r.name, parentFolder: r.parentFolder?.id })),
		);

		const isCurrentFolderCached = foldersStore.breadcrumbsCache[parentFolder ?? ''] !== undefined;
		const needToFetchFolderPath = parentFolder && !isCurrentFolderCached && routeProjectId;

		if (needToFetchFolderPath) {
			breadcrumbsLoading.value = true;
			await foldersStore.getFolderPath(routeProjectId, parentFolder);
			breadcrumbsLoading.value = false;
		}

		workflowsAndFolders.value = fetchedResources;

		// Toggle ownership cards visibility only after we have fetched the workflows
		showCardsBadge.value = isOverviewPage.value || filters.value.search !== '';

		return fetchedResources;
	} catch (error) {
		toast.showError(error, i18n.baseText('workflows.list.error.fetching'));
		// redirect to the project page if the folder is not found
		void router.push({ name: VIEWS.PROJECTS_FOLDERS, params: { projectId: routeProjectId } });
		return [];
	} finally {
		delayedLoading.cancel();
		loading.value = false;
		if (breadcrumbsLoading.value) {
			breadcrumbsLoading.value = false;
		}
	}
};

// Filter and sort methods

const onSortUpdated = async (sort: string) => {
	currentSort.value =
		WORKFLOWS_SORT_MAP[sort as keyof typeof WORKFLOWS_SORT_MAP] ?? 'updatedAt:desc';
	if (currentSort.value !== 'updatedAt:desc') {
		void router.replace({ query: { ...route.query, sort } });
	} else {
		void router.replace({ query: { ...route.query, sort: undefined } });
	}
	await fetchWorkflows();
};

const onFiltersUpdated = async () => {
	currentPage.value = 1;
	saveFiltersOnQueryString();
	await callDebounced(fetchWorkflows, { debounceTime: FILTERS_DEBOUNCE_TIME, trailing: true });
};

const onSearchUpdated = async (search: string) => {
	currentPage.value = 1;
	saveFiltersOnQueryString();
	if (search) {
		await callDebounced(fetchWorkflows, { debounceTime: SEARCH_DEBOUNCE_TIME, trailing: true });
	} else {
		// No need to debounce when clearing search
		await fetchWorkflows();
	}
};

const setCurrentPage = async (page: number) => {
	currentPage.value = page;
	await callDebounced(fetchWorkflows, { debounceTime: FILTERS_DEBOUNCE_TIME, trailing: true });
};

const setPageSize = async (size: number) => {
	pageSize.value = size;
	await callDebounced(fetchWorkflows, { debounceTime: FILTERS_DEBOUNCE_TIME, trailing: true });
};

const onClickTag = async (tagId: string) => {
	if (!filters.value.tags.includes(tagId)) {
		filters.value.tags.push(tagId);
		currentPage.value = 1;
		saveFiltersOnQueryString();
		await fetchWorkflows();
	}
};

// Query string methods

const saveFiltersOnQueryString = () => {
	// Get current query parameters
	const currentQuery = { ...route.query };

	// Update filter parameters
	if (filters.value.search) {
		currentQuery.search = filters.value.search;
	} else {
		delete currentQuery.search;
	}

	if (filters.value.status !== StatusFilter.ALL) {
		currentQuery.status = (filters.value.status === StatusFilter.ACTIVE).toString();
	} else {
		delete currentQuery.status;
	}

	if (filters.value.tags.length) {
		currentQuery.tags = filters.value.tags.join(',');
	} else {
		delete currentQuery.tags;
	}

	if (filters.value.homeProject) {
		currentQuery.homeProject = filters.value.homeProject;
	} else {
		delete currentQuery.homeProject;
	}

	void router.replace({
		query: Object.keys(currentQuery).length ? currentQuery : undefined,
	});
};

const setFiltersFromQueryString = async () => {
	const newQuery: LocationQueryRaw = { ...route.query };
	const { tags, status, search, homeProject, sort } = route.query ?? {};

	// Helper to check if string value is not empty
	const isValidString = (value: unknown): value is string =>
		typeof value === 'string' && value.trim().length > 0;

	// Handle home project
	if (isValidString(homeProject)) {
		await projectsStore.getAvailableProjects();
		if (isValidProjectId(homeProject)) {
			newQuery.homeProject = homeProject;
			filters.value.homeProject = homeProject;
		} else {
			delete newQuery.homeProject;
		}
	} else {
		delete newQuery.homeProject;
	}

	// Handle search
	if (isValidString(search)) {
		newQuery.search = search;
		filters.value.search = search;
	} else {
		delete newQuery.search;
	}

	// Handle tags
	if (isValidString(tags)) {
		await tagsStore.fetchAll();
		const validTags = tags
			.split(',')
			.filter((tag) => tagsStore.allTags.map((t) => t.id).includes(tag));

		if (validTags.length) {
			newQuery.tags = validTags.join(',');
			filters.value.tags = validTags;
		} else {
			delete newQuery.tags;
		}
	} else {
		delete newQuery.tags;
	}

	// Handle status
	const validStatusValues = ['true', 'false'];
	if (isValidString(status) && validStatusValues.includes(status)) {
		newQuery.status = status;
		filters.value.status = status === 'true' ? StatusFilter.ACTIVE : StatusFilter.DEACTIVATED;
	} else {
		delete newQuery.status;
	}

	// Handle sort
	if (isValidString(sort)) {
		const newSort = WORKFLOWS_SORT_MAP[sort as keyof typeof WORKFLOWS_SORT_MAP] ?? 'updatedAt:desc';
		newQuery.sort = sort;
		currentSort.value = newSort;
	} else {
		delete newQuery.sort;
	}

	void router.replace({ query: newQuery });
};

// Misc methods

const addWorkflow = () => {
	uiStore.nodeViewInitialized = false;
	void router.push({
		name: VIEWS.NEW_WORKFLOW,
		query: { projectId: route.params?.projectId, parentFolderId: route.params?.folderId },
	});

	telemetry.track('User clicked add workflow button', {
		source: 'Workflows list',
	});
	trackEmptyCardClick('blank');
};

const trackEmptyCardClick = (option: 'blank' | 'templates' | 'courses') => {
	telemetry.track('User clicked empty page option', {
		option,
	});
};

function isValidProjectId(projectId: string) {
	return projectsStore.availableProjects.some((project) => project.id === projectId);
}

const openAIWorkflow = async (source: string) => {
	dismissEasyAICallout();
	telemetry.track(
		'User clicked test AI workflow',
		{
			source,
		},
		{ withPostHog: true },
	);

	const easyAiWorkflowJson = getEasyAiWorkflowJson();

	await router.push({
		name: VIEWS.TEMPLATE_IMPORT,
		params: { id: easyAiWorkflowJson.meta.templateId },
		query: { fromJson: 'true', parentFolderId: route.params.folderId },
	});
};

const dismissEasyAICallout = () => {
	easyAICalloutVisible.value = false;
};

const onWorkflowActiveToggle = (data: { id: string; active: boolean }) => {
	const workflow: WorkflowListItem | undefined = workflowsAndFolders.value.find(
		(w): w is WorkflowListItem => w.id === data.id,
	);
	if (!workflow) return;
	workflow.active = data.active;
};

const getFolderListItem = (folderId: string): FolderListItem | undefined => {
	return workflowsAndFolders.value.find(
		(resource): resource is FolderListItem =>
			resource.resource === 'folder' && resource.id === folderId,
	);
};

const getFolderContent = async (folderId: string) => {
	const folderListItem = getFolderListItem(folderId);
	if (folderListItem) {
		return {
			workflowCount: folderListItem.workflowCount,
			subFolderCount: folderListItem.subFolderCount,
		};
	}
	try {
		// Fetch the folder content from API
		const content = await foldersStore.fetchFolderContent(currentProject.value?.id ?? '', folderId);
		return { workflowCount: content.totalWorkflows, subFolderCount: content.totalSubFolders };
	} catch (error) {
		toast.showMessage({
			title: i18n.baseText('folders.delete.error.message'),
			message: i18n.baseText('folders.not.found.message'),
			type: 'error',
		});
		return { workflowCount: 0, subFolderCount: 0 };
	}
};

// Breadcrumbs methods

/**
 * Breadcrumbs: Calculate visible and hidden items for both main breadcrumbs and card breadcrumbs
 * We do this here and pass to each component to avoid recalculating in each card
 */
const visibleBreadcrumbsItems = computed<FolderPathItem[]>(() => {
	if (!currentFolder.value) return [];
	const items: FolderPathItem[] = [];
	const parent = foldersStore.getCachedFolder(currentFolder.value.parentFolder ?? '');
	if (parent) {
		items.push({
			id: parent.id,
			label: parent.name,
			href: `/projects/${route.params.projectId}/folders/${parent.id}/workflows`,
			parentFolder: parent.parentFolder,
		});
	}
	items.push({
		id: currentFolder.value.id,
		label: currentFolder.value.name,
		parentFolder: parent?.parentFolder,
	});
	return items;
});

const hiddenBreadcrumbsItems = computed<FolderPathItem[]>(() => {
	const lastVisibleParent: FolderPathItem =
		visibleBreadcrumbsItems.value[visibleBreadcrumbsItems.value.length - 1];
	if (!lastVisibleParent) return [];
	const items: FolderPathItem[] = [];
	// Go through all the parent folders and add them to the hidden items
	let parentFolder = lastVisibleParent.parentFolder;
	while (parentFolder) {
		const parent = foldersStore.getCachedFolder(parentFolder);

		if (!parent) break;
		items.unshift({
			id: parent.id,
			label: parent.name,
			href: `/projects/${route.params.projectId}/folders/${parent.id}/workflows`,
			parentFolder: parent.parentFolder,
		});
		parentFolder = parent.parentFolder;
	}
	return items;
});

/**
 * Main breadcrumbs items that show on top of the list
 * These show path to the current folder with up to 2 parents visible
 */
const mainBreadcrumbs = computed(() => {
	return {
		visibleItems: visibleBreadcrumbsItems.value,
		hiddenItems: hiddenBreadcrumbsItems.value,
	};
});

const onBreadcrumbItemClick = (item: PathItem) => {
	if (item.href) {
		loading.value = true;
		void router
			.push(item.href)
			.then(() => {
				currentFolderId.value = item.id;
				loading.value = false;
			})
			.catch((error) => {
				toast.showError(error, i18n.baseText('folders.open.error.title'));
			});
	}
};

// Folder actions

// Main header action handlers
// These render next to the breadcrumbs and are applied to the current folder/project
const onBreadCrumbsAction = async (action: string) => {
	switch (action) {
		case FOLDER_LIST_ITEM_ACTIONS.CREATE:
			if (!route.params.projectId) return;
			const currentParent = currentFolder.value?.name || projectName.value;
			if (!currentParent) return;
			await createFolder({
				id: (route.params.folderId as string) ?? '-1',
				name: currentParent,
				type: currentFolder.value ? 'folder' : 'project',
			});
			break;
		case FOLDER_LIST_ITEM_ACTIONS.CREATE_WORKFLOW:
			addWorkflow();
			break;
		case FOLDER_LIST_ITEM_ACTIONS.DELETE:
			if (!route.params.folderId) return;
			const content = await getFolderContent(route.params.folderId as string);
			await deleteFolder(
				route.params.folderId as string,
				content.workflowCount,
				content.subFolderCount,
			);
			break;
		case FOLDER_LIST_ITEM_ACTIONS.RENAME:
			if (!route.params.folderId) return;
			await renameFolder(route.params.folderId as string);
			break;
		case FOLDER_LIST_ITEM_ACTIONS.MOVE:
			if (!currentFolder.value) return;
			uiStore.openMoveToFolderModal(
				'folder',
				{
					id: currentFolder.value?.id,
					name: currentFolder.value?.name,
					parentFolderId: currentFolder.value?.parentFolder,
				},
				workflowListEventBus,
			);
			break;
		default:
			break;
	}
};

// Folder card action handlers
// These render on each folder card and are applied to the clicked folder
const onFolderCardAction = async (payload: { action: string; folderId: string }) => {
	const clickedFolder = foldersStore.getCachedFolder(payload.folderId);
	if (!clickedFolder) return;
	switch (payload.action) {
		case FOLDER_LIST_ITEM_ACTIONS.CREATE:
			await createFolder(
				{
					id: clickedFolder.id,
					name: clickedFolder.name,
					type: 'folder',
				},
				{ openAfterCreate: true },
			);
			break;
		case FOLDER_LIST_ITEM_ACTIONS.CREATE_WORKFLOW:
			currentFolderId.value = clickedFolder.id;
			void router.push({
				name: VIEWS.NEW_WORKFLOW,
				query: { projectId: route.params?.projectId, parentFolderId: clickedFolder.id },
			});
			break;
		case FOLDER_LIST_ITEM_ACTIONS.DELETE: {
			const content = await getFolderContent(clickedFolder.id);
			await deleteFolder(clickedFolder.id, content.workflowCount, content.subFolderCount);
			break;
		}
		case FOLDER_LIST_ITEM_ACTIONS.RENAME:
			await renameFolder(clickedFolder.id);
			break;
		case FOLDER_LIST_ITEM_ACTIONS.MOVE:
			uiStore.openMoveToFolderModal(
				'folder',
				{
					id: clickedFolder.id,
					name: clickedFolder.name,
					parentFolderId: clickedFolder.parentFolder,
				},
				workflowListEventBus,
			);
			break;
		default:
			break;
	}
};

// Reusable action handlers
// Both action handlers ultimately call these methods once folder to apply action to is determined
const createFolder = async (
	parent: { id: string; name: string; type: 'project' | 'folder' },
	options: { openAfterCreate: boolean } = { openAfterCreate: false },
) => {
	const promptResponsePromise = message.prompt(
		i18n.baseText('folders.add.to.parent.message', { interpolate: { parent: parent.name } }),
		{
			confirmButtonText: i18n.baseText('generic.create'),
			cancelButtonText: i18n.baseText('generic.cancel'),
			inputValidator: folderHelpers.validateFolderName,
			customClass: 'add-folder-modal',
		},
	);
	const promptResponse = await promptResponsePromise;
	if (promptResponse.action === MODAL_CONFIRM) {
		const folderName = promptResponse.value;
		try {
			const newFolder = await foldersStore.createFolder(
				folderName,
				route.params.projectId as string,
				parent.type === 'folder' ? parent.id : undefined,
			);

			const newFolderURL = router.resolve({
				name: VIEWS.PROJECTS_FOLDERS,
				params: { projectId: route.params.projectId, folderId: newFolder.id },
			}).href;
			toast.showToast({
				title: i18n.baseText('folders.add.success.title'),
				message: i18n.baseText('folders.add.success.message', {
					interpolate: {
						link: newFolderURL,
						folderName: newFolder.name,
					},
				}),
				onClick: (event: MouseEvent | undefined) => {
					if (event?.target instanceof HTMLAnchorElement) {
						event.preventDefault();
						void router.push(newFolderURL);
					}
				},
				type: 'success',
			});
			telemetry.track('User created folder', {
				folder_id: newFolder.id,
			});
			if (options.openAfterCreate) {
				// Navigate to parent folder id option specified by the caller
				await router.push({
					name: VIEWS.PROJECTS_FOLDERS,
					params: { projectId: route.params.projectId, folderId: parent.id },
				});
			} else {
				// If we are on an empty list, just add the new folder to the list
				if (!workflowsAndFolders.value.length) {
					workflowsAndFolders.value = [
						{
							id: newFolder.id,
							name: newFolder.name,
							resource: 'folder',
							createdAt: newFolder.createdAt,
							updatedAt: newFolder.updatedAt,
							homeProject: projectsStore.currentProject as ProjectSharingData,
							sharedWithProjects: [],
							workflowCount: 0,
							subFolderCount: 0,
						},
					];
					foldersStore.cacheFolders([
						{ id: newFolder.id, name: newFolder.name, parentFolder: currentFolder.value?.id },
					]);
				} else {
					// Else fetch again with same filters & pagination applied
					await fetchWorkflows();
				}
			}
		} catch (error) {
			toast.showError(error, i18n.baseText('folders.create.error.title'));
		}
	}
};

const renameFolder = async (folderId: string) => {
	const folder = foldersStore.getCachedFolder(folderId);
	if (!folder || !currentProject.value) return;
	const promptResponsePromise = message.prompt(
		i18n.baseText('folders.rename.message', { interpolate: { folderName: folder.name } }),
		{
			confirmButtonText: i18n.baseText('generic.rename'),
			cancelButtonText: i18n.baseText('generic.cancel'),
			inputValue: folder.name,
			customClass: 'rename-folder-modal',
			inputValidator: folderHelpers.validateFolderName,
		},
	);
	const promptResponse = await promptResponsePromise;
	if (promptResponse.action === MODAL_CONFIRM) {
		const newFolderName = promptResponse.value;
		try {
			await foldersStore.renameFolder(currentProject.value?.id, folderId, newFolderName);
			foldersStore.breadcrumbsCache[folderId].name = newFolderName;
			toast.showMessage({
				title: i18n.baseText('folders.rename.success.message', {
					interpolate: { folderName: newFolderName },
				}),
				type: 'success',
			});
			await fetchWorkflows();
			telemetry.track('User renamed folder', {
				folder_id: folderId,
			});
		} catch (error) {
			toast.showError(error, i18n.baseText('folders.rename.error.title'));
		}
	}
};

const createFolderInCurrent = async () => {
	// Show the community plus enrollment modal if the user is self-hosted, and hasn't enabled folders
	if (showRegisteredCommunityCTA.value) {
		uiStore.openModalWithData({
			name: COMMUNITY_PLUS_ENROLLMENT_MODAL,
			data: { customHeading: i18n.baseText('folders.registeredCommunity.cta.heading') },
		});
		return;
	}
	if (!route.params.projectId) return;
	const currentParent = currentFolder.value?.name || projectName.value;
	if (!currentParent) return;
	await createFolder({
		id: (route.params.folderId as string) ?? '-1',
		name: currentParent,
		type: currentFolder.value ? 'folder' : 'project',
	});
};

const deleteFolder = async (folderId: string, workflowCount: number, subFolderCount: number) => {
	if (subFolderCount || workflowCount) {
		uiStore.openDeleteFolderModal(folderId, workflowListEventBus, {
			workflowCount,
			subFolderCount,
		});
	} else {
		await foldersStore.deleteFolder(route.params.projectId as string, folderId);
		toast.showMessage({
			title: i18n.baseText('folders.delete.success.message'),
			type: 'success',
		});
		await onFolderDeleted({ folderId, workflowCount, folderCount: subFolderCount });
	}
};

const moveFolder = async (payload: {
	folder: { id: string; name: string };
	newParent: { id: string; name: string; type: 'folder' | 'project' };
}) => {
	if (!route.params.projectId) return;
	try {
		await foldersStore.moveFolder(
			route.params.projectId as string,
			payload.folder.id,
			payload.newParent.type === 'project' ? '0' : payload.newParent.id,
		);
		const isCurrentFolder = currentFolderId.value === payload.folder.id;
		const newFolderURL = router.resolve({
			name: VIEWS.PROJECTS_FOLDERS,
			params: {
				projectId: route.params.projectId,
				folderId: payload.newParent.type === 'project' ? undefined : payload.newParent.id,
			},
		}).href;
		if (isCurrentFolder) {
			// If we just moved the current folder, automatically navigate to the new folder
			void router.push(newFolderURL);
		} else {
			// Else show success message and update the list
			toast.showToast({
				title: i18n.baseText('folders.move.success.title'),
				message: i18n.baseText('folders.move.success.message', {
					interpolate: { folderName: payload.folder.name, newFolderName: payload.newParent.name },
				}),
				onClick: (event: MouseEvent | undefined) => {
					if (event?.target instanceof HTMLAnchorElement) {
						event.preventDefault();
						void router.push(newFolderURL);
					}
				},
				type: 'success',
			});
			await fetchWorkflows();
		}
	} catch (error) {
		toast.showError(error, i18n.baseText('folders.move.error.title'));
	}
};

const moveWorkflowToFolder = async (payload: {
	id: string;
	name: string;
	parentFolderId?: string;
}) => {
	if (showRegisteredCommunityCTA.value) {
		uiStore.openModalWithData({
			name: COMMUNITY_PLUS_ENROLLMENT_MODAL,
			data: { customHeading: i18n.baseText('folders.registeredCommunity.cta.heading') },
		});
		return;
	}
	uiStore.openMoveToFolderModal(
		'workflow',
		{ id: payload.id, name: payload.name, parentFolderId: payload.parentFolderId },
		workflowListEventBus,
	);
};

const onWorkflowMoved = async (payload: {
	workflow: { id: string; name: string; oldParentId: string };
	newParent: { id: string; name: string; type: 'folder' | 'project' };
}) => {
	if (!route.params.projectId) return;
	try {
		const newFolderURL = router.resolve({
			name: VIEWS.PROJECTS_FOLDERS,
			params: {
				projectId: route.params.projectId,
				folderId: payload.newParent.type === 'project' ? undefined : payload.newParent.id,
			},
		}).href;
		const workflowResource = workflowsAndFolders.value.find(
			(resource): resource is WorkflowListItem => resource.id === payload.workflow.id,
		);
		await workflowsStore.updateWorkflow(payload.workflow.id, {
			parentFolderId: payload.newParent.type === 'project' ? '0' : payload.newParent.id,
			versionId: workflowResource?.versionId,
		});
		await fetchWorkflows();
		toast.showToast({
			title: i18n.baseText('folders.move.workflow.success.title'),
			message: i18n.baseText('folders.move.workflow.success.message', {
				interpolate: { workflowName: payload.workflow.name, newFolderName: payload.newParent.name },
			}),
			onClick: (event: MouseEvent | undefined) => {
				if (event?.target instanceof HTMLAnchorElement) {
					event.preventDefault();
					void router.push(newFolderURL);
				}
			},
			type: 'success',
		});
		telemetry.track('User moved content', {
			workflow_id: payload.workflow.id,
			source_folder_id: payload.workflow.oldParentId,
			destination_folder_id: payload.newParent.id,
		});
	} catch (error) {
		toast.showError(error, i18n.baseText('folders.move.workflow.error.title'));
	}
};

const onCreateWorkflowClick = () => {
	void router.push({
		name: VIEWS.NEW_WORKFLOW,
		query: {
			projectId: currentProject.value?.id,
			parentFolderId: route.params.folderId as string,
		},
	});
};

const formatDate = (date: string) => {
	return new Date(date).toLocaleDateString('ko-KR', {
		year: 'numeric',
		month: 'long',
		day: 'numeric',
	});
};

const onEditWorkflow = (workflow: WorkflowResource) => {
	void router.push({
		name: VIEWS.WORKFLOW,
		params: { name: workflow.id },
	});
};

const onDeleteWorkflow = async (workflow: WorkflowResource) => {
	const confirm = await message.confirm(
		i18n.baseText('workflows.deleteConfirm.message'),
		i18n.baseText('workflows.deleteConfirm.title'),
		{
			confirmButtonText: i18n.baseText('workflows.deleteConfirm.confirmButton'),
			cancelButtonText: i18n.baseText('workflows.deleteConfirm.cancelButton'),
		},
	);

	if (confirm) {
		try {
			await workflowsStore.deleteWorkflow(workflow.id);
			await fetchWorkflows();
			toast.showMessage({
				title: i18n.baseText('workflows.deleted.title'),
				message: i18n.baseText('workflows.deleted.message'),
				type: 'success',
			});
		} catch (error) {
			toast.showError(error, i18n.baseText('workflows.deleted.error.title'));
		}
	}
};

const onTestClick = () => {
	// Add test functionality here
	toast.showMessage({
		title: i18n.baseText('workflows.test.title'),
		message: i18n.baseText('workflows.test.message'),
		type: 'info',
	});
};

// Add Korean translations
i18n.extend({
	workflows: {
		search: {
			placeholder: '워크플로우 검색...',
		},
		addWorkflow: '새 워크플로우',
		test: '테스트',
		edit: '편집',
		delete: '삭제',
		lastUpdated: '마지막 업데이트',
		deleteConfirm: {
			title: '워크플로우 삭제',
			message: '이 워크플로우를 삭제하시겠습니까?',
			confirmButton: '삭제',
			cancelButton: '취소',
		},
		deleted: {
			title: '삭제 완료',
			message: '워크플로우가 삭제되었습니다.',
			error: {
				title: '삭제 실패',
			},
		},
		test: {
			title: '테스트',
			message: '테스트 기능이 준비 중입니다.',
		},
	},
});
</script>

<template>
	<ResourcesListLayout
		v-model:filters="filters"
		resource-key="workflows"
		type="list-paginated"
		:resources="workflowListResources"
		:type-props="{ itemSize: 80 }"
		:shareable="isShareable"
		:initialize="initialize"
		:disabled="readOnlyEnv || !projectPermissions.workflow.create"
		:loading="false"
		:resources-refreshing="loading"
		:custom-page-size="DEFAULT_WORKFLOW_PAGE_SIZE"
		:total-items="workflowsStore.totalWorkflowCount"
		:dont-perform-sorting-and-filtering="true"
		:has-empty-state="foldersStore.totalWorkflowCount === 0 && !currentFolderId"
		class="workflow-list"
		@click:add="addWorkflow"
		@update:search="onSearchUpdated"
		@update:current-page="setCurrentPage"
		@update:page-size="setPageSize"
		@update:filters="onFiltersUpdated"
		@sort="onSortUpdated"
	>
		<template #header>
			<div class="workflow-header">
				<div class="search-container">
					<n8n-input
						:placeholder="i18n.baseText('workflows.search.placeholder')"
						v-model="filters.search"
						size="large"
						clearable
						:prefix-icon="'search'"
					/>
				</div>
				<div class="actions-container">
					<n8n-button
						type="primary"
						size="large"
						@click="addWorkflow"
						:label="i18n.baseText('workflows.addWorkflow')"
						:loading="loading"
						:disabled="readOnlyEnv || !projectPermissions.workflow.create"
					/>
					<n8n-button
						type="tertiary"
						size="large"
						@click="onTestClick"
						:label="i18n.baseText('workflows.test')"
						:loading="loading"
					/>
				</div>
			</div>
		</template>
		<template #item="{ item: data, index }">
			<div class="workflow-card" :key="index">
				<div class="workflow-card-content">
					<div class="workflow-info">
						<n8n-heading tag="h4" size="small">{{ data.name }}</n8n-heading>
						<n8n-text size="small" color="text-light">
							{{ i18n.baseText('workflows.lastUpdated') }}: {{ formatDate(data.updatedAt) }}
						</n8n-text>
					</div>
					<div class="workflow-actions">
						<n8n-button
							type="primary"
							size="small"
							@click="onEditWorkflow(data)"
							:label="i18n.baseText('workflows.edit')"
						/>
						<n8n-button
							type="tertiary"
							size="small"
							@click="onDeleteWorkflow(data)"
							:label="i18n.baseText('workflows.delete')"
						/>
					</div>
				</div>
			</div>
		</template>
	</ResourcesListLayout>
</template>

<style lang="scss" module>
.workflow-list {
	padding: var(--spacing-l);
	background-color: #f8f9fc;
	min-height: 100vh;
}

.workflow-header {
	display: flex;
	justify-content: space-between;
	align-items: center;
	margin-bottom: var(--spacing-l);
	padding: var(--spacing-s) 0;
}

.search-container {
	flex: 1;
	margin-right: var(--spacing-l);
	max-width: 400px;

	:global(input) {
		background-color: white;
		border: 1px solid #e0e3e8;
		border-radius: 8px;
		height: 44px;
		font-size: 14px;

		&::placeholder {
			color: #8c93a3;
		}
	}
}

.actions-container {
	display: flex;
	gap: var(--spacing-s);

	:global(.n8n-button) {
		height: 44px;
		padding: 0 24px;
		font-weight: 500;
		border-radius: 8px;
	}

	:global(.n8n-button--primary) {
		background-color: #4489fe;
		border-color: #4489fe;

		&:hover {
			background-color: #3a75db;
			border-color: #3a75db;
		}
	}

	:global(.n8n-button--tertiary) {
		color: #4489fe;
		border: 1px solid #e0e3e8;
		background-color: white;

		&:hover {
			background-color: #f5f7fa;
			border-color: #d0d4dd;
		}
	}
}

.workflow-card {
	background-color: white;
	border-radius: 12px;
	padding: var(--spacing-l);
	margin-bottom: var(--spacing-s);
	border: 1px solid #e0e3e8;
	transition: all 0.2s ease;

	&:hover {
		transform: translateY(-2px);
		box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
	}
}

.workflow-card-content {
	display: flex;
	justify-content: space-between;
	align-items: center;
}

.workflow-info {
	flex: 1;

	:global(h4) {
		color: #14142b;
		font-size: 16px;
		font-weight: 600;
		margin-bottom: 4px;
	}

	:global(.n8n-text) {
		color: #8c93a3;
		font-size: 13px;
	}
}

.workflow-actions {
	display: flex;
	gap: var(--spacing-s);

	:global(.n8n-button) {
		min-width: 80px;
		height: 36px;
		font-size: 13px;
		border-radius: 6px;
	}

	:global(.n8n-button--primary) {
		background-color: #4489fe;
		border-color: #4489fe;

		&:hover {
			background-color: #3a75db;
			border-color: #3a75db;
		}
	}

	:global(.n8n-button--tertiary) {
		color: #4489fe;
		border: 1px solid #e0e3e8;
		background-color: white;

		&:hover {
			background-color: #f5f7fa;
			border-color: #d0d4dd;
		}
	}
}
</style>

<style lang="scss">
.add-folder-modal {
	width: 500px;
	padding-bottom: 0;
	.el-message-box__message {
		font-size: var(--font-size-xl);
	}
	.el-message-box__btns {
		padding: 0 var(--spacing-l) var(--spacing-l);
	}
	.el-message-box__content {
		padding: var(--spacing-l);
	}
}
</style>
