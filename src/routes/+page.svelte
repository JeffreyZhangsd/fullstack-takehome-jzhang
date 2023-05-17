<script lang="ts">
	import { cacheExchange, createClient, fetchExchange, gql, queryStore } from '@urql/svelte';
	import Loader from 'components/Loader.svelte';
	import User from 'components/User.svelte';
	import type { UserType } from 'lib/types';
	import Layout from './+layout.svelte';
	import { Result } from 'postcss';

	const client = createClient({
		url: '/graphql',
		exchanges: [cacheExchange, fetchExchange]
	});

	// num of results per load, init user list to store loaded users
	let limit = 10;
	let from = 0;
	let userList: UserType[] = [];

	// from urql/svelt documentation
	// ($from: Int!, $limit: Int!)
	// (from: $from, limit: $limit)

	// function to get 10 users and return result query store
	const getUsers = () => {
		const result = queryStore<{ users: UserType[] }>({
			client,
			query: gql`
				query Users($from: Int!, $limit: Int!) {
					users(from: $from, limit: $limit) {
						id
						name
						avatar
						email
					}
				}
			`,
			variables: { from, limit }
		});

		// when result is updated, update userlist with the list of new users
		result.subscribe(({ data }) => {
			if (data) {
				userList = userList.concat(data.users);
				console.log(userList);
			}
		});

		// returns users for current call
		return result;
	};

	// $result.data.users - array of objects (users)
	// initial call of query
	let result = getUsers();

	const nextPage = () => {
		// limit += 10; // adds 10 to the query, but then query has to search from 0 to limit again
		// updates only 10 users at a time
		from += limit;
		result = getUsers();
	};
</script>

<div class="w-full h-full overflow-scroll">
	<div class="flex flex-col gap-4 items-center p-4">
		<!-- Gets users and displays them -->
		{#each userList as user (user.id)}
			<User {user} />
		{/each}
		{#if $result.fetching}
			<!-- Spinner that appears on load (while fetching data) -->
			<Loader />
		{:else if $result.data && from < userList.length}
			<button on:click={nextPage}>test load {from}</button>
		{/if}
	</div>
</div>
