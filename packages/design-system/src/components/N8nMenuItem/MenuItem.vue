<script lang="ts" setup>
import { ElSubMenu, ElMenuItem } from 'element-plus';
import { computed, useCssModule } from 'vue';
import { useRoute } from 'vue-router';

import { doesMenuItemMatchCurrentRoute } from './routerUtil';
import type { IMenuItem } from '../../types';
import { getInitials } from '../../utils/labelUtil';
import ConditionalRouterLink from '../ConditionalRouterLink';
import N8nIcon from '../N8nIcon';
import N8nTooltip from '../N8nTooltip';

interface MenuItemProps {
	item: IMenuItem;
	compact?: boolean;
	tooltipDelay?: number;
	popperClass?: string;
	mode?: 'router' | 'tabs';
	activeTab?: string;
	handleSelect?: (item: IMenuItem) => void;
}

const props = withDefaults(defineProps<MenuItemProps>(), {
	compact: false,
	tooltipDelay: 300,
	popperClass: '',
	mode: 'router',
});

const $style = useCssModule();
const $route = useRoute();

const availableChildren = computed((): IMenuItem[] =>
	Array.isArray(props.item.children)
		? props.item.children.filter((child) => child.available !== false)
		: [],
);

const currentRoute = computed(() => {
	return $route ?? { name: '', path: '' };
});

const submenuPopperClass = computed((): string => {
	const popperClass = [$style.submenuPopper, props.popperClass];
	if (props.compact) {
		popperClass.push($style.compact);
	}
	return popperClass.join(' ');
});

const isActive = (item: IMenuItem): boolean => {
	if (props.mode === 'router') {
		return doesMenuItemMatchCurrentRoute(item, currentRoute.value);
	} else {
		return item.id === props.activeTab;
	}
};

const isItemActive = (item: IMenuItem): boolean => {
	const hasActiveChild =
		Array.isArray(item.children) && item.children.some((child) => isActive(child));
	return isActive(item) || hasActiveChild;
};
</script>

<template>
	<div :class="['flowstate-menu-item', $style.item]">
		<N8nTooltip
			placement="right"
			:content="compact ? item.label : ''"
			:disabled="!compact"
			:show-after="tooltipDelay"
		>
			<ConditionalRouterLink v-bind="item.route ?? item.link">
				<ElMenuItem
					:id="item.id"
					:class="{
						[$style.menuItem]: true,
						[$style.item]: true,
						[$style.disableActiveStyle]: !isActive(item),
						[$style.active]: isActive(item),
						[$style.compact]: compact,
					}"
					data-test-id="menu-item"
					:index="item.id"
					:disabled="item.disabled"
					@click="handleSelect?.(item)"
				>
					<component
						:is="item.icon"
						v-if="item.icon && typeof item.icon === 'function'"
						:class="$style.menuIcon"
					/>
					<span v-if="!compact" :class="$style.label">{{ item.label }}</span>
					<span v-if="!item.icon && compact" :class="[$style.label, $style.compactLabel]">{{
						getInitials(item.label)
					}}</span>
					<N8nTooltip
						v-if="item.secondaryIcon"
						:placement="item.secondaryIcon?.tooltip?.placement || 'right'"
						:content="item.secondaryIcon?.tooltip?.content"
						:disabled="compact || !item.secondaryIcon?.tooltip?.content"
						:show-after="tooltipDelay"
					>
						<N8nIcon
							:class="$style.secondaryIcon"
							:icon="item.secondaryIcon.name"
							:size="item.secondaryIcon.size || 'small'"
						/>
					</N8nTooltip>
					<N8nSpinner v-if="item.isLoading" :class="$style.loading" size="small" />
				</ElMenuItem>
			</ConditionalRouterLink>
		</N8nTooltip>
	</div>
</template>

<style module lang="scss">
// Element menu-item overrides
:global(.el-menu-item),
:global(.el-sub-menu__title) {
	--menu-font-color: var(--color-text-base);
	--menu-item-active-font-color: var(--color-text-dark);
	--menu-item-height: 35px;
	--sub-menu-item-height: 27px;
}

.submenu {
	background: none !important;

	&.compact :global(.el-sub-menu__title) {
		i {
			display: none;
		}
	}

	:global(.el-sub-menu__title) {
		display: flex;
		align-items: center;
		border-radius: var(--border-radius-base) !important;
		padding: var(--spacing-2xs) var(--spacing-xs) !important;
		user-select: none;

		i {
			padding-top: 2px;
			&:hover {
				color: var(--color-primary);
			}
		}

		&:hover {
			.icon {
				color: var(--color-text-dark);
			}
		}
	}

	.menuItem {
		height: var(--sub-menu-item-height) !important;
		min-width: auto !important;
		margin: var(--spacing-2xs) 0 !important;
		padding-left: var(--spacing-l) !important;
		user-select: none;

		&:hover {
			.icon {
				color: var(--color-text-dark);
			}
		}
	}
}

.menuIcon {
	height: 18px !important;
	margin-right: 5px;
}

.disableActiveStyle {
	background-color: initial !important;
	color: var(--color-text-base) !important;

	svg {
		color: var(--color-text-base) !important;
	}

	&:hover {
		color: var(--color-text-dark) !important;
		svg {
			color: var(--color-text-dark) !important;
		}
		&:global(.el-sub-menu) {
			background-color: unset !important;
		}
	}
}

.active {
	&.menuItem {
		position: relative;
	}
	&.menuItem::before {
		content: '';
		position: absolute;
		top: 0;
		left: 0;
		bottom: 0;
		width: 5px;
		background-color: black;
	}
}

.menuItem {
	height: 52px;
	display: flex;
	padding: var(--spacing-2xs) var(--spacing-xs) !important;
	margin: 0 !important;
	border-radius: var(--border-radius-base) !important;
	overflow: hidden;
	background-color: white !important;
	border: 1px solid #e6e6e6;
	box-shadow: 0px 2px 8px 0px #8c8c8c1a;

	&.compact {
		padding: var(--spacing-2xs) 0 !important;
		justify-content: center;
	}
}

.icon {
	min-width: var(--spacing-s);
	margin-right: var(--spacing-xs);
	text-align: center;
}

.loading {
	margin-left: var(--spacing-xs);
}

.secondaryIcon {
	display: flex;
	align-items: center;
	justify-content: flex-end;
	flex: 1;
	margin-left: 20px;
}

.label {
	overflow: hidden;
	text-overflow: ellipsis;
	user-select: none;
}

.compactLabel {
	text-overflow: unset;
}

.compact {
	.icon {
		margin: 0;
		overflow: visible !important;
		visibility: visible !important;
		width: initial !important;
		height: initial !important;
	}
	.secondaryIcon {
		display: none;
	}
}

.submenuPopper {
	display: block;

	ul {
		padding: 0 var(--spacing-xs) !important;
	}
	.menuItem {
		display: flex;
		padding: var(--spacing-2xs) var(--spacing-xs) !important;
		margin: var(--spacing-2xs) 0 !important;
	}

	.icon {
		margin-right: var(--spacing-xs);
	}

	&.compact {
		.label {
			display: inline-block;
		}
	}
}
</style>
