<script lang="ts">
	import { PUBLIC_BACKEND_URL } from "$env/static/public";
	import {
		authState,
		Button,
		Field,
		Flex,
		Form,
		Icon,
		Modal,
		postFetch,
		sleep,
		TextArea
	} from "@davidnet-net/svelte-ui";
	import { token } from "@davidnet-net/svelte-ui/tokens";

	interface Props {
		isOpen: boolean;
		reportType: "profile" | "short";
		reportedId: string;
	}

	let { isOpen = $bindable(false), reportType, reportedId }: Props = $props();

	let reasonValue = $state("");
	let reasonInvalid = $state("");
	let isSubmitting = $state(false);
	let reportFinished = $state(false);
	let reportFailed = $state(false);

	async function submitReport(event: SubmitEvent & { currentTarget: HTMLFormElement }) {
		event.preventDefault();

		// Validate reason
		if (!reasonValue || reasonValue.trim().length === 0) {
			reasonInvalid = "Please provide a reason for this report.";
			return;
		}

		if (reasonValue.length > 2000) {
			reasonInvalid = "Reason is too long (max 2000 characters).";
			return;
		}

		reasonInvalid = "";
		isSubmitting = true;

		const payload = {
			reportType,
			reportedId,
			reason: reasonValue.trim()
		};

		const result = await postFetch(
			PUBLIC_BACKEND_URL + "/support/moderation/report",
			payload,
			undefined,
			true
		);

		await sleep(500); // UI polish for loading state

		if (result.success) {
			reportFinished = true;
		} else {
			reportFinished = true;
			reportFailed = true;
		}
	}

	function resetModal() {
		reportFinished = false;
		reportFailed = false;
		isSubmitting = false;
		reasonValue = "";
		reasonInvalid = "";
		isOpen = false;
	}
</script>

{#if isOpen && !reportFinished && authState.isLoggedIn}
	<Modal title="Report {reportType === 'profile' ? 'User Profile' : 'Short'}">
		<Flex height="100%" gap="medium" justifyContent="center" alignItems="center" direction="column">
			<Form id="report-form" autocomplete="off" onsubmit={submitReport}>
				<div
					style="color: {token.theme.color.text.secondary}; font-size: {token.global.font.size
						.small}; margin-bottom: 8px;">
					Please provide a detailed reason for reporting this {reportType}. False reports may result
					in account penalties.
				</div>
				<Field required label="Reason for reporting" name="reason" invalid={reasonInvalid}>
					<TextArea bind:value={reasonValue} maxlength={2000} disabled={isSubmitting} />
				</Field>
			</Form>
		</Flex>
		{#snippet actions()}
			<Button
				disabled={isSubmitting}
				onclick={() => {
					isOpen = false;
					reasonValue = "";
					reasonInvalid = "";
				}}>
				Cancel
			</Button>
			<Button
				form="report-form"
				type="submit"
				appearance="primary"
				loading={isSubmitting}
				disabled={reasonValue.trim().length === 0 || reasonValue.length > 2000}>
				Submit Report
			</Button>
		{/snippet}
	</Modal>
{:else if isOpen && reportFailed}
	<Modal title="Report Failed">
		<Flex height="100%" gap="medium" justifyContent="center" alignItems="center" direction="column">
			<Icon icon="error" size="giant" color="danger" />
			<span style="font-size: {token.global.font.size.large}; text-align: center">
				Failed to submit the report. Please try again later.
			</span>
		</Flex>
		{#snippet actions()}
			<Button appearance="primary" onclick={resetModal}>Close</Button>
		{/snippet}
	</Modal>
{:else if isOpen && !authState.isLoggedIn}
	<Modal title="Authentication Required">
		<Flex height="100%" gap="medium" justifyContent="center" alignItems="center" direction="column">
			<Icon icon="account_circle" size="giant" color="danger" />
			<span style="font-size: {token.global.font.size.large}; text-align: center">
				You must be logged in to submit a report.
			</span>
		</Flex>
		{#snippet actions()}
			<Button appearance="primary" onclick={resetModal}>Close</Button>
		{/snippet}
	</Modal>
{:else if isOpen && reportFinished}
	<Modal title="Report Submitted">
		<Flex height="100%" gap="medium" justifyContent="center" alignItems="center" direction="column">
			<Icon icon="check_circle" size="giant" color="success" />
			<span style="font-size: {token.global.font.size.large}; text-align: center">
				Thank you.
				<br />
				Your report has been received and will be reviewed by a moderator.
			</span>
		</Flex>
		{#snippet actions()}
			<Button appearance="primary" onclick={resetModal}>Close</Button>
		{/snippet}
	</Modal>
{/if}
