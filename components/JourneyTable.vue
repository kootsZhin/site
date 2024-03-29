<template>
  <div
    id="stake"
    class="flex flex-wrap justify-center text-center container mx-auto"
  >
    <div class="p-2 sm:w-1/2 w-full">
      <BaseBox class="h-80">
        <h1 class="text-center">
          Your Unstaked Realms
        </h1>
        <p>
          You need to be staked for an entire epoch to be entitled to your Lords
        </p>
        <p class="text-6xl my-auto">
          {{ numberRealms || 0 }}
        </p>
        <BButton
          v-if="!account"
          class="w-full mt-auto"
          type="settling"
          @click="open"
        >
          Connect Account
        </BButton>
        <BButton
          v-else
          class="w-full mt-auto"
          type="settling"
          :disabled="numberRealms == 0"
          @click="stakeRealms"
        >
          Board the Ship (stake)
        </BButton>
      </BaseBox>
    </div>
    <div class="p-2 sm:w-1/2 w-full">
      <BaseBox class="h-80">
        <h1>Your Staked Realms</h1>
        <p class="text-6xl my-auto">
          {{ bridgedRealms || 0 }}
        </p>
        <BButton
          v-if="!account"
          class="w-full mt-auto"
          type="settling"
          @click="open"
        >
          Connect Account
        </BButton>
        <BButton
          v-else
          class="w-full mt-auto"
          type="settling"
          @click="unstakeRealms"
        >
          Leave Ship (unstake)
        </BButton>
      </BaseBox>
    </div>
    <div class="p-2 sm:w-1/2 w-full">
      <BaseBox class="h-80">
        <h1>Your Lords Available To Claim</h1>
        <p>You will earn 625 per Realm per full epoch staked</p>
        <p class="text-6xl mx-auto my-auto">
          <LoadingRings
            v-if="loading.lords"
            class="mx-auto fill-none stroke-current text-off-200 w-32"
          />
          <span v-else>{{ claimableBalance }}</span>
        </p>
        <BButton
          v-if="!account"
          class="w-full mt-auto"
          type="settling"
          @click="open"
        >
          Connect Account
        </BButton>
        <BButton
          v-else
          class="w-full mt-auto"
          type="settling"
          :disabled="claimableBalance == '0'"
          @click="claimAllLords"
        >
          Claim Lords
        </BButton>
        <div class="w-full mt-2">
          <span
            class="cursor-pointer hover:underline"
            @click="isLordsAdded"
          >Add LORDS to MetaMask</span>
        </div>
      </BaseBox>
    </div>
    <div class="p-2 sm:w-1/2 w-full">
      <BaseBox class="h-80">
        <h1>Current Epoch</h1>
        <p>An epoch is 1 week</p>
        <p class="text-6xl text-center my-auto">
          {{ epoch }}
        </p>
        <no-ssr>
          <vac
            v-if="timeLeft"
            class="text-center"
            :end-time="new Date().getTime() + timeLeft * 1000"
          >
            <span slot="process" slot-scope="{ timeObj }">{{
              `Time Left in Epoch: ${timeObj.d} days ${timeObj.h} hrs ${timeObj.m} mins ${timeObj.s} seconds`
            }}</span>
          </vac>
        </no-ssr>
      </BaseBox>
    </div>
  </div>
</template>

<script>
import { useWeb3 } from '@instadapp/vue-web3'
import {
  defineComponent,
  onMounted,
  watch,
  computed,
  ref
} from '@nuxtjs/composition-api'
import { useModal } from '~/composables/useModal'
import { useNetwork, activeNetworkId } from '~/composables/useNetwork'
import { useRealms } from '~/composables/useRealms'
import { useStaking } from '~/composables/useStaking'
import { useIncentive } from '~/composables/useIncentive'
import JourneySettling from '~/components/modal/JourneySettling.vue'
import { useWeb3Modal } from '~/composables/useWeb3Modal'
import LoadingRings from '~/assets/img/loadingRings.svg?inline'

export default defineComponent({
  components: {
    LoadingRings
  },
  layout: 'settling',

  setup () {
    const tab = ref(false)
    const { getWalletRealms, userRealms } = useRealms()
    const { account } = useWeb3()
    const { open } = useWeb3Modal()
    const {
      rewardInfo,
      lpPositions,
      getLp,
      withdraw,
      deposit,
      getRewards,
      userRewards,
      claim,
      fetchUserPositions,
      userPositions,
      loading: loadingIncentive,
      getIncentivesForPool,
      poolIncentives
    } = useIncentive()
    const { checkForNetworkMismatch, networkMismatch, useL1Network } =
      useNetwork()
    const {
      claimableBalance,
      loading,
      getClaimableLordsBalance,
      getEpoch,
      epoch,
      getTimeToNextEpoch,
      claimAllLords,
      timeLeft,
      getTotalRealmsStaked,
      totalRealmsStaked,
      isLordsAdded
    } = useStaking()
    const { showComponent } = useModal()

    const v1PositionsFilter = computed(() => {
      return userPositions.value.filter(
        a => a.staked && a.incentivePositions.length === 1
      )
    })
    onMounted(async () => {
      activeNetworkId.value = useL1Network.value.id
      await getIncentivesForPool()

      if (account.value) {
        if (networkMismatch.value) {
          checkForNetworkMismatch()
        }
        await getWalletRealms({ address: account.value, first: 1000 })
      }

      await getRewards()
      await getClaimableLordsBalance()
      await getEpoch()
      await getTimeToNextEpoch()
      await getTotalRealmsStaked()
      // await isLordsAdded()
    })
    const stakeRealms = () => {
      showComponent(JourneySettling, {
        type: 'stake'
      })
    }
    const unstakeRealms = () => {
      showComponent(JourneySettling, {
        type: 'unstake'
      })
    }
    const numberRealms = computed(() => {
      return userRealms.value.l1?.wallet?.realmsHeld
    })
    const bridgedRealms = computed(() => {
      return userRealms.value.l1?.wallet?.bridgedRealmsHeld
    })
    watch(
      account,
      async (val) => {
        if (val) {
          await getRewards()
          await fetchUserPositions()
          await getClaimableLordsBalance()
          await getWalletRealms({ address: account.value, first: 1000 })
          await getTotalRealmsStaked()
          if (networkMismatch.value) {
            checkForNetworkMismatch()
          }
        }
      },
      {
        immediate: true
      }
    )
    const faqs = [
      {
        title: 'What does this staking contract do?',
        body: 'By boarding the ship now, you can begin to earn $LORDS. Once the StarkNet bridge is complete, your Realms will be available to claim directly on StarkNet.'
      },
      {
        title: 'How long do I need to be staked for to be entitled?',
        body: 'For each Realm staked per epoch, you will be rewarded with 625x $LORDS.'
      },
      {
        title:
          'How long do you need to be staked for to be entitled to rewards?',
        body: 'You need to be staked for one entire epoch (7 days) to be entitled to your $LORDS. For example, if you stake at the end of epoch 2 and remain staked until epoch 4, you will receive the epoch 3 reward. '
      },
      {
        title: 'Once staked, do you need to stake again for future epochs?',
        body: 'No, once you stake and remained staked, you’ll be entitled to all future epoch rewards (unless you unstake). '
      },
      {
        title: 'Why can’t I see my Realm(s) in OpenSea after staking?',
        body: 'Your realms won’t appear in OS while staking as they are transferred into the contract wallet in this process. You can view your staked Realms at your Adventurers page. '
      },
      {
        title: 'Can I list/sell my Realms on OpenSea while staking?',
        body: 'No, during staking you can’t list/sell your Realms as they are in the contract wallet. If you wish to list/sell your Realms then you must first unstake them.'
      },
      {
        title: 'How long will the Journey last for?',
        body: 'Until the StarkNet bridge is complete, the Journey will happen for a minimum 10 weeks. You can withdraw when your balance is above 0.'
      },
      {
        title: 'Do you have to claim $LORDS after each epoch?',
        body: 'No, you can let rewards accrue and claim in bulk to save gas. If you wish to avoid claiming on L1 we will have a solution to migrate your $LORDS to StarkNet and claim there to save further gas.'
      },
      {
        title: 'What are Realms?',
        body: 'Realms are mythical maps of the Lootverse. Every realm has been procedurally generated and is unique down to the language. Each realm has a map showing the regions, cities, rivers and topography that exist in the world. Resource deposits and exclusive Wonders can be found in each realm with varying rarity, and each belongs to one of the 16 Loot Orders.'
      },
      {
        title: 'The $LORDS Token',
        body: 'The $LORDS token is the utility token of the Realmsverse, used to transact on marketplaces on StarkNet. There will be a native StarkNet marketplace for trading Realms along with an AMM (Uniswap style) for trading the resources generated. Both marketplaces will be denominated in $LORDS. Read the full tokenomics breakdown here: <a class="font-semibold underline" href="https://docs.bibliothecaforloot.com/economics/lords">$LORDS Tokenomics</a>'
      },
      {
        title: 'Why is settling all on-chain?',
        body: 'All game activity will be on-chain, this means no central servers and no central backend. This allows the game assets to be composed and used in other games within the Realmverse, with no reliance on a central entity for their ongoing use.'
      },
      {
        title: 'What is StarkNet?',
        body: "StarkNet is a ZK-Rollup Layer 2 network. It allows for extreme computation and cheap transaction costs, without sacrificing the security of your assets. The Bibliotheca team is building one of the world's first fully on-chain games on a ZK-Rollup. You will be able to trade with speed and with little fees."
      },
      {
        title: 'Community Driven - The Bibliotheca DAO',
        body: 'Settling of the Realms is governed by the Realm Lords under the Bibliotheca DAO. Game direction, testing and feedback are all directed by the community. There is a core team of developers building the game, however all code is open source and we encourage outside contributors.'
      },
      {
        title: 'Can I audit your Contracts?',
        body: 'Yes! Find them here. <br> <a class="font-semibold underline" href="https://etherscan.io/address/0x686f2404e77ab0d9070a46cdfb0b7fecdd2318b0">$LORDS</a><br> <a class="font-semibold underline" href="https://etherscan.io/address/0x7afe30cb3e53dba6801aa0ea647a0ecea7cbe18d">Realms</a><br> <a class="font-semibold underline" href="https://etherscan.io/address/0x17963290db8c30552d0cfa2a6453ff20a28c31a2#code">Journey</a>'
      }
    ]

    return {
      faqs,
      account,
      poolIncentives,
      open,
      loading,
      epoch,
      userRealms,
      bridgedRealms,
      claimableBalance,
      numberRealms,
      fetch,
      stakeRealms,
      unstakeRealms,
      claimAllLords,
      timeLeft,
      getTotalRealmsStaked,
      totalRealmsStaked,
      rewardInfo,
      lpPositions,
      getLp,
      withdraw,
      deposit,
      getRewards,
      userRewards,
      claim,
      fetchUserPositions,
      userPositions,
      loadingIncentive,
      isLordsAdded,
      v1PositionsFilter,
      tab
    }
  }
})
</script>
