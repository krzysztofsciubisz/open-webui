<script lang="ts">
	import { toast } from 'svelte-sonner';
	import { onMount, getContext } from 'svelte';

	import { user, config } from '$lib/stores';
	import { updateUserProfile, getSessionUser } from '$lib/apis/auths';
	import { getUserVariables, updateUserVariables } from '$lib/apis/users';

	import UpdatePassword from './Account/UpdatePassword.svelte';
	import Mfa from './Account/Mfa.svelte';
	import { generateInitialsImage } from '$lib/utils';
	import Textarea from '$lib/components/common/Textarea.svelte';
	import Modal from '$lib/components/common/Modal.svelte';
	import UserProfileImage from './Account/UserProfileImage.svelte';
	import UserSettingField from './UserSettingField.svelte';
	import SettingsSelect from '$lib/components/common/SettingsSelect.svelte';
	import UserSettingSection from './UserSettingSection.svelte';

	const i18n = getContext('i18n');

	export let saveHandler: () => void | Promise<void>;

	let profileImageUrl = '';
	let name = '';
	let bio = '';

	let _gender = '';
	let gender = '';
	let dateOfBirth = '';
	let variableRows: { key: string; value: string }[] = [];
	let variableModalOpen = false;
	let variableFormIndex: number | null = null;
	let variableFormKey = '';
	let variableFormValue = '';

	const textareaClass =
		'w-full resize-y rounded-lg border border-gray-100/50 bg-gray-50/40 px-2 py-1.5 text-xs text-gray-700 outline-hidden transition-colors placeholder:text-gray-300 focus:border-blue-400 dark:border-white/[0.04] dark:bg-white/[0.03] dark:text-gray-300 dark:placeholder:text-gray-700 dark:focus:border-blue-500';
	const inputClass =
		'h-7 w-full rounded-lg border border-gray-100/50 bg-gray-50/40 px-2 text-xs text-gray-700 outline-hidden transition-colors placeholder:text-gray-300 focus:border-blue-400 dark:border-white/[0.04] dark:bg-white/[0.03] dark:text-gray-300 dark:placeholder:text-gray-700 dark:focus:border-blue-500';
	const variableValueClass =
		'w-full resize-none rounded-lg border border-gray-100/50 bg-gray-50/40 px-2 py-1.5 text-xs text-gray-700 outline-hidden transition-colors placeholder:text-gray-300 focus:border-blue-400 dark:border-white/[0.04] dark:bg-white/[0.03] dark:text-gray-300 dark:placeholder:text-gray-700 dark:focus:border-blue-500';
	const actionButtonClass =
		'text-xs text-gray-500 transition-colors hover:text-gray-900 dark:text-gray-500 dark:hover:text-white';
	const variableRowClass = 'flex w-full items-center gap-1.5';
	const variableKeyRegex = /^[a-z][a-z0-9_]*$/;

	const setVariableRows = (variables = {}) => {
		variableRows = Object.entries(variables).map(([key, value]) => ({
			key,
			value: String(value ?? '')
		}));
	};

	const openVariableModal = (idx: number | null = null) => {
		const row = idx === null ? null : variableRows[idx];
		variableFormIndex = idx;
		variableFormKey = row?.key ?? '';
		variableFormValue = row?.value ?? '';
		variableModalOpen = true;
	};

	const removeVariable = (idx: number) => {
		variableRows = variableRows.filter((_, rowIdx) => rowIdx !== idx);
	};

	const saveVariableForm = () => {
		const key = variableFormKey.trim();

		if (!variableKeyRegex.test(key)) {
			toast.error($i18n.t('Variable keys must use lowercase snake case.'));
			return;
		}
		if (variableRows.some((row, idx) => idx !== variableFormIndex && row.key === key)) {
			toast.error($i18n.t('Variable keys must be unique.'));
			return;
		}

		const row = { key, value: variableFormValue ?? '' };
		variableRows =
			variableFormIndex === null
				? [...variableRows, row]
				: variableRows.map((current, idx) => (idx === variableFormIndex ? row : current));
		variableModalOpen = false;
	};

	const deleteVariableForm = () => {
		if (variableFormIndex !== null) {
			removeVariable(variableFormIndex);
		}
		variableModalOpen = false;
	};

	const getVariablesPayload = () => {
		const variables: Record<string, string> = {};
		for (const row of variableRows) {
			const key = row.key.trim();
			if (!key && !row.value) {
				continue;
			}
			if (!variableKeyRegex.test(key)) {
				throw $i18n.t('Variable keys must use lowercase snake case.');
			}
			if (Object.prototype.hasOwnProperty.call(variables, key)) {
				throw $i18n.t('Variable keys must be unique.');
			}
			variables[key] = row.value ?? '';
		}
		return variables;
	};

	const submitHandler = async () => {
		let variables: Record<string, string>;
		try {
			variables = getVariablesPayload();
		} catch (error) {
			toast.error(`${error}`);
			return false;
		}

		if (name !== $user?.name) {
			if (profileImageUrl === generateInitialsImage($user?.name) || profileImageUrl === '') {
				profileImageUrl = generateInitialsImage(name);
			}
		}

		const updatedUser = await updateUserProfile(localStorage.token, {
			name: name,
			profile_image_url: profileImageUrl,
			bio: bio ? bio : null,
			gender: gender ? gender : null,
			date_of_birth: dateOfBirth ? dateOfBirth : null
		}).catch((error) => {
			toast.error(`${error}`);
		});

		const variablesRes = await updateUserVariables(localStorage.token, variables).catch((error) => {
			toast.error(`${error}`);
			return null;
		});

		if (updatedUser && variablesRes) {
			setVariableRows(variablesRes.variables ?? {});
			// Get Session User Info
			const sessionUser = await getSessionUser(localStorage.token).catch((error) => {
				toast.error(`${error}`);
				return null;
			});

			await user.set(sessionUser);
			return true;
		}
		return false;
	};

	onMount(async () => {
		const user = await getSessionUser(localStorage.token).catch((error) => {
			toast.error(`${error}`);
			return null;
		});

		if (user) {
			name = user?.name ?? '';
			profileImageUrl = user?.profile_image_url ?? '';
			bio = user?.bio ?? '';

			gender = user?.gender ?? '';
			_gender = ['', 'male', 'female'].includes(gender) ? gender : 'custom';

			dateOfBirth = user?.date_of_birth ?? '';
		}

		const userVariables = await getUserVariables(localStorage.token).catch((error) => {
			toast.error(`${error}`);
			return null;
		});
		setVariableRows(userVariables?.variables ?? {});
	});
</script>

<form
	id="tab-account"
	class="flex h-full flex-col text-sm"
	on:submit|preventDefault={async () => {
		const res = await submitHandler();

		if (res) {
			saveHandler();
		}
	}}
>
	<div class="flex-1 min-h-0 w-full overflow-y-auto scrollbar-hover pr-1.5">
		<h2 class="mb-4 text-sm font-medium text-gray-900 dark:text-white">
			{$i18n.t('settings.personal.account.title')}
		</h2>

		<UserSettingSection title={$i18n.t('settings.personal.account.sections.profile.title')} first>
			<UserProfileImage
				bind:profileImageUrl
				user={$user}
				variant="account"
				displayName={$user?.name}
			/>

			<UserSettingField
				label={$i18n.t('settings.personal.account.name.label')}
				description={$i18n.t('settings.personal.account.name.description')}
			>
				<input
					class={inputClass}
					type="text"
					bind:value={name}
					aria-label={$i18n.t('settings.personal.account.name.label')}
					required
					placeholder={$i18n.t('Enter your name')}
				/>
			</UserSettingField>

			<UserSettingField
				label={$i18n.t('settings.personal.account.bio.label')}
				description={$i18n.t('settings.personal.account.bio.description')}
			>
				<Textarea
					className={textareaClass}
					minSize={60}
					bind:value={bio}
					ariaLabel={$i18n.t('settings.personal.account.bio.label')}
					placeholder={$i18n.t('Share your background and interests')}
				/>
			</UserSettingField>

			<UserSettingField
				label={$i18n.t('settings.personal.account.gender.label')}
				description={$i18n.t('settings.personal.account.gender.description')}
			>
				<SettingsSelect
					bind:value={_gender}
					className="w-full"
					ariaLabel={$i18n.t('settings.personal.account.gender.label')}
					on:change={() => {
						console.log(_gender);

						if (_gender === 'custom') {
							// Handle custom gender input
							gender = '';
						} else {
							gender = _gender;
						}
					}}
				>
					<option value="" selected>{$i18n.t('Prefer not to say')}</option>
					<option value="male">{$i18n.t('Male')}</option>
					<option value="female">{$i18n.t('Female')}</option>
					<option value="custom">{$i18n.t('Custom')}</option>
				</SettingsSelect>

				{#if _gender === 'custom'}
					<input
						class="mt-1 {inputClass}"
						type="text"
						required
						aria-label={$i18n.t('Custom Gender')}
						placeholder={$i18n.t('Enter your gender')}
						bind:value={gender}
					/>
				{/if}
			</UserSettingField>

			<UserSettingField
				label={$i18n.t('settings.personal.account.birthDate.label')}
				description={$i18n.t('settings.personal.account.birthDate.description')}
			>
				<input
					class="{inputClass} dark:scheme-dark"
					type="date"
					aria-label={$i18n.t('settings.personal.account.birthDate.label')}
					bind:value={dateOfBirth}
				/>
			</UserSettingField>
		</UserSettingSection>

		<section class="mt-4 w-full">
			<div
				class="mb-0.5 flex items-center justify-between gap-2 text-xs text-gray-600 dark:text-gray-400"
			>
				<div class="flex min-w-0 items-center gap-1.5">
					{$i18n.t('User Variables')}
					<span class="text-gray-400 dark:text-gray-600">{variableRows.length}</span>
				</div>
				<button class={actionButtonClass} type="button" on:click={() => openVariableModal()}>
					{$i18n.t('Add')}
				</button>
			</div>
			<div class="flex flex-col gap-1 py-1">
				{#each variableRows as row, idx}
					<div class={variableRowClass}>
						<div class="min-w-0 truncate font-mono text-xs text-gray-700 dark:text-gray-300">
							{row.key || $i18n.t('key_name')}
						</div>
						<div class="min-w-0 flex-1 truncate text-xs text-gray-500 dark:text-gray-500">
							{row.value || $i18n.t('Empty')}
						</div>
						<button class={actionButtonClass} type="button" on:click={() => openVariableModal(idx)}>
							{$i18n.t('Edit')}
						</button>
					</div>
				{/each}

				{#if variableRows.length === 0}
					<div class="text-xs text-gray-400 dark:text-gray-600">
						{$i18n.t('No user variables configured.')}
					</div>
				{/if}
			</div>

			<div class="text-[0.6875rem] text-gray-400 dark:text-gray-600">
				{$i18n.t('Use these in model system prompts as {{example}}.', {
					example: '{{user.variables.key_name}}'
				})}
			</div>
		</section>

		{#if $config?.features.enable_login_form && $config?.features.enable_password_change_form}
			<UserSettingSection title={$i18n.t('settings.personal.account.sections.password.title')}>
				<UpdatePassword />
			</UserSettingSection>
		{/if}

		<UserSettingSection title={$i18n.t('Multi-factor authentication')}><Mfa /></UserSettingSection>
	</div>

	<div class="shrink-0 flex w-full justify-end pt-3 text-sm font-normal">
		<button
			class="flex h-7 shrink-0 items-center justify-center gap-1.5 rounded-lg bg-gray-900 px-2.5 text-xs font-normal text-white transition hover:bg-black disabled:opacity-60 dark:bg-gray-100 dark:text-gray-900 dark:hover:bg-white"
			type="submit"
		>
			{$i18n.t('Save')}
		</button>
	</div>
</form>

<Modal size="sm" bind:show={variableModalOpen}>
	<form class="p-4" on:submit|preventDefault={saveVariableForm}>
		<h2 class="mb-3 text-sm font-medium text-gray-900 dark:text-white">
			{variableFormIndex === null ? $i18n.t('Add User Variable') : $i18n.t('Edit User Variable')}
		</h2>

		<div class="mb-1 text-[0.625rem] text-gray-400 dark:text-gray-600">
			{$i18n.t('Key')}
		</div>
		<input
			class={inputClass}
			type="text"
			bind:value={variableFormKey}
			aria-label={$i18n.t('Variable key')}
			placeholder={$i18n.t('key_name')}
			autocomplete="off"
			spellcheck="false"
		/>

		<div class="mb-1 mt-3 text-[0.625rem] text-gray-400 dark:text-gray-600">
			{$i18n.t('Value')}
		</div>
		<Textarea
			className={variableValueClass}
			rows="6"
			minSize={132}
			bind:value={variableFormValue}
			ariaLabel={$i18n.t('Variable value')}
			placeholder={$i18n.t('Value')}
		/>

		<div class="mt-4 flex items-center justify-between gap-2">
			<div>
				{#if variableFormIndex !== null}
					<button
						class="text-xs text-gray-500 transition-colors hover:text-gray-900 dark:text-gray-500 dark:hover:text-white"
						type="button"
						on:click={deleteVariableForm}
					>
						{$i18n.t('Delete')}
					</button>
				{/if}
			</div>

			<div class="flex items-center gap-3">
				<button
					class="text-xs text-gray-500 transition-colors hover:text-gray-900 dark:text-gray-500 dark:hover:text-white"
					type="button"
					on:click={() => {
						variableModalOpen = false;
					}}
				>
					{$i18n.t('Cancel')}
				</button>
				<button
					class="flex h-7 shrink-0 items-center justify-center gap-1.5 rounded-lg bg-gray-900 px-2.5 text-xs font-normal text-white transition hover:bg-black disabled:opacity-60 dark:bg-gray-100 dark:text-gray-900 dark:hover:bg-white"
					type="submit"
				>
					{$i18n.t('Done')}
				</button>
			</div>
		</div>
	</form>
</Modal>
