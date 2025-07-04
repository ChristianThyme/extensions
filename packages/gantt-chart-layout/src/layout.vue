<script lang="ts">
export default {
	inheritAttrs: false,
};
</script>

<script lang="ts" setup>
import { useApi, useStores, useSync } from '@directus/extensions-sdk';
import { onMounted, ref, toRefs, watch } from 'vue';
import { useRouter } from 'vue-router';
import Gantt from '../vendor/frappe-gantt.mjs';

const props = defineProps<{
	collection: {
		type: string;
		required: true;
	};
	fieldsInCollection: any;
	labelField?: string;
	startDateField?: string;
	endDateField?: string;
	dependenciesField?: string;
	viewMode?: string;
	primaryKeyField: any;
	items: any;
	loading: boolean;
	error: any;
	search: {
		type: string;
		default: null;
	};
	selection: any;
	selectMode?: boolean;
	readonly?: boolean;
}>();

const emit = defineEmits(['update:selection', 'update:viewMode']);

let gantt;

const api = useApi();
const router = useRouter();
const viewModes = ref(['Hour', 'Quarter Day', 'Half Day', 'Day', 'Week', 'Month', 'Year']);

const { collection, items, loading, fieldsInCollection, primaryKeyField, labelField, startDateField, endDateField, dependenciesField, selectMode, readonly	} = toRefs(props);

const viewModeWritable = useSync(props, 'viewMode', emit);
const selection = useSync(props, 'selection', emit);
const target = ref();
const { useNotificationsStore } = useStores();
const notificationsStore = useNotificationsStore();
const startDateType = fieldsInCollection.value.find((field) => field.field == startDateField.value)?.type;
const endDateType = fieldsInCollection.value.find((field) => field.field == endDateField.value)?.type;

function getUnreactiveTasks() {
	if (!labelField.value || !startDateField.value || !endDateField.value)
		return [];

	const newItems: any = [];
	const missingDateItems: any = [];
	const invalidDateIntervalItems: any = [];

	props.items.forEach((item) => {
		if (!item[startDateField.value as string] || !item[endDateField.value as string]) {
			missingDateItems.push(item);
			return;
		}

		const start = new Date(item[startDateField.value as string]);
		const end = new Date(item[endDateField.value as string]);

		if (start > end) {
			invalidDateIntervalItems.push(item);
			return;
		}

		newItems.push({
			id: `item-${String(item[primaryKeyField.value.field])}`,
			originalID: item[primaryKeyField.value.field],
			name: item[labelField.value as string],
			start,
			end,
			progress: 0,
			readonly: readonly.value,
			dependencies: (() => {
				if (dependenciesField.value && item[dependenciesField.value as string]) {
					return [`item-${String(item[dependenciesField.value as string])}`];
				}

				return [];
			})(),
		});
	});

	if (missingDateItems.length > 0) {
		// We create a hash of any errors to ensure that we don't repeat the same error message
		var errorHash = missingDateItems.map((item) => item[primaryKeyField.value.field]).sort((a, b) => a < b).join(',').split('').reduce((a, b) => { a = ((a << 5) - a) + b.charCodeAt(0); return a & a; }, 0);

		if (!sessionStorage.getItem(`gantt-md-${errorHash}`)) {
			sessionStorage.setItem(`gantt-md-${errorHash}`, 'true');

			notificationsStore.add({
				title: 'Some items could not be displayed - they are missing either a start of end date',
				text: `Affected items: ${missingDateItems.map((item) => item[labelField.value as string]).join(', ')}`,
				type: 'warning',
				persist: true,
				closeable: true,
			});
		}
	}

	if (invalidDateIntervalItems.length > 0) {
		// We create a hash of any errors to ensure that we don't repeat the same error message
		var errorHash = invalidDateIntervalItems.map((item) => item[primaryKeyField.value.field]).sort((a, b) => a < b).join(',').split('').reduce((a, b) => { a = ((a << 5) - a) + b.charCodeAt(0); return a & a; }, 0);

		if (!sessionStorage.getItem(`gantt-idi-${errorHash}`)) {
			sessionStorage.setItem(`gantt-idi-${errorHash}`, 'true');

			notificationsStore.add({
				title: 'Some items could not be displayed - it is impossible for an item to end before it starts',
				text: `Affected items: ${invalidDateIntervalItems.map((item) => item[labelField.value as string]).join(', ')}`,
				type: 'warning',
				persist: true,
				closeable: true,
			});
		}
	}

	newItems.sort((a, b) => a.start - b.start);
	return newItems;
}

function gotoToday() {
	if (gantt) {
		gantt.scroll_today();
	}
}

function changeViewMode(mode) {
	if (gantt) {
		viewModeWritable.value = mode;
		gantt.change_view_mode(mode);
	}
}

// Made our own click handler so we don't conflict with dragging events in the Gantt-library

const delta = 5;
let startX, startY, startTime;

function mouseDown(e) {
	startX = e.pageX;
	startY = e.pageY;
	startTime = (new Date()).valueOf();
}

function mouseUp(e) {
	const diffX = Math.abs(e.pageX - startX);
	const diffY = Math.abs(e.pageY - startY);

	// Ignore drags and long presses
	if (diffX < delta && diffY < delta && (new Date()).valueOf() - startTime < 1000) {
		if (e.target.classList.contains('bar') || e.target.closest('.bar')) {
			const wrapper = e.target.closest('.bar-wrapper');
			const task = gantt.get_task(wrapper.dataset.id);

			if (task) {
				e.preventDefault();

				if (selectMode.value) {
					// Handle selections when used in select modals
					if (selection.value.includes(task.originalID)) {
						wrapper.classList.remove('selected');
						selection.value = selection.value.filter((selected) => selected !== task.originalID);
					}
					else {
						wrapper.classList.add('selected');
						selection.value = [...selection.value, task.originalID];
					}
				}
				else {
					// Route into the edit page
					router.push(`/content/${collection.value}/${encodeURIComponent(task.originalID)}`);
				}
			}
		}
	}
}

watch([() => props.items, labelField, startDateField, endDateField, dependenciesField], () => {
	if (gantt) {
		gantt.refresh(getUnreactiveTasks());
	}
});

onMounted(() => {
	gantt = new Gantt(target.value, getUnreactiveTasks(), {
		bar_height: 44,
		bar_corner_radius: 5,
		arrow_curve: 5,
		padding: 19,
		view_mode: props.viewMode ?? 'Day',
		language: 'en',
		header_height: 65,
		column_width: 30,
		step: 24,
		date_format: 'YYYY-MM-DD',
		popup_trigger: 'click',
		show_expected_progress: false,
		readonly: readonly.value || selectMode.value,
		highlight_weekend: true,
		scroll_to: 'today',
		lines: 'both',
		auto_move_label: false,
		today_button: false,
		view_mode_select: false,
		view_mode_padding: {
			MONTH: ['6m', '6m'],
			YEAR: ['5y', '5y'],
		},
		async on_date_change(task, start: Date, end: Date) {
			const startDate = new Date(start);
			const endDate = new Date(end);

			startDate.setUTCFullYear(start.getUTCFullYear());
			endDate.setUTCFullYear(end.getUTCFullYear());

			if (viewModeWritable.value) {
				if (['Hour', 'Quarter Day', 'Half Day', 'Day', 'Week', 'Month'].includes(viewModeWritable.value)) {
					startDate.setUTCMonth(start.getUTCMonth());
					startDate.setUTCDate(start.getUTCDate());
					endDate.setUTCMonth(end.getUTCMonth());
					endDate.setUTCDate(end.getUTCDate());
				}

				if (['Hour', 'Quarter Day', 'Half Day'].includes(viewModeWritable.value)) {
					startDate.setUTCHours(start.getUTCHours());
					startDate.setUTCMinutes(start.getUTCMinutes());
					startDate.setUTCSeconds(start.getUTCSeconds());
					endDate.setUTCHours(end.getUTCHours());
					endDate.setUTCMinutes(end.getUTCMinutes());
					endDate.setUTCSeconds(end.getUTCSeconds());
				}
			}

			// Fix for fields using the data type 'dateTime', as they don't contain
			// timezone information and JavaScript prefers UTC, there will be an offset
			// if we don't send it in local time. Using toLocaleString('sv') is a quick
			// way to get a proper formatting without using external libraries, so it is
			// not because of the origin of the developer writing this code.
			const startDateString = startDateType == 'dateTime' ? startDate.toLocaleString('sv').replace(' ', 'T') : startDate.toISOString();
			const endDateString = endDateType == 'dateTime' ? endDate.toLocaleString('sv').replace(' ', 'T') : endDate.toISOString();

			const itemChanges = {
				[startDateField.value as string]: startDateString,
				[endDateField.value as string]: endDateString,
			};

			const edit = await api.patch(`/items/${collection.value}/${task.originalID}`, itemChanges);

			if (edit.status !== 200) {
				alert('An error occurred when saving the changes');
			}
		},
	});
});
</script>

<template>
	<div class="gantt-layout">
		<div class="gantt-toolbar">
			<div class="gantt-toolbar-group">
				<button class="gantt-toolbar-item" @click="gotoToday">
					Today
				</button>
			</div>
			<div class="gantt-toolbar-group">
				<button v-for="mode in viewModes" class="gantt-toolbar-item" :class="{ active: mode == viewModeWritable }" @click="changeViewMode(mode)">
					{{ mode }}
				</button>
			</div>
		</div>
		<div ref="target" class="gantt-target" @mousedown="mouseDown" @mouseup="mouseUp" />
	</div>
</template>

<style lang="scss">
	@import url('../vendor/frappe-gantt.css');

.gantt-layout {
	height: calc(100% - (var(--header-bar-height) + 48px));
	padding: var(--content-padding);
	padding-top: 0;
	position: relative;

	display: flex;
	flex-direction: column;

	.gantt-target {
		flex: 1;
		border: 1px solid var(--theme--form--field--input--border-color);
	}

	.gantt-toolbar {
		position: sticky;
		display: inline-flex;
		top: var(--layout-offset-top);
		z-index: 4;
		width: 100%;
		height: 52px;
		margin-bottom: 36px;
		padding: 0 8px;
		font-weight: inherit !important;
		font-size: inherit !important;
		background-color: var(--theme--background);
		border-top: 2px solid var(--theme--border-color-subdued);
		border-bottom: 2px solid var(--theme--border-color-subdued);
		box-shadow: 0 0 0 2px var(--theme--background);
		align-items: center;
		display: flex;
		justify-content: space-between;
	}

	.gantt-toolbar-group {
		display: inline-flex;
		position: relative;
		vertical-align: middle;
	}

	.gantt-toolbar-item {
		border: 1px solid transparent;
		border-radius: 0.25em;
		display: inline-block;
		font-size: 1em;
		font-weight: 400;
		line-height: 1.5;
		padding: 0.4em 0.65em;
		text-align: center;
		-webkit-appearance: button;
		-webkit-user-select: none;
		-moz-user-select: none;
		user-select: none;
		vertical-align: middle;
		background-color: transparent;
		border-color: transparent;
		color: var(--theme--foreground-subdued);

		&:hover,
		&:focus {
			color: var(--theme--foreground);
		}

		&.active {
			color: var(--theme--primary);
		}
	}
	.gantt-container {
		height: 100%;
	}
	.date-highlight {
		background: var(--theme--primary-background);
		font-size: inherit !important;
		height: 24px !important;
		border-radius: 999px;
	}
	.grid-header {
		position: absolute;
		border-bottom: 1px solid var(--theme--form--field--input--border-color);
	}
	.gantt-container .grid-header {
		background: var(--theme--background);
	}
	.gantt-container .lower-text,
	.gantt-container .upper-text {
		color: var(--theme-foreground);
	}
	.gantt-container .current-date-highlight,
	.current-highlight {
		color: var(--foreground-inverted);
		background: var(--theme--secondary);
	}

	.gantt {
		.tick,
		.row-line {
			stroke: var(--theme--form--field--input--border-color);
		}
		.arrow {
			stroke: var(--theme--gantt-layout--arrow, var(--theme--foreground-subdued));
		}
		.grid-row {
			fill: var(--theme--gantt-layout--grid-row, var(--theme--background));
		}
		.bar-label {
			font-size: var(--v-button-font-size, 16px);
			line-height: var(--v-button-line-height, 22px);
		}
		.holiday-highlight {
			fill: var(--theme--form--field--input--background-subdued);
		}
		.bar {
			fill: var(--theme--gantt-layout--bar, var(--v-button-background-color, var(--theme--primary)));
		}
		.bar-label {
			fill: var(--theme--gantt-layout--bar-foreground, var(--v-button-color, var(--foreground-inverted)));
		}
		.bar-label.big {
			fill: var(--theme-foreground);
		}
		.bar-wrapper:hover .bar {
			fill: var(
				--theme--gantt-layout--bar-active,
				var(--v-button-background-color-hover, var(--theme--primary-accent))
			);
		}
		.bar-wrapper.active .bar {
			fill: var(
				--theme--gantt-layout--bar-active,
				var(--v-button-background-color-hover, var(--theme--primary-accent))
			);
		}
		.bar-wrapper.selected .bar {
			fill: var(--theme--gantt-layout--bar-selected, var(--theme--primary-subdued));
		}
		.handle {
			fill: var(--theme--gantt-layout--handle, var(--theme--form--field--input--border-color));
		}
		.bar-wrapper {
			pointer-events: all;
		}
		.bar-wrapper:hover .handle {
			visibility: visible;
			opacity: 1;
		}

		/* Hide progress as we don't support it at this time */
		.handle.progress,
		.bar-progress {
			display: none;
		}
	}

	/* Hide popup as it contains progress and we don't support it at this time */
	.popup-wrapper {
		display: none;
	}
}
</style>
