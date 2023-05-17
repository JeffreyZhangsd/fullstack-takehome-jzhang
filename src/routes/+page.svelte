<script lang="ts">
	import { cacheExchange, createClient, fetchExchange, gql, queryStore } from '@urql/svelte';
	import Loader from 'components/Loader.svelte';
	import User from 'components/User.svelte';
	import type { UserType } from 'lib/types';

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
				userList = [...userList, ...data.users];
			}
		});

		// returns users for current call
		return result;
	};
	// $result.data.users - array of objects (users)

	// initial call of query
	let result = getUsers();

	// function to load next 10
	const nextPage = () => {
		// limit += 10; // adds 10 to the query, but then query has to search from 0 to limit again
		// updates only 10 users at a time
		from += limit;
		result = getUsers();
	};

	// using bind for svelte with on:scroll on div
	let outerDiv: any;

	// update on scroll instead of using button using scroll event listener
	const handleScroll = () => {
		const { scrollTop, scrollHeight, clientHeight } = outerDiv;
		// console.log('positions', scrollTop, scrollHeight, clientHeight);
		// console.log(scrollTop + clientHeight, scrollHeight);

		// when we reach bottom of page, call load
		if (scrollTop + clientHeight >= scrollHeight) {
			nextPage();
		}

		/* remove handleScroll if no values
		sends few extra requests because when user scrolls too quickly,
		"from" might update before we add 10 new users to userList

		This could be fixed by using backend to keep track of if there are more users to load
		instead of using the front end
		*/
		if (from > userList.length + 10) {
			outerDiv.removeEventListener('scroll', handleScroll);
			console.log(from, userList.length);
		} else {
			outerDiv.addEventListener('scroll', handleScroll);
			console.log(from, userList.length);
		}
	};
</script>

<div class="w-full h-full overflow-scroll" on:scroll={handleScroll} bind:this={outerDiv}>
	<div class="flex flex-col gap-4 items-center p-4">
		<!-- Gets users and displays them -->
		{#each userList as user (user.id)}
			<User {user} />
		{/each}
		<!-- While loading, display spinner -->
		{#if $result.fetching}
			<Loader />
		{/if}
		<!-- Buffer space/ allows us to see we've reached end -->
		<div style="height:50px" />
	</div>
</div>
