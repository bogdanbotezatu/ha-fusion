<script lang="ts">
	import ComputeIcon from '$lib/Components/ComputeIcon.svelte';
	import CircularBrightnessIndicator from '$lib/Components/CircularBrightnessIndicator.svelte';
	import CircularTemperatureIndicator from '$lib/Components/CircularTemperatureIndicator.svelte';
	import StateLogic from '$lib/Components/StateLogic.svelte';
	import {
		connection,
		editMode,
		itemHeight,
		lang,
		motion,
		onStates,
		climateHvacActionToMode,
		ripple,
		states,
		templates,
		config,
		selectedLanguage,
		calendarView,
		calendarFirstDay
	} from '$lib/Stores';
	import { getDomain, getName, getTogglableService } from '$lib/Utils';
	import Icon, { loadIcon } from '@iconify/svelte';
	import { callService, type HassEntity } from 'home-assistant-js-websocket';
	import { marked } from 'marked';
	import { onDestroy } from 'svelte';
	import { openModal } from 'svelte-modals';
	import Ripple from 'svelte-ripple';
	import parser from 'js-yaml';

	export let demo: string | undefined = undefined;
	export let sel: any;
	export let sectionName: string | undefined = undefined;

	$: entity_id = demo || sel?.entity_id;
	$: template = $templates?.[sel?.id];
	$: icon = (sel?.template?.icon && template?.icon?.output) || sel?.icon;
	$: color = (sel?.template?.color && template?.color?.output) || sel?.color;
	$: marquee = sel?.marquee;
	$: more_info = sel?.more_info;

	let entity: HassEntity;
	let contentWidth: number;
	let container: HTMLDivElement;
	let loading: boolean;
	let resetLoading: ReturnType<typeof setTimeout> | null;
	let stateOn: boolean;

	/** display loader if no state change has occurred within `$motion`ms */
	let delayLoading: ReturnType<typeof setTimeout> | null;

	// Long press functionality
	let pressTimer: ReturnType<typeof setTimeout> | null = null;
	let isLongPress = false;
	const LONG_PRESS_DURATION = 500; // milliseconds

	/**
	 * Observes changes in the `last_updated` property of an entity.
	 * When the `last_updated` property changes:
	 *
	 * - Updates `entity` with the new state from `$states`
	 * - Resets the `loading` state to `false`
	 * - Clears any pending loading or reset timeouts
	 */
	$: if (entity_id && $states?.[entity_id]?.last_updated !== entity?.last_updated) {
		entity = $states?.[entity_id];

		loading = false;

		if (delayLoading) {
			clearTimeout(delayLoading);
			delayLoading = null;
		}

		if (resetLoading) {
			clearTimeout(resetLoading);
			resetLoading = null;
		}
	}

	$: attributes = entity?.attributes;

	$: iconColor = color
		? color
		: attributes?.hs_color
			? `hsl(${attributes?.hs_color}%, 50%)`
			: 'rgb(75, 166, 237)';

	// icon is image if extension, e.g. test.png, OR if person entity with entity_picture
	$: image = icon?.includes('.') || (getDomain(entity_id) === 'person' && attributes?.entity_picture);

	$: if (sel?.template?.set_state && template?.set_state?.output) {
		// template
		stateOn = $onStates?.includes(template?.set_state?.output?.toLocaleLowerCase());
	} else if (attributes?.hvac_action) {
		// climate
		stateOn = $onStates?.includes(
			$climateHvacActionToMode?.[attributes?.hvac_action]?.toLocaleLowerCase()
		);
	} else if (attributes?.in_progress) {
		// update
		stateOn = typeof attributes.in_progress === 'number';
	} else {
		// default
		stateOn = $onStates?.includes(entity?.state?.toLocaleLowerCase());
	}

	/**
	 * Toggles the state of the specified entity
	 * using the correct service call...
	 */
	function toggle() {
		// if service template
		if (sel?.template?.service && template?.service?.output) {
			try {
				// template is string, try to parse it
				const _template = parser.load(template?.service?.output) as {
					service: string;
					data: Record<string, string | number | boolean>;
				};

				if (_template?.service) {
					const [domain, service] = _template.service.split('.');
					callService($connection, domain, service, _template?.data);
				}
			} catch (error) {
				console.error('Template service YAML parse error:', error);
			}

			return;
		}

		// default
		const service = getTogglableService(entity);

		if (service) {
			// use returned domain to handle specific cases such
			// as 'remote', which uses 'homeassistant.toggle'
			const [_domain, _service] = service.split('.');
			callService($connection, _domain, _service, {
				entity_id
			});

			// loader
			delayLoading = setTimeout(() => {
				loading = true;
			}, $motion);

			// loader 20s fallback
			resetLoading = setTimeout(() => {
				loading = false;
			}, 20_000);
		} else {
			// not in getTogglableService just open modal
			handleClickEvent();
		}
	}

	/**
	 * Handle pointer down - start long press timer
	 */
	function handlePointerDown(event: PointerEvent) {
		if ($editMode) return;
		
		isLongPress = false;
		pressTimer = setTimeout(() => {
			isLongPress = true;
			// Provide haptic feedback if available
			if (navigator.vibrate) {
				navigator.vibrate(50);
			}
			// Open modal on long press
			handleClickEvent();
		}, LONG_PRESS_DURATION);
	}

	/**
	 * Handle pointer up - execute short press action or cancel long press
	 */
	function handlePointerUp(event: PointerEvent) {
		if (pressTimer) {
			clearTimeout(pressTimer);
			pressTimer = null;
		}

		// If it wasn't a long press, execute toggle
		if (!isLongPress && !$editMode) {
			toggle();
		}

		isLongPress = false;
	}

	/**
	 * Handle pointer leave - cancel long press if pointer leaves button
	 */
	function handlePointerLeave() {
		if (pressTimer) {
			clearTimeout(pressTimer);
			pressTimer = null;
		}
		isLongPress = false;
	}

	/**
	 * Delegate to handleEvent
	 */
	function handlePointer() {
		handleEvent({ type: 'preload' });
	}

	/**
	 * handleEvent
	 * pointerenter | pointerdown | click
	 */
	async function handleEvent(event: any) {
		if (event.type === 'click') {
			// In edit mode, allow click to open config
			if ($editMode) {
				await handleClickEvent();
			}
			// In normal mode, click is handled by pointer events for long press
		} else {
			await handlePointerEvent();
		}
	}

	/**
	 * Handle click events
	 * Opens modal for specified domain
	 */
	async function handleClickEvent() {
		if ($editMode) {
			openModal(() => import('$lib/Modal/ButtonConfig.svelte'), {
				demo: entity_id,
				sel,
				sectionName
			});
		} else if (more_info === false) {
			toggle();
		} else {
			switch (getDomain(sel?.entity_id)) {
				// light
				case 'light':
					openModal(() => import('$lib/Modal/LightModal.svelte'), {
						sel: sel
					});
					break;

				// switch
				case 'input_boolean':
				case 'remote':
				case 'siren':
				case 'switch':
					openModal(() => import('$lib/Modal/SwitchModal.svelte'), { sel });
					break;

				// script
				case 'script':
					openModal(() => import('$lib/Modal/ScriptModal.svelte'), { sel });
					break;

				// automation
				case 'automation':
					openModal(() => import('$lib/Modal/AutomationModal.svelte'), { sel });
					break;

				// calendar
				case 'calendar': {
					// set first day of week
					$calendarFirstDay =
						'getWeekInfo' in Intl.Locale.prototype
							? (new Intl.Locale($selectedLanguage) as any)?.getWeekInfo().firstDay
							: (await import('weekstart')).getWeekStartByLocale($selectedLanguage);

					// set calendar view type
					$calendarView = localStorage.getItem('calendar');

					openModal(() => import('$lib/Modal/CalendarModal.svelte'), { sel });
					break;
				}

				// sensor
				case 'air_quality':
				case 'date':
				case 'time':
				case 'event':
				case 'image_processing':
				case 'mailbox':
				case 'sensor':
				case 'binary_sensor':
				case 'stt':
				case 'weather':
				case 'button':
				case 'scene':
				case 'schedule':
				case 'sun':
				case 'person':
				case 'zone':
				case 'input_button':
					openModal(() => import('$lib/Modal/SensorModal.svelte'), { sel });
					break;

				// update
				case 'update':
					openModal(() => import('$lib/Modal/UpdateModal.svelte'), { sel });
					break;

				// number
				case 'input_number':
				case 'number':
					openModal(() => import('$lib/Modal/InputNumberModal.svelte'), { sel });
					break;

				// date
				case 'input_datetime':
				case 'datetime':
					openModal(() => import('$lib/Modal/InputDateModal.svelte'), { sel });
					break;

				// select
				case 'input_select':
				case 'select':
					openModal(() => import('$lib/Modal/InputSelectModal.svelte'), { sel });
					break;

				// text
				case 'input_text':
				case 'text':
					openModal(() => import('$lib/Modal/InputTextModal.svelte'), { sel });
					break;

				case 'timer':
					openModal(() => import('$lib/Modal/TimerModal.svelte'), { sel });
					break;

				case 'vacuum':
					openModal(() => import('$lib/Modal/VacuumModal.svelte'), { sel });
					break;

				case 'lawn_mower':
					openModal(() => import('$lib/Modal/LawnMowerModal.svelte'), { sel });
					break;

				case 'valve':
					openModal(() => import('$lib/Modal/ValveModal.svelte'), { sel });
					break;

				case 'image':
					openModal(() => import('$lib/Modal/ImageModal.svelte'), { sel });
					break;

				case 'todo':
					openModal(() => import('$lib/Modal/TodoModal.svelte'), { sel });
					break;

				case 'counter':
					openModal(() => import('$lib/Modal/CounterModal.svelte'), { sel });
					break;

				case 'alarm_control_panel':
					openModal(() => import('$lib/Modal/AlarmControlPanelModal.svelte'), { sel });
					break;

				case 'lock':
					openModal(() => import('$lib/Modal/LockModal.svelte'), { sel });
					break;

				case 'climate':
					openModal(() => import('$lib/Modal/ClimateModal.svelte'), { sel });
					break;

				case 'camera':
					openModal(() => import('$lib/Modal/CameraModal.svelte'), { sel });
					break;

				case 'water_heater':
					openModal(() => import('$lib/Modal/WaterHeaterModal.svelte'), { sel });
					break;

				case 'humidifier':
					openModal(() => import('$lib/Modal/HumidifierModal.svelte'), { sel });
					break;

				case 'media_player':
					openModal(() => import('$lib/Modal/MediaPlayer.svelte'), {
						selected: sel
					});
					break;

				case 'group':
					openModal(() => import('$lib/Modal/GroupModal.svelte'), { sel });
					break;

				case 'device_tracker': {
					if ($states?.[sel?.entity_id]?.attributes?.source_type === 'gps') {
						openModal(() => import('$lib/Modal/DeviceTrackerModal.svelte'), { sel });
					} else {
						openModal(() => import('$lib/Modal/SensorModal.svelte'), { sel });
					}
					break;
				}

				case 'cover':
					openModal(() => import('$lib/Modal/CoverModal.svelte'), {
						selected: sel
					});
					break;

				case 'fan':
					openModal(() => import('$lib/Modal/FanModal.svelte'), {
						selected: sel
					});
					break;

				default:
					openModal(() => import('$lib/Modal/Unknown.svelte'), {
						selected: sel
					});
					break;
			}
		}
	}

	/**
	 * Preloads module before click event
	 */
	async function handlePointerEvent() {
		if ($editMode) {
			await import('$lib/Modal/ButtonConfig.svelte');
		} else {
			switch (getDomain(sel?.entity_id)) {
				case 'light':
					await import('$lib/Modal/LightModal.svelte');
					break;
				case 'switch':
					await import('$lib/Modal/SwitchModal.svelte');
					break;
				case 'climate':
					await import('$lib/Modal/ClimateModal.svelte');
					break;
				case 'media_player':
					await import('$lib/Modal/MediaPlayer.svelte');
					break;
				default:
					await import('$lib/Modal/Unknown.svelte');
					break;
			}
		}
	}

	////// templates //////

	$: if ($config?.state === 'RUNNING' && sel?.template) {
		// for each changed entry in template
		Object.entries(sel?.template as Record<string, string>).forEach(([key, value]) => {
			const compareTemplate = value === template?.[key]?.input;
			const compareEntityId = sel?.entity_id === template?.[key]?.entity_id;
			if (compareTemplate && compareEntityId) return;
			renderTemplate(key, value);
		});
	}

	let unsubscribe: () => void;

	async function renderTemplate(key: string, value: string) {
		if (!$connection || !sel?.id) return;

		try {
			unsubscribe = await $connection.subscribeMessage(
				(response: { result: string } | { error: string; level: 'ERROR' | 'WARNING' }) => {
					let data: any = {
						input: value
					};

					if ('result' in response) {
						data.output =
							key === 'state' || key === 'name'
								? marked.parseInline(String(response.result))
								: String(response.result);
					} else if (response?.level === 'ERROR') {
						console.error(response.error);
						data.error = response.error;
					}

					data.entity_id = sel?.entity_id;

					$templates[sel?.id] = { ...$templates[sel?.id], [key]: data };
				},
				{
					type: 'render_template',
					template: value,
					report_errors: true,
					variables: {
						entity_id: sel?.entity_id
					}
				}
			);
		} catch (error) {
			console.error('Template error:', error);
		}
	}

	onDestroy(() => {
		unsubscribe?.();
		if (pressTimer) {
			clearTimeout(pressTimer);
		}
	});
</script>

<div
	class="container"
	bind:this={container}
	data-state={stateOn}
	data-layout={sel?.layout || 'square'}
	tabindex="-1"
	style={!$editMode ? 'cursor: pointer;' : ''}
	style:min-height="{$itemHeight}px"
	on:pointerenter={handlePointer}
	on:pointerdown={handlePointerDown}
	on:pointerup={handlePointerUp}
	on:pointerleave={handlePointerLeave}
	on:click={handleEvent}
	use:Ripple={{
		...$ripple,
		color: !$editMode
			? stateOn
				? 'rgba(0, 0, 0, 0.25)'
				: 'rgba(255, 255, 255, 0.15)'
			: 'rgba(0, 0, 0, 0)'
	}}
>
	<!-- CIRCULAR INDICATORS -->
	{#if stateOn && attributes?.brightness && getDomain(entity_id) === 'light'}
		<div class="brightness-indicator">
			<CircularBrightnessIndicator 
				{entity_id}
				{attributes}
				value={Math.round((attributes.brightness / 255) * 100)}
				size={44}
				strokeWidth={3}
			/>
		</div>
	{:else if stateOn && attributes?.current_position !== undefined && getDomain(entity_id) === 'cover'}
		<div class="brightness-indicator">
			<CircularBrightnessIndicator 
				{entity_id}
				{attributes}
				value={attributes.current_position}
				size={44}
				strokeWidth={3}
			/>
		</div>
	{:else if stateOn && attributes?.volume_level !== undefined && getDomain(entity_id) === 'media_player'}
		<div class="brightness-indicator">
			<CircularBrightnessIndicator 
				{entity_id}
				{attributes}
				value={Math.round(attributes.volume_level * 100)}
				size={44}
				strokeWidth={3}
			/>
		</div>
	{:else if getDomain(entity_id) === 'climate' && attributes?.current_temperature !== undefined}
		<div class="brightness-indicator">
			<CircularTemperatureIndicator 
				{entity_id}
				{attributes}
				temperature={attributes.current_temperature}
				size={44}
				strokeWidth={3}
			/>
		</div>
	{/if}

	<!-- ICON -->
	<div
		class="icon-container"
		on:keydown
		role="button"
		tabindex="0"
	>
		<div
			class="icon"
			data-state={stateOn}
			style:--icon-color={iconColor}
			style:background-color={sel?.template?.color && template?.color?.output
				? template?.color?.output
				: undefined}
			style:background-image={(getDomain(entity_id) === 'person' && attributes?.entity_picture) || (!icon && attributes?.entity_picture)
				? `url(${attributes?.entity_picture})`
				: image && icon
					? `url(${icon})`
					: 'none'}
			class:image
			class:person-avatar={getDomain(entity_id) === 'person' && attributes?.entity_picture}
		>
			{#if loading}
				<img src="loader.svg" alt="loading" style="margin:0 auto" />
			{:else if image || (getDomain(entity_id) === 'person' && attributes?.entity_picture) || (!icon && attributes?.entity_picture)}
				&nbsp;
			{:else if icon}
				{#await loadIcon(icon)}
					<!-- loading -->
					<Icon icon="ooui:help-ltr" height="none" width="100%" />
				{:then resolvedIcon}
					<!-- exists -->
					<Icon icon={resolvedIcon} height="none" width="100%" />
				{:catch}
					<!-- doesn't exist -->
					<Icon icon="ooui:help-ltr" height="none" width="100%" />
				{/await}
			{:else if entity_id}
				<ComputeIcon {entity_id} skipEntityPicture={getDomain(entity_id) === 'person'} />
			{:else}
				<Icon icon="ooui:help-ltr" height="none" width="100%" />
			{/if}
		</div>
	</div>

	<!-- CONTENT -->
	<div class="content" on:keydown role="button" tabindex="0">
		<!-- NAME -->
		<div class="name" data-state={stateOn}>
			{@html (sel?.template?.name && template?.name?.output) ||
				getName(sel, entity, sectionName) ||
				$lang('unknown')}
		</div>

		<!-- STATE -->
		<div class="state" data-state={stateOn}>
			{#if getDomain(entity_id) === 'climate'}
				<!-- Use StateLogic for climate entities to show hvac_action -->
				<StateLogic {entity_id} selected={sel} />
			{:else if sel?.state || (sel?.template?.state && template?.state?.output)}
				{@html sel?.state || template?.state?.output}
			{:else if sel?.template?.set_state && template?.set_state?.output}
				{@html sel?.template?.set_state && $lang(template?.set_state?.output)}
			{:else}
				<!-- Use StateLogic component to handle all entity states properly -->
				<StateLogic {entity_id} selected={sel} />
			{/if}
		</div>
	</div>
</div>

<style>
	.container {
		/* Enhanced background with fallbacks */
		background: var(--theme-button-background-color-off, var(--color-surface-elevated));
		font-family: inherit;
		width: 100%;
		height: 100%;
		display: grid;
		
		/* Dynamic aspect ratio based on layout */
		aspect-ratio: var(--button-aspect-ratio, 1);
		
		/* Modern border radius and visual effects */
		border-radius: var(--radius-xl);
		border: 1px solid var(--color-border-subtle);
		box-shadow: var(--shadow-md);
		
		margin: 0;
		
		/* Dynamic grid layout based on layout type */
		grid-template-rows: var(--button-grid-rows, auto 1fr);
		grid-template-columns: var(--button-grid-columns, 1fr);
		grid-template-areas: var(--button-grid-areas, 'icon' 'content');
		gap: var(--space-2);
		padding: var(--space-3);

		/* Enhanced ripple effect support */
		transform: translateZ(0);
		overflow: hidden;
		position: relative;
		
		/* Modern glassmorphism effect */
		backdrop-filter: blur(var(--blur-sm));
		-webkit-backdrop-filter: blur(var(--blur-sm));
		
		/* Smooth transitions */
		transition: var(--transition-all);
		
		/* Subtle gradient overlay */
		background-image: var(--gradient-surface);
	}
	
	/* Square layout (default) */
	.container:not([data-layout='rectangular']) {
		--button-aspect-ratio: 1;
		--button-grid-rows: auto 1fr;
		--button-grid-columns: 1fr;
		--button-grid-areas: 'icon' 'content';
	}
	
	/* Rectangular layout */
	.container[data-layout='rectangular'] {
		--button-aspect-ratio: 2 / 1;
		--button-grid-rows: 1fr;
		--button-grid-columns: auto 1fr;
		--button-grid-areas: 'icon content';
	}
	
	/* Enhanced hover states */
	.container:hover {
		box-shadow: var(--shadow-xl);
		transform: translateY(-2px);
		border-color: var(--color-border-emphasis);
	}
	
	/* Modern glass overlay effect */
	.container::before {
		content: '';
		position: absolute;
		inset: 0;
		background: var(--gradient-glass);
		opacity: 0;
		transition: opacity var(--transition-medium);
		pointer-events: none;
		border-radius: inherit;
	}
	
	.container:hover::before {
		opacity: 0.5;
	}

	.image {
		background-size: cover;
		background-repeat: no-repeat;
		background-position: center center;
	}
	
	/* Specific styling for person entity avatars to ensure faces are visible */
	.person-avatar {
		background-size: cover !important;
		background-position: center 30% !important; /* Focus on upper area where face usually is */
		border-radius: 50% !important; /* Make avatars circular */
		padding: 0 !important; /* Remove padding for full avatar display */
	}
	
	/* Preserve circular avatar styling even when active */
	.icon[data-state='true'].person-avatar {
		background: transparent;
		border-color: var(--color-border-subtle);
		border-radius: 50% !important;
		padding: 0 !important;
	}

	/* Brightness indicator in top-right corner */
	.brightness-indicator {
		position: absolute;
		top: var(--space-2);
		right: var(--space-2);
		z-index: 3;
		
		/* Prevent interference with button interactions */
		pointer-events: auto;
	}

	/* Icon container positioned at top-left */
	.icon-container {
		grid-area: icon;
		justify-self: start;
		align-self: start;
		position: relative;
		z-index: 2;
	}
	
	/* Rectangular layout icon adjustments */
	.container[data-layout='rectangular'] .icon-container {
		align-self: center;
	}
	
	/* Ensure icon size is optimized for rectangular layout */
	.container[data-layout='rectangular'] .icon {
		--icon-size: 2.5rem;
	}

	/* Content area with larger text */
	.content {
		grid-area: content;
		display: flex;
		flex-direction: column;
		justify-content: center;
		gap: var(--space-1);
		position: relative;
		z-index: 2;
		min-height: 0; /* Allow content to shrink */
	}
	
	/* Rectangular layout content adjustments */
	.container[data-layout='rectangular'] .content {
		justify-content: center;
		align-items: flex-start;
		padding-left: var(--space-2);
		text-align: left;
	}
	
	/* Improve text spacing for rectangular layout */
	.container[data-layout='rectangular'] .name {
		font-size: 1.1rem;
		font-weight: 500;
		line-height: 1.2;
	}
	
	.container[data-layout='rectangular'] .state {
		font-size: 0.95rem;
		opacity: 0.85;
	}

	.icon {
		--icon-size: 3rem;
		height: var(--icon-size);
		width: var(--icon-size);
		color: var(--icon-color);
		
		/* Enhanced icon background */
		background: var(--color-surface);
		border: 1px solid var(--color-border-subtle);
		box-shadow: var(--shadow-sm);
		
		border-radius: var(--radius-lg);
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 0.5rem;
		background-position: center center;
		background-size: cover;
		background-repeat: no-repeat;
		
		/* Glass effect for icon */
		backdrop-filter: blur(8px);
		-webkit-backdrop-filter: blur(8px);
		
		/* Smooth transitions */
		transition: var(--transition-all);
		position: relative;
		z-index: 1;
		
		/* Flex shrink to prevent icon from being compressed */
		flex-shrink: 0;
	}
	
	/* Enhanced active state for icons */
	.icon[data-state='true'] {
		color: var(--color-white);
		background: var(--icon-color);
		border-color: var(--icon-color);
		box-shadow: var(--shadow-md);
		transform: scale(1.05);
	}
	
	/* Preserve entity_picture for person entities even when active */
	.icon[data-state='true'].image {
		background: transparent;
		border-color: var(--color-border-subtle);
	}

	.name {
		font-weight: var(--font-medium);
		color: var(--theme-button-name-color-off, var(--color-text-primary));
		font-size: 1.1rem;
		line-height: var(--leading-tight);
		transition: var(--transition-colors);
		margin: 0;
		
		/* Better text handling for long names */
		overflow: hidden;
		text-overflow: ellipsis;
		display: -webkit-box;
		-webkit-line-clamp: 2;
		line-clamp: 2;
		-webkit-box-orient: vertical;
		line-height: 1.2;
		max-height: 2.4em; /* 2 lines at 1.2 line-height */
	}

	.state {
		font-weight: var(--font-normal);
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
		color: var(--theme-button-state-color-off, var(--color-text-secondary));
		font-size: 0.75rem;
		line-height: var(--leading-normal);
		transition: var(--transition-colors);
		margin: 0;
		opacity: 0.8;
	}

	.container[data-state='true'] {
		background: var(--theme-button-background-color-on, var(--color-primary));
		border-color: var(--color-primary);
		box-shadow: var(--shadow-lg);
		color: var(--color-white);
	}
	
	/* Enhanced glass effect for active state */
	.container[data-state='true']::before {
		background: linear-gradient(145deg, 
			rgba(255, 255, 255, 0.2) 0%, 
			rgba(255, 255, 255, 0.1) 100%);
		opacity: 1;
	}

	.name[data-state='true'] {
		color: var(--theme-button-name-color-on, var(--color-white));
	}

	.state[data-state='true'] {
		color: var(--theme-button-state-color-on, var(--color-white));
		opacity: 0.9;
	}

	/* Modern focus states for accessibility */
	.container:focus-visible {
		outline: none;
		box-shadow: 
			var(--shadow-lg),
			0 0 0 3px var(--color-primary-hover);
	}
	
	.container[data-state='true']:focus-visible {
		box-shadow: 
			var(--shadow-lg),
			0 0 0 3px var(--color-white);
	}

	/* Phone and Tablet (portrait) */
	@media all and (max-width: 768px) {
		.container {
			/* Maintain consistent aspect ratio on mobile */
			aspect-ratio: 1;
			padding: var(--space-2);
			gap: var(--space-1);
			/* Ensure minimum width for consistency */
			min-width: 8.5rem;
			width: 100%;
			/* Optimize for mobile performance */
			will-change: auto;
		}
		
		/* Force rectangular layouts to be square on mobile for consistency */
		.container[data-layout='rectangular'] {
			aspect-ratio: 1;
			--button-grid-rows: auto 1fr;
			--button-grid-columns: 1fr;
			--button-grid-areas: 'icon' 'content';
		}
		
		/* Reset rectangular content positioning for mobile */
		.container[data-layout='rectangular'] .content {
			justify-content: center;
			align-items: center;
			padding-left: 0;
			text-align: center;
		}
		
		/* Reset rectangular icon positioning for mobile */
		.container[data-layout='rectangular'] .icon-container {
			align-self: start;
			justify-self: center;
		}
		
		.icon {
			--icon-size: 3rem;
			padding: 0.375rem;
		}
		
		.name {
			font-size: 0.8rem;
		}
		
		.state {
			font-size: 0.65rem;
		}
		
		.brightness-indicator {
			top: var(--space-1);
			right: var(--space-1);
		}
		
		.container:hover {
			transform: none; /* Disable transform on mobile for better performance */
		}
		
		.icon[data-state='true'] {
			transform: scale(1.02); /* Reduce scale on mobile */
		}
	}
	
	/* Reduced motion accessibility */
	@media (prefers-reduced-motion: reduce) {
		.container,
		.container::before,
		.icon,
		.name,
		.state {
			transition: none;
		}
		
		.container:hover,
		.icon[data-state='true'] {
			transform: none;
		}
	}
	
	/* Dark theme specific enhancements */
	@media (prefers-color-scheme: dark) {
		.container {
			box-shadow: 
				var(--shadow-md),
				0 0 0 1px var(--color-glass-light);
		}
		
		.container:hover {
			box-shadow: 
				var(--shadow-xl),
				0 0 0 1px var(--color-glass-medium);
		}
		
		.container[data-state='true'] {
			box-shadow: 
				var(--shadow-lg),
				0 0 0 1px var(--color-primary);
		}
	}
</style>
