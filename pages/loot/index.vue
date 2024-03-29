<template>
  <section>
    <h1>Loot</h1>
    <form
      class="flex flex-wrap sm:space-x-4"
      method="POST"
      @submit.prevent="submitSearch"
    >
      <DropDown
        class="my-2 sm:my-0 w-full sm:w-auto"
        :items="lootFeatures"
        @itemSelect="setQuery"
      />
      <input
        v-model="search"
        placeholder="Insert Loot item name"
        class="
          bg-black
          rounded-2xl
          px-4
          py-2
          text-xl
          border-4 border-double border-off-200
          w-full
          sm:w-auto
        "
        type="text"
      >
      <div class="w-full sm:w-auto self-center">
        <BButton
          class="mt-2 sm:mt-0 sm:px-4 w-full"
          type="primary"
        >
          find bags
        </BButton>
      </div>
    </form>
    <div v-if="!$fetchState.pending">
      <InfiniteScroll
        v-if="!queryLoading"
        class="
          grid grid-cols-1
          md:grid-cols-2
          lg:grid-cols-3
          xl:grid-cols-4
          2xl:grid-cols-5
          gap-4
          xl:gap-6
        "
        :content-change-key="loot.length"
        @fetchNextBlock="fetchMore"
      >
        <div v-for="(l, index) in loot" :key="index">
          <LootCard is-o-g :loot="l" />
        </div>
        <template v-if="loading">
          <Loader
            v-for="(loader, index) in 4"
            :key="'dummy' + index"
            class="mr-3 mb-3"
          />
        </template>
      </InfiniteScroll>

      <div v-if="queryLoading" class="flex flex-wrap mt-8">
        <Loader v-for="(loader, index) in 4" :key="index" class="mr-3 mb-3" />
      </div>
      <div v-if="!queryLoading && !loot.length" class="my-3">
        <div class="text-2xl">
          No Loot found - Try adjusting your query.
        </div>
      </div>
    </div>
    <div v-else class="mt-4">
      <Loader />
    </div>
  </section>
</template>

<script>
import { gql } from 'nuxt-graphql-request'
import {
  defineComponent,
  ref,
  useContext,
  useFetch
} from '@nuxtjs/composition-api'

export default defineComponent({
  setup (props, context) {
    const { $graphql } = useContext()
    const search = ref()
    const offset = ref(1)
    const query = ref(gql`
      query bagsQuery($offset: Int!) {
        bags(first: 16, skip: $offset) {
          id
          head
          neck
          chest
          hand
          ring
          weapon
          waist
          foot
          chestSuffixId
          footSuffixId
          handSuffixId
          headSuffixId
          neckSuffixId
          ringSuffixId
          waistSuffixId
          weaponSuffixId
          manasClaimed
          currentOwner {
            address
            bagsHeld
            joined
          }
        }
      }
    `)
    const getSearchQueryById = (param) => {
      return ref(gql`
        query bagsQuery($id: String) {
          bags(where: { id: $id }) {
            id
            head
            neck
            chest
            hand
            ring
            weapon
            waist
            foot
            chestSuffixId
            footSuffixId
            handSuffixId
            headSuffixId
            neckSuffixId
            ringSuffixId
            waistSuffixId
            weaponSuffixId
            manasClaimed
            currentOwner {
              address
              bagsHeld
              joined
            }
          }
        }
      `)
    }

    const getSearchQuery = (param) => {
      return ref(gql`
      query bagsQuery($offset: Int!, $search: String) {
        bags(first: 100, skip: $offset, where: { ${param}_contains: $search }) {
          id
          head
          neck
          chest
          hand
          ring
          weapon
          waist
          foot
          chestSuffixId
          footSuffixId
          handSuffixId
          headSuffixId
          neckSuffixId
          ringSuffixId
          waistSuffixId
          weaponSuffixId
          manasClaimed
          currentOwner {
            address
            bagsHeld
            joined
          }
        }
      }
    `)
    }

    const loot = ref(null)

    const lootFeatures = [
      {
        name: 'id'
      },
      {
        name: 'head'
      },
      {
        name: 'neck'
      },
      {
        name: 'chest'
      },
      {
        name: 'hand'
      },
      {
        name: 'ring'
      },
      {
        name: 'weapon'
      },
      {
        name: 'waist'
      },
      {
        name: 'foot'
      }
    ]

    const lootQuery = ref(lootFeatures[0])
    const queryLoading = ref(false)
    const loading = ref(false)

    const setQuery = (value) => {
      lootQuery.value = value
    }

    const toSentenceCase = (str) => {
      return str
        .replace(/\w\S*/g, function (txt) {
          return txt.charAt(0).toUpperCase() + txt.substr(1).toLowerCase()
        })
        .replace('Of', 'of')
    }

    const searchBy = async () => {
      queryLoading.value = true

      try {
        if (lootQuery.value.name === 'id') {
          const newQuery = getSearchQueryById(lootQuery.value.name)
          const response = await $graphql.ecosystem.request(newQuery.value, {
            id: search.value
          })
          loot.value = response.bags
        } else {
          const newQuery = getSearchQuery(lootQuery.value.name)
          const response = await $graphql.ecosystem.request(newQuery.value, {
            offset: offset.value,
            search: toSentenceCase(search.value)
          })
          loot.value = response.bags
        }
      } catch (e) {
        console.log(e)
      } finally {
        queryLoading.value = false
      }
    }

    const submitSearch = () => {
      searchBy()
    }

    useFetch(async () => {
      const response = await $graphql.ecosystem.request(query.value, {
        offset: offset.value
      })
      loot.value = response.bags
    })

    const fetchMore = async () => {
      loading.value = true
      offset.value = offset.value + 100
      try {
        if (search.value) {
          const newQuery = getSearchQuery(lootQuery.value.name)
          const response = await $graphql.ecosystem.request(newQuery.value, {
            offset: offset.value,
            search: search.value
          })
          loot.value = loot.value.concat(response.bags)
        } else {
          const response = await $graphql.ecosystem.request(query.value, {
            offset: offset.value
          })
          loot.value = loot.value.concat(response.bags)
        }
      } catch (e) {
        console.log(e)
      } finally {
        loading.value = false
      }
    }

    return {
      loot,
      search,
      submitSearch,
      fetchMore,
      loading,
      lootFeatures,
      setQuery,
      lootQuery,
      queryLoading,
      offset
    }
  }
})
</script>
