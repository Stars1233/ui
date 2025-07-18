<script lang="ts">
  import { goto } from '$app/navigation';
  import { page } from '$app/stores';

  import CancelConfirmationModal from '$lib/components/workflow/client-actions/cancel-confirmation-modal.svelte';
  import ResetConfirmationModal from '$lib/components/workflow/client-actions/reset-confirmation-modal.svelte';
  import SignalConfirmationModal from '$lib/components/workflow/client-actions/signal-confirmation-modal.svelte';
  import TerminateConfirmationModal from '$lib/components/workflow/client-actions/terminate-confirmation-modal.svelte';
  import UpdateConfirmationModal from '$lib/components/workflow/client-actions/update-confirmation-modal.svelte';
  import { MenuDivider, MenuItem } from '$lib/holocene/menu';
  import SplitButton from '$lib/holocene/split-button.svelte';
  import { translate } from '$lib/i18n/translate';
  import { isCloud } from '$lib/stores/advanced-visibility';
  import { coreUserStore } from '$lib/stores/core-user';
  import { resetEvents } from '$lib/stores/events';
  import { temporalVersion } from '$lib/stores/versions';
  import { refresh } from '$lib/stores/workflow-run';
  import type { WorkflowExecution } from '$lib/types/workflows';
  import { routeForWorkflowStart } from '$lib/utilities/route-for';
  import { minimumVersionRequired } from '$lib/utilities/version-check';
  import { workflowCancelEnabled } from '$lib/utilities/workflow-cancel-enabled';
  import { workflowCreateDisabled } from '$lib/utilities/workflow-create-disabled';
  import { workflowResetEnabled } from '$lib/utilities/workflow-reset-enabled';
  import { workflowSignalEnabled } from '$lib/utilities/workflow-signal-enabled';
  import { workflowTerminateEnabled } from '$lib/utilities/workflow-terminate-enabled';
  import { workflowUpdateEnabled } from '$lib/utilities/workflow-update-enabled';

  export let workflow: WorkflowExecution;
  export let namespace: string;
  export let cancelInProgress: boolean;
  export let isRunning: boolean;
  export let first: string | undefined = undefined;
  export let next: string | undefined = undefined;

  let cancelConfirmationModalOpen = false;
  let terminateConfirmationModalOpen = false;
  let resetConfirmationModalOpen = false;
  let signalConfirmationModalOpen = false;
  let updateConfirmationModalOpen = false;

  let resetDescription: string;
  let coreUser = coreUserStore();

  $: cancelEnabled = workflowCancelEnabled(
    $page.data.settings,
    $coreUser,
    namespace,
  );

  $: signalEnabled = workflowSignalEnabled(
    $page.data.settings,
    $coreUser,
    namespace,
  );

  $: updateEnabled = workflowUpdateEnabled(
    $page.data.settings,
    $coreUser,
    namespace,
  );

  $: terminateEnabled = workflowTerminateEnabled(
    $page.data.settings,
    $coreUser,
    namespace,
  );

  $: resetAuthorized = workflowResetEnabled(
    $page.data.settings,
    $coreUser,
    namespace,
  );

  // https://github.com/temporalio/temporal/releases/tag/v1.27.1
  $: canResetWithPendingChildWorkflows =
    minimumVersionRequired('1.27.1', $temporalVersion) ||
    $isCloud ||
    workflow.pendingChildren.length === 0;

  $: resetEnabled =
    resetAuthorized &&
    canResetWithPendingChildWorkflows &&
    $resetEvents.length > 0;
  $: actionsDisabled = !resetEnabled && !signalEnabled && !terminateEnabled;

  let workflowActions: {
    label: string;
    onClick: () => void;
    enabled: boolean;
    testId: string;
    destructive?: boolean;
    description?: string;
  }[];

  $: {
    if (!resetAuthorized) {
      resetDescription = translate('workflows.reset-disabled-unauthorized');
    } else if (resetAuthorized && !canResetWithPendingChildWorkflows) {
      resetDescription = translate('workflows.reset-disabled-pending-children');
    } else if (
      resetAuthorized &&
      canResetWithPendingChildWorkflows &&
      $resetEvents.length === 0
    ) {
      resetDescription = translate('workflows.reset-disabled-no-events');
    }
  }

  $: workflowActions = [
    {
      label: translate('workflows.reset'),
      onClick: () => (resetConfirmationModalOpen = true),
      testId: 'reset-button',
      enabled: resetEnabled,
      description: resetEnabled ? '' : resetDescription,
    },
    {
      label: translate('workflows.signal'),
      onClick: () => (signalConfirmationModalOpen = true),
      testId: 'signal-button',
      enabled: signalEnabled,
      description: signalEnabled ? '' : translate('workflows.signal-disabled'),
    },
    {
      label: translate('workflows.update'),
      onClick: () => (updateConfirmationModalOpen = true),
      testId: 'update-button',
      enabled: updateEnabled,
      description: updateEnabled ? '' : translate('workflows.update-disabled'),
    },
    {
      label: translate('workflows.terminate'),
      onClick: () => (terminateConfirmationModalOpen = true),
      testId: 'terminate-button',
      enabled: terminateEnabled,
      destructive: true,
      description: terminateEnabled
        ? ''
        : translate('workflows.terminate-disabled'),
    },
  ];
</script>

{#if isRunning}
  <SplitButton
    id="workflow-actions"
    menuClass="w-[16rem] md:w-[24rem]"
    position="right"
    disabled={actionsDisabled}
    primaryActionDisabled={!cancelEnabled || cancelInProgress}
    on:click={() => (cancelConfirmationModalOpen = true)}
    label={translate('workflows.request-cancellation')}
    menuLabel={translate('workflows.workflow-actions')}
  >
    {#each workflowActions as { onClick, destructive, label, enabled, testId, description }}
      {#if destructive}
        <MenuDivider />
      {/if}
      <MenuItem
        on:click={onClick}
        {destructive}
        disabled={!enabled}
        data-testid={testId}
        {description}
      >
        {label}
      </MenuItem>
    {/each}
    {#if !workflowCreateDisabled($page)}
      <MenuDivider />
      <MenuItem
        on:click={() =>
          goto(
            routeForWorkflowStart({
              namespace,
              workflowId: workflow.id,
              taskQueue: workflow.taskQueue,
              workflowType: workflow.name,
            }),
          )}
        data-testid="start-workflow-button"
      >
        {translate('workflows.start-workflow-like-this-one')}
      </MenuItem>
    {/if}
  </SplitButton>
{:else}
  <SplitButton
    id="workflow-actions"
    menuClass="w-[16rem]"
    position="right"
    primaryActionDisabled={!resetEnabled}
    on:click={() => (resetConfirmationModalOpen = true)}
    label={translate('workflows.reset')}
    menuLabel={translate('workflows.workflow-actions')}
  >
    <MenuItem
      on:click={() =>
        goto(
          routeForWorkflowStart({
            namespace,
            workflowId: workflow.id,
            taskQueue: workflow.taskQueue,
            workflowType: workflow.name,
          }),
        )}
      disabled={workflowCreateDisabled($page)}
      data-testid="start-workflow-button"
    >
      {translate('workflows.start-workflow-like-this-one')}
    </MenuItem>
    {#if terminateEnabled && next}
      <MenuDivider />
      <MenuItem
        on:click={() => (terminateConfirmationModalOpen = true)}
        data-testid="terminate-button"
        destructive
      >
        {translate('workflows.terminate-latest')}
      </MenuItem>
    {/if}
  </SplitButton>
{/if}

{#if resetEnabled}
  <ResetConfirmationModal
    {refresh}
    {workflow}
    {namespace}
    bind:open={resetConfirmationModalOpen}
  />
{/if}

{#if signalEnabled}
  <SignalConfirmationModal
    {refresh}
    {workflow}
    {namespace}
    bind:open={signalConfirmationModalOpen}
  />
{/if}

{#if updateEnabled}
  <UpdateConfirmationModal
    {workflow}
    {namespace}
    bind:open={updateConfirmationModalOpen}
  />
{/if}

{#if cancelEnabled}
  <CancelConfirmationModal
    {refresh}
    {workflow}
    {namespace}
    bind:open={cancelConfirmationModalOpen}
  />
{/if}

{#if terminateEnabled}
  <TerminateConfirmationModal
    {refresh}
    {workflow}
    {namespace}
    {first}
    bind:open={terminateConfirmationModalOpen}
  />
{/if}
