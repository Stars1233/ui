<script lang="ts">
  import { writable } from 'svelte/store';

  import { page } from '$app/stores';

  import Checkbox from '$lib/holocene/checkbox.svelte';
  import Icon from '$lib/holocene/icon/icon.svelte';
  import MenuButton from '$lib/holocene/menu/menu-button.svelte';
  import MenuContainer from '$lib/holocene/menu/menu-container.svelte';
  import MenuDivider from '$lib/holocene/menu/menu-divider.svelte';
  import MenuItem from '$lib/holocene/menu/menu-item.svelte';
  import Menu from '$lib/holocene/menu/menu.svelte';
  import { translate } from '$lib/i18n/translate';
  import {
    allEventTypeOptions,
    compactEventTypeOptions,
  } from '$lib/models/event-history/get-event-categorization';
  import { clearActiveGroups } from '$lib/stores/active-events';
  import { eventViewType } from '$lib/stores/event-view';
  import { eventStatusFilter, eventTypeFilter } from '$lib/stores/filters';
  import { temporalVersion } from '$lib/stores/versions';
  import { nexusEnabled } from '$lib/utilities/nexus-enabled';
  import { isVersionNewer } from '$lib/utilities/version-check';

  import { CategoryIcon } from './constants';

  export let compact = false;
  export let minimized = true;

  let open = writable(false);

  $: defaultOptions = compact
    ? compactEventTypeOptions.map((o) => o.value)
    : allEventTypeOptions.map((o) => o.value);

  $: options = [
    ...(compact ? compactEventTypeOptions : allEventTypeOptions).map((o) => ({
      ...o,
      label: translate(o.label),
      icon: CategoryIcon[o.value],
      description: translate(o.description),
    })),
  ];

  $: {
    if (isVersionNewer('1.21.0', $temporalVersion)) {
      options = options.filter(({ value }) => value !== 'update');
    }
  }

  $: {
    if (!nexusEnabled($page.data.systemInfo?.capabilities)) {
      options = options.filter(({ value }) => value !== 'nexus');
    }
  }

  const onOptionClick = ({ value }) => {
    clearActiveGroups();
    $eventTypeFilter = $eventTypeFilter.some((type) => type === value)
      ? $eventTypeFilter.filter((type) => type !== value)
      : [...$eventTypeFilter, value];
  };

  $: filterActive = $eventTypeFilter.length < defaultOptions.length;
</script>

<MenuContainer {open}>
  <MenuButton controls="status-menu">
    <div
      slot="leading"
      class="flex h-6 w-6 flex-col items-center justify-center rounded-full transition-colors duration-200"
      class:bg-interactive={filterActive}
    >
      <Icon name="filter" class={filterActive && 'pt-0.5 text-white'} />
    </div>
    <span class="hidden text-sm md:block">{translate('common.filter')}</span>
  </MenuButton>
  <Menu
    id="event-type-menu"
    keepOpen
    position={minimized ? 'top-right' : 'right'}
    class="w-[220px] md:w-[360px]"
  >
    <MenuItem
      data-testid={translate('common.all')}
      on:click={() => {
        $eventTypeFilter = defaultOptions;
        $eventStatusFilter = false;
      }}
    >
      <Checkbox
        on:change={() => {
          $eventTypeFilter = defaultOptions;
          $eventStatusFilter = false;
        }}
        slot="leading"
        checked={!$eventStatusFilter &&
          $eventTypeFilter.length === defaultOptions.length}
        label={translate('common.all')}
        labelHidden
      />
      {translate('common.all')}
    </MenuItem>
    {#if $eventViewType !== 'json'}
      <MenuItem
        data-testid={translate('common.pending-and-failed')}
        description={translate('common.pending-and-failed-description')}
        on:click={() => {
          $eventTypeFilter = defaultOptions;
          $eventStatusFilter = !$eventStatusFilter;
        }}
        class="items-start"
      >
        <Checkbox
          on:change={() => {
            $eventTypeFilter = defaultOptions;
            $eventStatusFilter = !$eventStatusFilter;
          }}
          slot="leading"
          checked={$eventStatusFilter}
          label={translate('common.all')}
          labelHidden
        />
        {translate('common.pending-and-failed')}
      </MenuItem>
    {/if}
    <MenuItem
      data-testid={translate('common.none')}
      on:click={() => {
        $eventTypeFilter = [];
        $eventStatusFilter = false;
      }}
    >
      <Checkbox
        on:change={() => {
          $eventTypeFilter = [];
          $eventStatusFilter = false;
        }}
        slot="leading"
        checked={!$eventStatusFilter && !$eventTypeFilter.length}
        label={translate('common.none')}
        labelHidden
      />
      {translate('common.none')}
    </MenuItem>
    <MenuDivider />
    {#each options as option}
      <MenuItem
        data-testid={option.label}
        description={option.description}
        on:click={() => {
          onOptionClick(option);
        }}
        class="items-start"
      >
        <Checkbox
          on:click={() => onOptionClick(option)}
          slot="leading"
          checked={$eventTypeFilter.some((type) => type === option.value)}
          label={option.label}
          labelHidden
        />
        {option.label}
      </MenuItem>
    {/each}
  </Menu>
</MenuContainer>
