<Title title={nameAtInstance(communityView.community, communityView.community.title)} />
{#key data}
	<ContentViewProvider store={cvStore}>
		<PostsPage
			on:more={more}
			on:refresh={() => refresh(data)}
			feedType="community"
			{communityView}
			{moderators}
			{endOfFeed}
			selectedType={data.query.type}
			selectedListing={data.query.listing}
			selectedSort={data.query.sort}
			{loadingContent}
			{loadingContentFailed}
		/>
	</ContentViewProvider>
{/key}

<script lang="ts">
	import PostsPage from '$lib/feeds/posts/PostsPage.svelte';
	import { postCommentFeedLoader } from '$lib/post-loader.js';
	import ContentViewProvider from '$lib/ContentViewProvider.svelte';
	import type { PageData } from './$types';
	import Title from '$lib/Title.svelte';
	import { loadFeedData } from '$lib/feed-query';
	import { createContentViewStore } from '$lib/content-views';
	import { nameAtInstance } from '$lib/nav-utils';
	import { getCommunityContext } from '$lib/community-context/community-context';

	export let data;
	$: communityView = data.communityRes.community_view;
	$: moderators = data.communityRes.moderators;

	const cvStore = createContentViewStore(),
		communityContext = getCommunityContext();

	$: communityContext.updateCommunity(data.communityRes);

	let loadingContent = false,
		loadingContentFailed = false,
		endOfFeed = false;

	let loader: ReturnType<typeof initFeed>;
	$: {
		refresh(data);
	}

	function refresh(d: PageData) {
		loader = initFeed(d);
		cvStore.clear();
		// load the first page of data
		more();
	}

	function initFeed(data: PageData) {
		if (communityView.blocked) {
			endOfFeed = true;
			return;
		}
		const newLoader = postCommentFeedLoader({
			type: data.query.type,
			queryFn: async (page: number) => {
				return await loadFeedData({
					page,
					listing: data.query.listing,
					sort: data.query.sort,
					type: data.query.type,
					communityName: data.communityName
				});
			}
		});

		loadingContent = false;
		loadingContentFailed = false;
		endOfFeed = false;

		return newLoader;
	}

	async function more() {
		if (endOfFeed || loadingContent || !loader) {
			return;
		}
		loadingContent = true;

		const qs = location.search;
		const feedData = (await loader.next()).value;
		if (qs === location.search) {
			loadingContentFailed = feedData.error;
			endOfFeed = feedData.endOfFeed;
			cvStore.append(feedData.contentViews);

			loadingContent = false;
		}
	}
</script>
