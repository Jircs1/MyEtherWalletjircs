<template>
  <the-wrapper-dapp
    :is-new-header="true"
    :dapp-img="headerImg"
    :banner-text="header"
    :tab-items="tabs"
    :active-tab="activeTab"
    :valid-networks="validNetworks"
    top-strip
  >
  </the-wrapper-dapp>
</template>

<script>
import { mapActions, mapGetters, mapState } from 'vuex';
import BigNumber from 'bignumber.js';
import { fromWei } from 'web3-utils';

import { STAKEWISE_ROUTES } from './configsRoutes';
import { SUPPORTED_NETWORKS } from './handlers/helpers/supportedNetworks';
import { toBNSafe } from '@/core/helpers/numberFormatHelper';

import handler from './handlers/stakewiseHandler';

export default {
  name: 'TheStakewiseLayout',
  components: {
    TheWrapperDapp: () => import('@/dapps/TheWrapperDapp.vue')
  },
  data() {
    return {
      header: {
        title: 'Stakewise',
        subtext: 'Stake any amount of ETH and begin earning rewards.'
      },
      activeTab: 0,
      headerImg: require('@/assets/images/icons/dapps/icon-dapp-stakewise.svg'),
      validNetworks: SUPPORTED_NETWORKS,
      stakewiseHandler: {},
      fetchInterval: null
    };
  },
  computed: {
    ...mapState('wallet', ['web3', 'address']),
    ...mapGetters('global', ['isEthNetwork', 'network']),
    isSupported() {
      const isSupported = this.validNetworks.find(item => {
        return item.name === this.network.type.name;
      });
      return !!isSupported;
    },
    tabs() {
      const arr = [
        {
          name: 'Stake ETH',
          route: { name: STAKEWISE_ROUTES.CORE.NAME },
          id: 0
        }
      ];

      if (this.isEthNetwork) {
        arr.push({
          name: 'Compound Rewards',
          route: {
            name: STAKEWISE_ROUTES.REWARDS.NAME
          },
          id: 1
        });
      }
      return arr;
    }
  },
  watch: {
    $route(to) {
      if (to.name === STAKEWISE_ROUTES.REWARDS.NAME) {
        this.activeTab = this.tabs[1].id;
      } else {
        this.activeTab = this.tabs[0].id;
      }
    },
    web3() {
      clearInterval(this.fetchInterval);
      if (this.isSupported) {
        this.setup();
      }
    },
    network() {
      clearInterval(this.fetchInterval);
      if (this.isSupported) {
        this.setup();
      }
    }
  },
  mounted() {
    if (this.$route.name === STAKEWISE_ROUTES.REWARDS.NAME) {
      this.activeTab = this.tabs[1].id;
    }
    if (this.isSupported) {
      this.setup();
    }
  },
  beforeDestroy() {
    clearInterval(this.fetchInterval);
  },
  methods: {
    ...mapActions('stakewise', [
      'setPoolSupply',
      'setStakingFee',
      'setValidatorApr',
      'setRewardBalance',
      'setStakeBalance'
    ]),
    setup() {
      this.stakewiseHandler = new handler(this.web3, this.isEthNetwork);
      this.collectiveFetch();
      this.fetchInterval = setInterval(() => {
        this.collectiveFetch();
      }, 14000);
    },
    collectiveFetch() {
      Promise.all([
        this.stakewiseHandler.getEthPool(),
        this.stakewiseHandler.getStakingFee(),
        this.stakewiseHandler.getValidatorApr(),
        this.stakewiseHandler.getSethBalance(this.address),
        this.stakewiseHandler.getRethBalance(this.address)
      ]).then(res => {
        this.setPoolSupply(res[0]);
        this.setStakingFee(res[1]);
        this.setValidatorApr(
          BigNumber(res[2]).minus(BigNumber(res[2]).times(0.1)).dp(2).toString()
        );
        this.setStakeBalance(fromWei(toBNSafe(res[3])));
        this.setRewardBalance(fromWei(toBNSafe(res[4])));
      });
    }
  }
};
</script>
