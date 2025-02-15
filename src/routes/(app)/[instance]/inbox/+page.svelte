<Title title="Inbox" />

<BusyButton
	cl="tertiary"
	busy={$markAllReadState.busy}
	on:click={$markAllReadState.submit}
	disabled={$unreadCount === 0}>Mark All As Read</BusyButton
>

<form method="GET" use:navigateOnChange bind:this={formEl}>
	<section>
		<Stack gap={4} align="center" cl="py-4" dir="r" justify="between">
			<Stack gap={4} align="center" dir="r">
				<Select name="type" bind:value={queryType} label="Type">
					{#each InboxTypes as opt}
						<option value={opt.value}>{opt.label}</option>
					{/each}
				</Select>
				<Select name="listing" value={data.query.listing} label="Listing">
					{#each InboxListings as opt}
						<option value={opt.value}>{opt.label}</option>
					{/each}
				</Select>
				<Select label="Sort" name="sort" required value={data.query.sort}>
					{#each InboxSortOptions as opt}
						<option value={opt.value}>{opt.label}</option>
					{/each}
				</Select>
			</Stack>
			<BusyButton on:click={$refreshState.submit} busy={$refreshState.busy} cl="tertiary" type="button"
				>Refresh</BusyButton
			>
		</Stack>
	</section>
</form>

<Stack dir="c" gap={2}>
	<ContentViewProvider store={cvStore}>
		<VirtualFeed
			feedSize={$cvStore.length}
			on:more={more}
			{endOfFeed}
			feedEndIcon="file-circle-xmark"
			feedEndMessage="No more messages!"
			loading={loadingContent}
			loadMoreFailed={loadingContentFailed}
		>
			<div slot="feed-end-banner">
				{#if data.query.type === 'Unread'}
					<button class="secondary" on:click={onGotoRead}>
						<Icon icon="envelope-circle-check" /> View Read
					</button>
				{/if}
			</div>
			<svelte:fragment let:index>
				{@const content = $cvStore[index]}
				{#if content.type === 'mention' || content.type === 'reply'}
					<Comment contentView={content} postOP="" showPost postLocked={content.view.post.locked}>
						<InboxReadButton {content} slot="actions-start" />
					</Comment>
				{:else if content.type === 'message'}
					<PrivateMessage privateMessageView={content.view}>
						<svelte:fragment slot="actions-start" let:toMe>
							{#if toMe}
								<InboxReadButton {content} />
							{/if}
						</svelte:fragment>
					</PrivateMessage>
				{/if}
				{#if index + 1 < $cvStore.length}
					<hr class="w-100" />
				{/if}
			</svelte:fragment>
		</VirtualFeed>
	</ContentViewProvider>
</Stack>

<script lang="ts">
	import { Stack, Select, Icon } from 'sheodox-ui';
	import { InboxListings, InboxSortOptions, InboxTypes } from '$lib/feed-filters';
	import Comment from '$lib/comments/Comment.svelte';
	import PrivateMessage from '$lib/PrivateMessage.svelte';
	import BusyButton from '$lib/BusyButton.svelte';
	import ContentViewProvider from '$lib/ContentViewProvider.svelte';
	import InboxReadButton from '$lib/InboxReadButton.svelte';
	import type { PageData } from './$types';
	import { getAppContext } from '$lib/app-context';
	import Title from '$lib/Title.svelte';
	import VirtualFeed from '$lib/VirtualFeed.svelte';
	import { feedLoader } from '$lib/post-loader';
	import type { CommentSortType } from 'lemmy-js-client';
	import { createStatefulAction, navigateOnChange, parseDate } from '$lib/utils';
	import { invalidateAll } from '$app/navigation';
	import {
		createContentViewStore,
		mentionViewToContentView,
		messageViewToContentView,
		replyViewToContentView
	} from '$lib/content-views';
	import { profile } from '$lib/profiles/profiles';
	import { tick } from 'svelte';

	export let data;

	let queryType = data.query.type;

	let formEl: HTMLFormElement;

	const { unreadCount, checkUnread } = getAppContext();
	$: client = $profile.client;
	$: jwt = $profile.jwt;

	const cvStore = createContentViewStore();

	let loadingContent = false,
		loadingContentFailed = false,
		endOfFeed = false,
		loader: ReturnType<typeof initFeed>;

	$: refreshState = createStatefulAction(async () => {
		checkUnread();
		loader = initFeed(data);
		cvStore.clear();
		await more();
	});

	$: markAllReadState = createStatefulAction(async () => {
		if (!jwt) {
			return;
		}

		await client.markAllAsRead();
		invalidateAll();
		checkUnread();
	});

	$: {
		checkUnread();
		loader = initFeed(data);
		// load the first page of data
		more();
	}

	type InboxData = Awaited<ReturnType<typeof fetchInboxPage>>;

	async function onGotoRead() {
		queryType = 'All';

		await tick();
		formEl.dispatchEvent(new Event('change'));
	}

	function initFeed(data: PageData) {
		const newLoader = feedLoader<InboxData>(
			async (page) => {
				return await fetchInboxPage(page, data);
			},
			(res) => (res ? res.replies.length + res.mentions.length + res.messages.length : 0)
		);

		loadingContent = false;
		loadingContentFailed = false;
		endOfFeed = false;
		cvStore.clear();

		return newLoader;
	}

	async function fetchInboxPage(page: number, data: PageData) {
		if (!jwt) {
			return;
		}

		const form = {
			sort: data.query.sort as CommentSortType,
			unread_only: data.query.type === 'Unread',
			page,
			limit: 20
		};

		const isListing = (listing: string) => data.query.listing === listing || data.query.listing === 'All';

		const [replies, mentions, messages] = await Promise.all([
			isListing('Replies') ? client.getReplies(form).then((c) => c.replies) : [],
			isListing('Mentions') ? client.getPersonMentions(form).then((c) => c.mentions) : [],
			isListing('Messages') ? client.getPrivateMessages(form).then((c) => c.private_messages) : []
		]);

		return {
			replies,
			mentions,
			messages
		};
	}

	async function more() {
		if (endOfFeed || loadingContent) {
			return;
		}
		loadingContent = true;

		const qs = location.search;
		const feedData = (await loader.next()).value;
		if (qs === location.search) {
			loadingContentFailed = feedData.error;
			endOfFeed = feedData.endOfFeed;
			if (!feedData.error && feedData.response) {
				cvStore.append(getContentViews(feedData.response));
			}

			loadingContent = false;
		}
	}

	function getContentViews(inboxData: InboxData) {
		if (!inboxData) {
			return [];
		}

		const replies = inboxData.replies.map(replyViewToContentView),
			mentions = inboxData.mentions.map(mentionViewToContentView),
			messages = inboxData.messages.map(messageViewToContentView);

		if (data.query.listing === 'All') {
			return [...replies, ...mentions, ...messages].sort((a, b) => {
				const aPublished = parseDate(a.published).getTime(),
					bPublished = parseDate(b.published).getTime();

				if (data.query.sort === 'New') {
					return bPublished - aPublished;
				} else if (data.query.sort === 'Old') {
					return aPublished - bPublished;
				} else {
					return b.id - a.id;
				}
			});
		}

		switch (data.query.listing) {
			case 'Replies':
				return replies;
			case 'Mentions':
				return mentions;
			case 'Messages':
				return messages;
		}

		return [];
	}
</script>
