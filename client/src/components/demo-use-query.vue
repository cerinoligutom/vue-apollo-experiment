<template>
  <h2>vue-apollo Query</h2>

  <div class="row alert alert-info m-0 mb-3">
    <div class="col-md-12">I recommend opening the developer tools and monitoring the console logs while you play with this demo.</div>
    <hr class="my-2"/>
    <div class="col-md-12">
      Changing any of the properties from a reactive variable passed to a
      <code>useQuery</code>
      s' parameter will trigger a refetch. Same goes with the
      <code>variables</code>
      returned by the
      <code>useQuery</code>
      . Trying playing with the parameters and see it in action. Afterwards, try fetching more data.
    </div>
  </div>

  <div class="row mb-3">
    <div class="col-md-3">
      <label for="numberOfItems" class="form-label">Number of users to fetch</label>
      <input v-model="variables!.first" :min="0" :max="30" type="number" class="form-control" id="numberOfItems" />
    </div>

    <div class="col-md-3">
      <label for="sortField" class="form-label">Sort Field</label>
      <select v-model="variables!.sortBy.field" class="form-select" id="sortField">
        <option :value="null"></option>
        <option v-for="option of UserSortFieldOptions" :key="option">{{ option }}</option>
      </select>
    </div>

    <div class="col-md-3">
      <label for="sortDirection" class="form-label">Sort Direction</label>
      <select v-model="variables!.sortBy.direction" class="form-select" id="sortDirection">
        <option :value="null"></option>
        <option v-for="option of SortDirectionOptions" :key="option">{{ option }}</option>
      </select>
    </div>

    <div class="col-md-3 fetch-more-btn-container">
      <button @click="fetchMoreItems()" type="button" class="btn btn-success me-1">Fetch More</button>
      <button @click="refetch()" type="button" class="btn btn-success">Refetch</button>
    </div>
  </div>

  <div class="row alert alert-warning m-0">
    <div v-if="users.length > (variables?.first ?? 0)" class="col-md-12 alert alert-danger">
      Both the
      <code>customQueryVariables</code>
      that we manually created and the
      <code>variables</code>
      returned by
      <code>useQuery</code>
      do not get updated when using
      <code>fetchMore()</code>
      . However, if you check the console logs, you can see that the variables from using
      <code>fetchMore()</code>
      now has the
      <code>after</code>
      property populated. See code comments on
      <code>updateQuery()</code>
      of this component for more info.
    </div>
    <div class="col-md-6">
      <div><code>variables (from useQuery):</code></div>
      <pre>{{ variables }}</pre>
    </div>
    <div class="col-md-6">
      <div><code>customQueryVariables (created manually):</code></div>
      <pre>{{ customQueryVariables }}</pre>
    </div>
  </div>

  <UserTable :users="users" :isLoading="isLoading" />

  <template v-if="!isLoading && error">
    <div class="alert alert-warning">
      Do note that if you try to interpolate the
      <code>ApolloError</code>
      object directly, it just prints the message.
    </div>
    <pre class="text-danger">{{ error }}</pre>
    <div class="alert alert-warning">For advanced use cases, you might want to do something with the full object.</div>
    <pre class="text-danger">{{ JSON.stringify(error, null, 2) }}</pre>
  </template>
</template>

<script lang="ts">
import { defineComponent, reactive, computed } from 'vue';
import { useQuery, useResult } from '@vue/apollo-composable';
import UserTable from './user-table.vue';
import {
  GetUsersDocument,
  GetUsersQuery,
  GetUsersQueryVariables,
  SortDirection,
  UserSortField,
  useGetUsersQuery,
} from '@/generated/graphql';
import { useGetUsersQuery as useWrappedGetUsersQuery } from '@/composables/use-get-users-query';

export const DemoUseQuery = defineComponent({
  name: 'DemoUseQuery',
  components: {
    UserTable,
  },
  setup() {
    /**
     * You might be tempted to do this. Just know that this can have a side effect that you may not want.
     * Particularly, if this is passed as a variable to a useQuery and any of the props here change, it'd
     * trigger a refetch on the query. Ultimately, YMMV.
     */
    const customQueryVariables = reactive<GetUsersQueryVariables>({
      first: 5,
      sortBy: { direction: SortDirection.ASC, field: UserSortField.NAME },

      // For cursor pagination
      after: undefined,
      before: undefined,
    });

    /**
     * Basic usage of vue-apollo to make Query graphql requests is to use the useQuery composable.
     * It has generic slots for TypeScript use and we have to supply this so that we can't get code completion
     * and strict type checking. First one is the query document, second one is the variables.
     * Normally you'd be defining these yourself but that's a chore and prone to human errors so we use
     * GraphQL Code Generator to generate them for us based on a GraphQL Document (defined on queries.graphql).
     *
     * The parameters should be self explanatory.
     */
    useQuery<GetUsersQuery, GetUsersQueryVariables>(
      GetUsersDocument,
      customQueryVariables,
      // Refer to docs on how to use the 3rd parameter for useQuery: https://v4.apollo.vuejs.org/api/use-query.html#parameters
      {
        // But for now, ignore this `enabled` property.
        enabled: false,
      },
    );

    /**
     * The query generated by the GraphQL Code Generator which basically wraps the vue-apollo way nicely with TypeScript.
     * If you check the signature of the function below, it already has types. Furthermore, if you check the implementation
     * of the function, it's similar to what we did above (vue-apollo way) and we don't have to supply the
     * graphql document anymore.
     */
    useGetUsersQuery(customQueryVariables, { enabled: false });

    const {
      // Typical variables you'll use
      result, // The response from the GraphQL Server but with just the data wrapped with a ref
      loading: isLoading, // Self-explanatory. The variable CAN react as well to refetches but you need to enable `notifyOnNetworkStatusChange` in the options.
      error, // Any GraphQL errors goes into this object.

      /**
       * The variables used in the GraphQL query, you may not be interested on using this if you use a
       * different ref/reactive variable to manage the query's variables like in this component
       * demo where we have a `customQueryVariables` reactive variable. I'd highly recommend you read the docs
       * as how you pass your variable will be different depending on what you pass (e.g. ref, reactive, plain object).
       * https://v4.apollo.vuejs.org/guide-composable/query.html#variables
       *
       * IMPORTANT: This will refetch the query each time a property from the `variables` object changes.
       */
      variables,

      // Event hooks you can attach to if you want to do something when these events occur.
      onError,
      onResult,

      /**
       * Used for pagination.
       * https://v4.apollo.vuejs.org/guide-composable/pagination.html#using-fetchmore
       */
      fetchMore,

      /**
       * You could use this if you have use cases like
       * - retrying a failed query due to network issues (make sure to pass a variable to its parameter to refetch the same set of data)
       * - just refetch the query from scratch (if no parameters are passed, it'll use the latest state of the variables)
       */
      refetch,
    } = useWrappedGetUsersQuery(customQueryVariables, {
      /**
       * There are multiple fetch policy options here and you might want to know each one of them.
       * You should use what's appropriate for your use case.
       * https://www.apollographql.com/docs/react/data/queries/#supported-fetch-policies
       *
       *
       * On a side note, you might want to look into this as well:
       * https://github.com/apollographql/apollo-client/issues/6916#issuecomment-811203002
       */
      fetchPolicy: 'cache-and-network',
      nextFetchPolicy: 'cache-first',

      // So that `loading` gets updated when `fetchMore()` is triggered.
      notifyOnNetworkStatusChange: true,
    });

    onError((err) => {
      // Handle accordingly when errors occur
      console.error(err);
    });
    onResult((res) => {
      // Do some side effects when the query is successful
      console.log('Result from DemoUseQuery:', res);
    });

    // Type helpers
    type UserQueryNode = Exclude<GetUsersQuery['users']['edges'][0], null | undefined>['node'];
    type UserQueryPageInfo = GetUsersQuery['users']['pageInfo'];

    /**
     * HANDLING RESULT - APPROACH #1 (Ideal for JavaScript and/or no pre-processing)
     *
     * Refers to docs on why you'd be interested on using `useResult()`
     * as there multiple ways to use this depending on your use case.
     * https://v4.apollo.vuejs.org/guide-composable/query.html#useresult
     *
     * You could use directly the `result` object from the Query but that comes
     * with some caveats and it'd be in your best interest to know when to use what
     * if you do decide to use this.
     */
    const usersFromUseResult = useResult(
      result,
      /**
       * Not sure how to do this more elegantly but the current version infers this as type `never[]`
       * unless we do Type Assertion. And even then, it's not the exact mapping we want since as you
       * can see (hover on the resulting variable), the `__typename` is optional on the asserted type
       * yet that seems to have been removed on the callback's parameter (`res` in this case).
       */
      [] as UserQueryNode[],
      (res) => res.users.edges.map((x) => x.node),
    );
    const pageInfoFromUseResult = useResult(
      result,
      /**
       * IMPORTANT: There are also instances like this where it just isn't compatible (try removing the ts-ignore annotation line).
       * You probably wouldn't want it not typed as it'd resort back to `never[]` and we're back to the original issue.
       */
      // @ts-ignore
      [] as UserQueryPageInfo,
      (res) => res.users.pageInfo,
    );

    /**
     * HANDLING RESULT - APPROACH #2 (Ideal for TypeScript and if you need pre-processing)
     *
     * The other approach we can do is make use of the `computed` from Vue and handle things on our own
     * in a sense that we get a more consistent type inference from TypeScript and more hands-on in what
     * to return like for example, we could do some pre-processing here if we want to such as converting
     * API Response DTOs into our frontend app's View Models or other types of Models.
     *
     * This is the approach we'll be using in this demo as the type inference is more reliable.
     */
    const usersFromComputed = computed<Array<UserQueryNode | undefined>>(() => result.value?.users?.edges?.map((x) => x?.node) ?? []);
    const pageInfoFromComputed = computed<UserQueryPageInfo>(
      () =>
        result.value?.users?.pageInfo ?? {
          hasNextPage: false,
          hasPreviousPage: false,
          startCursor: undefined,
          endCursor: undefined,
        },
    );

    /**
     * HANDLING RESULT - APPROACH #3 (Not Recommended - Personally)
     *
     * Just return the `result` object here on the setup and handle the logic on the template side.
     * This is just a comment so don't take the next lines to be related to this approach (#3).
     */

    /**
     * Load more users.
     */
    async function fetchMoreItems() {
      await fetchMore({
        variables: {
          after: pageInfoFromComputed.value.endCursor,
        },
        /**
         * IMPORTANT: The updateQuery callback for fetchMore() is DEPRECATED and will be REMOVED in the next major version of Apollo Client.
         * You might not want to use this if you're setting up a fresh Apollo Client but it'll be good to know how to use it as a lot of
         * projects that uses Apollo Client v2 (and some from v3) still uses this approach.
         *
         * `updateQuery` can be used to update the result object of this query when using `fetchMore()` (old way of doing it w/ Apollo Client 2).
         * Please read the implementation from the wrapped composable for an example of its usage.
         *
         * An example of the new approach, making use of Field Type Policies, can be found on `src/plugins/apollo.ts`.
         */
        // updateQuery: () => throw new Error('Please read the implementation from the composable'),
      });
    }

    return {
      result,
      users: usersFromComputed,
      isLoading,
      error,
      variables,
      customQueryVariables,
      SortDirectionOptions: SortDirection,
      UserSortFieldOptions: UserSortField,
      fetchMoreItems,
      refetch,
    };
  },
});
export default DemoUseQuery;
</script>

<style lang="scss" scoped>
.fetch-more-btn-container {
  display: flex;
  align-items: center;
  justify-content: center;
}
</style>
