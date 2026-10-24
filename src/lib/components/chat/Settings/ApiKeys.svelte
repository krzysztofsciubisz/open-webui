<script lang="ts">
	import { canUseApiKeys } from '$lib/utils/settings-access';
	import { toast } from 'svelte-sonner';
	import { onMount, getContext } from 'svelte';

	import { user, config } from '$lib/stores';
	import { createAPIKey, deleteAPIKey, getAPIKey } from '$lib/apis/auths';

	import { copyToClipboard } from '$lib/utils';
	import Dropdown from '$lib/components/common/Dropdown.svelte';
	import DropdownMenu from '$lib/components/common/DropdownMenu.svelte';
	import EllipsisHorizontal from '$lib/components/icons/EllipsisHorizontal.svelte';
	import Plus from '$lib/components/icons/Plus.svelte';
	import Refresh from '$lib/components/icons/Refresh.svelte';
	import XMark from '$lib/components/icons/XMark.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import SensitiveInput from '$lib/components/common/SensitiveInput.svelte';
	import ConfirmDialog from '$lib/components/common/ConfirmDialog.svelte';
	import UserSettingField from './UserSettingField.svelte';
	import UserSettingSection from './UserSettingSection.svelte';

	const i18n = getContext('i18n');

	let JWTTokenCopied = false;

	let APIKey = '';
	let APIKeyCopied = false;
	let showAPIKeyMenu = false;
	let showDeleteAPIKeyConfirm = false;

	const actionButtonClass =
		'text-xs text-gray-500 transition-colors hover:text-gray-900 dark:text-gray-500 dark:hover:text-white';

	const createAPIKeyHandler = async () => {
		APIKey = await createAPIKey(localStorage.token);
		if (APIKey) {
			toast.success($i18n.t('API Key created.'));
		} else {
			toast.error($i18n.t('Failed to create API Key.'));
		}
	};

	const deleteAPIKeyHandler = async () => {
		const res = await deleteAPIKey(localStorage.token).catch((error) => {
			toast.error(`${error}`);
			return false;
		});

		if (res) {
			APIKey = '';
			toast.success($i18n.t('API Key deleted.'));
		}
	};

	onMount(async () => {
		// Only fetch API key if the feature is enabled and user has permission
		if (canUseApiKeys({ user: $user, config: $config })) {
			APIKey = await getAPIKey(localStorage.token).catch((error) => {
				console.log(error);
				return '';
			});
		}
	});
</script>

<div id="tab-api-keys" class="flex h-full flex-col text-sm">
	<div class="flex-1 min-h-0 w-full overflow-y-auto scrollbar-hover pr-1.5">
		<h2 class="mb-4 text-sm font-medium text-gray-900 dark:text-white">
			{$i18n.t('settings.personal.apiKeys.title')}
		</h2>

		{#if canUseApiKeys({ user: $user, config: $config })}
			<UserSettingSection first>
				<UserSettingField
					label={$i18n.t('settings.personal.apiKeys.apiKey.label')}
					description={$i18n.t('settings.personal.apiKeys.apiKey.description')}
				>
					<div class="flex">
						{#if APIKey}
							<SensitiveInput variant="settings" value={APIKey} readOnly={true} />

							<button
								type="button"
								class="ml-1.5 rounded-sm px-1.5 py-1 text-gray-500 transition hover:text-gray-700 dark:hover:text-gray-300"
								aria-label={$i18n.t('Copy API Key')}
								on:click={() => {
									copyToClipboard(APIKey);
									APIKeyCopied = true;
									setTimeout(() => {
										APIKeyCopied = false;
									}, 2000);
								}}
							>
								{#if APIKeyCopied}
									<svg
										xmlns="http://www.w3.org/2000/svg"
										viewBox="0 0 20 20"
										fill="currentColor"
										class="w-4 h-4"
									>
										<path
											fill-rule="evenodd"
											d="M16.704 4.153a.75.75 0 01.143 1.052l-8 10.5a.75.75 0 01-1.127.075l-4.5-4.5a.75.75 0 011.06-1.06l3.894 3.893 7.48-9.817a.75.75 0 011.05-.143z"
											clip-rule="evenodd"
										/>
									</svg>
								{:else}
									<svg
										xmlns="http://www.w3.org/2000/svg"
										viewBox="0 0 16 16"
										fill="currentColor"
										class="w-4 h-4"
									>
										<path
											fill-rule="evenodd"
											d="M11.986 3H12a2 2 0 0 1 2 2v6a2 2 0 0 1-1.5 1.937V7A2.5 2.5 0 0 0 10 4.5H4.063A2 2 0 0 1 6 3h.014A2.25 2.25 0 0 1 8.25 1h1.5a2.25 2.25 0 0 1 2.236 2ZM10.5 4v-.75a.75.75 0 0 0-.75-.75h-1.5a.75.75 0 0 0-.75.75V4h3Z"
											clip-rule="evenodd"
										/>
										<path
											fill-rule="evenodd"
											d="M3 6a1 1 0 0 0-1 1v7a1 1 0 0 0 1 1h7a1 1 0 0 0 1-1V7a1 1 0 0 0-1-1H3Zm1.75 2.5a.75.75 0 0 0 0 1.5h3.5a.75.75 0 0 0 0-1.5h-3.5ZM4 11.75a.75.75 0 0 1 .75-.75h3.5a.75.75 0 0 1 0 1.5h-3.5a.75.75 0 0 1-.75-.75Z"
											clip-rule="evenodd"
										/>
									</svg>
								{/if}
							</button>

							<Dropdown bind:show={showAPIKeyMenu} align="end" sideOffset={4}>
								<Tooltip content={$i18n.t('More')}>
									<button
										type="button"
										class="rounded-sm px-1.5 py-1 text-gray-500 transition hover:text-gray-700 dark:hover:text-gray-300"
										aria-label={$i18n.t('More')}
									>
										<EllipsisHorizontal strokeWidth="2" className="size-4" />
									</button>
								</Tooltip>

								<div slot="content">
									<DropdownMenu className="min-w-[10.625rem]">
										<button
											type="button"
											on:click={() => {
												showAPIKeyMenu = false;
												createAPIKeyHandler();
											}}
										>
											<Refresh strokeWidth="2" className="size-3.5 scale-90" />
											<div class="flex items-center">{$i18n.t('Create new key')}</div>
										</button>

										<hr class="border-gray-50 dark:border-gray-850/30" />

										<button
											type="button"
											on:click={() => {
												showAPIKeyMenu = false;
												showDeleteAPIKeyConfirm = true;
											}}
										>
											<XMark strokeWidth="2" className="size-3.5" />
											<div class="flex items-center">{$i18n.t('Delete')}</div>
										</button>
									</DropdownMenu>
								</div>
							</Dropdown>
						{:else}
							<button
								class="inline-flex items-center gap-1.5 {actionButtonClass}"
								type="button"
								on:click={() => {
									createAPIKeyHandler();
								}}
							>
								<Plus strokeWidth="2" className="size-3.5" />

								{$i18n.t('settings.personal.apiKeys.createNewSecretKey.label')}</button
							>
						{/if}
					</div>
				</UserSettingField>

				{#if $user?.role === 'admin'}
					<UserSettingField
						label={$i18n.t('settings.personal.apiKeys.jwtToken.label')}
						description={$i18n.t('settings.personal.apiKeys.jwtToken.description')}
					>
						<div class="flex">
							<SensitiveInput variant="settings" value={localStorage.token} readOnly={true} />

							<button
								type="button"
								class="ml-1.5 rounded-sm px-1.5 py-1 text-gray-500 transition hover:text-gray-700 dark:hover:text-gray-300"
								aria-label={$i18n.t('Copy Token')}
								on:click={() => {
									copyToClipboard(localStorage.token);
									JWTTokenCopied = true;
									setTimeout(() => {
										JWTTokenCopied = false;
									}, 2000);
								}}
							>
								{#if JWTTokenCopied}
									<svg
										xmlns="http://www.w3.org/2000/svg"
										viewBox="0 0 20 20"
										fill="currentColor"
										class="w-4 h-4"
									>
										<path
											fill-rule="evenodd"
											d="M16.704 4.153a.75.75 0 01.143 1.052l-8 10.5a.75.75 0 01-1.127.075l-4.5-4.5a.75.75 0 011.06-1.06l3.894 3.893 7.48-9.817a.75.75 0 011.05-.143z"
											clip-rule="evenodd"
										/>
									</svg>
								{:else}
									<svg
										xmlns="http://www.w3.org/2000/svg"
										viewBox="0 0 16 16"
										fill="currentColor"
										class="w-4 h-4"
									>
										<path
											fill-rule="evenodd"
											d="M11.986 3H12a2 2 0 0 1 2 2v6a2 2 0 0 1-1.5 1.937V7A2.5 2.5 0 0 0 10 4.5H4.063A2 2 0 0 1 6 3h.014A2.25 2.25 0 0 1 8.25 1h1.5a2.25 2.25 0 0 1 2.236 2ZM10.5 4v-.75a.75.75 0 0 0-.75-.75h-1.5a.75.75 0 0 0-.75.75V4h3Z"
											clip-rule="evenodd"
										/>
										<path
											fill-rule="evenodd"
											d="M3 6a1 1 0 0 0-1 1v7a1 1 0 0 0 1 1h7a1 1 0 0 0 1-1V7a1 1 0 0 0-1-1H3Zm1.75 2.5a.75.75 0 0 0 0 1.5h3.5a.75.75 0 0 0 0-1.5h-3.5ZM4 11.75a.75.75 0 0 1 .75-.75h3.5a.75.75 0 0 1 0 1.5h-3.5a.75.75 0 0 1-.75-.75Z"
											clip-rule="evenodd"
										/>
									</svg>
								{/if}
							</button>
						</div>
					</UserSettingField>
				{/if}
			</UserSettingSection>
		{/if}
	</div>
</div>

<ConfirmDialog
	bind:show={showDeleteAPIKeyConfirm}
	title={$i18n.t('Delete API Key?')}
	confirmLabel={$i18n.t('Delete')}
	on:confirm={deleteAPIKeyHandler}
>
	<div class="text-sm text-gray-500">
		{$i18n.t('This will revoke the current API key.')}
	</div>
</ConfirmDialog>
