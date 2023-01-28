<script lang="ts">
	import Aside from '$lib/components/glue/Aside.svelte';
	import Main from '$lib/components/glue/Main.svelte';
	import PageContainer from '$lib/components/glue/PageContainer.svelte';
	import TextInput from '$lib/components/glue/TextInput.svelte';
	import { pb } from '$lib/glue/pocketbase';
	import IconAdd from '$lib/icons/glue/IconAdd.svelte';
	import IconDelete from '$lib/icons/glue/IconDelete.svelte';
	import dynamicAgo from '$lib/util/glue/dynamicAgo';
	import { differenceInDays } from 'date-fns';
	import { onMount } from 'svelte';
	import Youtube from 'svelte-youtube-embed';

	let channels = [];
	let videos = [];

	let newChannelName = '';
	let newChannelId = '';

	let deleteChannelId = '';

	onMount(async () => {
		fetchChannels();
	});

	const fetchChannels = async () => {
		channels = await pb.collection('channels').getFullList(200, {
			sort: '-created'
		});
	};

	const addChannel = async () => {
		const newChannel = await pb.collection('channels').create({
			name: newChannelName,
			channelId: newChannelId
		});
		channels = [newChannel, ...channels];
		newChannelName = '';
		newChannelId = '';
	};

	const syncChannelVideos = async (channelId) => {
		try {
			const url = `https://www.googleapis.com/youtube/v3/search?key=${
				import.meta.env.VITE_GOOGLE_API_KEY
			}&channelId=${channelId}&part=snippet,id&order=date&maxResults=4`;
			const res = await (await fetch(url)).json();

			const promises = res?.items?.map(async (video) => {
				const isExisting = videos?.some((existingVideo) => existingVideo?.id === video?.id);

				if (isExisting) return [];

				const newVideo = await pb.collection('videos').create({
					title: video?.snippet?.title,
					publishedAt: video?.snippet?.publishedAt,
					videoId: video?.id?.videoId
				});

				return newVideo;
			});

			return await Promise.all(promises);
		} catch (error) {
			return [];
		}
	};

	const syncVideos = async (targetChannels) => {
		const promises = targetChannels?.map(async (channel) => {
			const daysSinceFetch = differenceInDays(new Date(), new Date(channel?.lastFetchedDate));

			if (channel?.isEnabled && daysSinceFetch >= 1) {
				return syncChannelVideos(channel?.channelId);
			}

			return [];
		});
		const res = await Promise.all(promises);

		if (!res) return [];

		const newVideos = res
			?.flat()
			?.filter((video) => video)
			?.map((video) => ({ ...video, publishedAt: new Date(video?.publishedAt) }))
			?.sort((a, b) => {
				return b?.publishedAt?.getTime() - a?.publishedAt?.getTime();
			});
		videos = [...newVideos, ...videos];
	};

	$: (async () => {
		videos = (
			await pb.collection('videos').getList(1, 20, {
				sort: '-publishedAt'
			})
		)?.items;
		await syncVideos(channels);
	})();

	const handleDeleteChannel = async () => {
		if (deleteChannelId) {
			pb.collection('channels').delete(deleteChannelId);
			channels = channels?.filter((channel) => channel?.id !== deleteChannelId);
		}
	};

	const handleSetEnabled = async (channelId, value) => {
		pb.collection('channels').update(channelId, {
			isEnabled: value
		});
		channels = channels?.map((channel) => {
			if (channel?.id === channelId)
				return {
					...channel,
					isEnabled: value
				};
			return channel;
		});
	};
</script>

<PageContainer title="Home" layout="aside-main">
	<Aside>
		<div class="prose hidden space-y-4 md:block">
			<h3>Channels</h3>
			<div class="space-y-1">
				<TextInput placeholder="New channel name" bind:value={newChannelName} class="input-sm" />
				<TextInput placeholder="New channel ID" bind:value={newChannelId} class="input-sm" />
				<div class="flex justify-end">
					<button class="btn-xs btn" on:click={addChannel}><IconAdd /> Add channel</button>
				</div>
			</div>
			<div class="space-y-2">
				{#each channels as channel (channel?.id)}
					<div class="prose rounded-xl bg-base-200 py-3 px-4">
						<div class="mb-1 flex items-center justify-between">
							<div class="flex items-center space-x-2">
								<h4 class="m-0">{channel?.name}</h4>
								<input
									type="checkbox"
									class="toggle-success toggle toggle-xs"
									checked={channel?.isEnabled}
									on:input={() => handleSetEnabled(channel?.id, !channel?.isEnabled)}
								/>
							</div>
							<label
								for="modal-delete-channel"
								class="button btn-xs btn text-lg"
								on:click={() => {
									deleteChannelId = channel?.id;
								}}
							>
								<IconDelete />
							</label>
						</div>
						<p class="text-xs">{channel?.channelId}</p>
					</div>
				{/each}
			</div>
		</div>
	</Aside>
	<Main>
		<div class="space-y-8 p-2">
			{#each videos as video (video?.id)}
				<div class="">
					<div class="overflow-hidden rounded-xl [&_h3]:hidden">
						<Youtube id={video?.id} />
					</div>
					<div class="prose mt-2 ml-1">
						<h3 class="mb-1">{video?.title}</h3>
						<p class="text-sm">
							{dynamicAgo({
								date: video?.publishedAt
							})}
						</p>
					</div>
				</div>
			{/each}
		</div>
	</Main>

	<!-- modal: delete channel -->
	<input type="checkbox" id="modal-delete-channel" class="modal-toggle" />
	<label for="modal-delete-channel" class="modal cursor-pointer">
		<label class="prose modal-box relative w-[20rem]" for="">
			<h2 class="text-lg font-bold">Delete channel?</h2>
			<div class="modal-action">
				<label
					for="modal-delete-channel"
					class="btn-error btn-block btn"
					on:click={handleDeleteChannel}>Delete</label
				>
			</div>
		</label>
	</label>
</PageContainer>
