<template>
  <div class="manage-assets">
    <NavBar showMenu="true" showBack="true" backPath="/wallet" backLabel="Overview">
      <span class="wallet_header"><strong>Asset View</strong></span>
    </NavBar>
    <div class="manage-assets_search form wrapper">
      <div class="input-group">
        <SearchIcon /><input type="text" autocomplete="off" class="form-control form-control-sm" v-model="search" @keyup="sortAssets" placeholder="Search for an Asset" />
      </div>
      <router-link to="/settings/manage-assets/custom-token">Add Custom Token</router-link>
      <div v-if="assets.length === 0" class="mt-3 d-flex">
        <div>
          <h4>Can't find this token</h4>
          <p class="manage-assets_customText">Add Custom ERC20 tokens.</p>
          <a target="_blank" href="https://liquality.io/blog/liquality-wallet-0-9-0-release-notes/#add-custom-tokens-to-your-liquality-wallet">Learn how</a>
        </div>
      </div>
       <div v-if="assets.length === 0" class="mt-5 d-flex">
        <div class="manage-assets_noneBottom">
          <h5 class="manage-assets_noneBottomText">INQUIRE</h5>
          <a target="_blank" href="https://forms.gle/nsHeZFGgT3y7hwKp6">Enable other tokens</a><br />
          <a href="mailto:info@liquality.io">Offer liquidity</a>
        </div>
      </div>
      </div>
          <div class="manage-assets_list">
      <div v-for="asset in assets" :key="asset" class="asset-item d-flex align-items-center">
        <img :src="getAssetIcon(asset)" class="asset-icon asset-item_icon" />
        <div class="asset-item_name flex-fill">{{getAssetName(asset)}} ({{asset}})
          <!-- <span v-if="asset in networkWalletBalances" class="asset-item_balance">{{getAssetBalance(asset)}} {{asset}}</span> -->
        </div>
        <div class="asset-item_toggle">
          <toggle-button :css-colors="true" :value="isAssetEnabled(asset)" @change="e => toggleAsset(asset, e.value)" />
        </div>
      </div>
    </div>
    <div v-if="search" class="wrapper manage-assets_bottomSection">
      <button class="btn btn-light btn-outline-primary btn-lg btn-block" @click="clearSearch">Done</button>
    </div>
  </div>
</template>

<script>
import { isEmpty } from 'lodash-es'
import { mapState, mapActions } from 'vuex'
import cryptoassets from '@/utils/cryptoassets'
// import { prettyBalance } from '@/utils/coinFormatter'
import { getAssetIcon } from '@/utils/asset'
import NavBar from '@/components/NavBar.vue'
import SearchIcon from '@/assets/icons/search.svg'

export default {
  components: {
    SearchIcon,
    NavBar
  },
  data () {
    return {
      search: '',
      assets: []
    }
  },
  computed: {
    ...mapState(['activeNetwork', 'activeWalletId', 'enabledAssets', 'balances']),
    networkAssets () {
      return this.enabledAssets[this.activeNetwork][this.activeWalletId]
    }
  },
  methods: {
    ...mapActions(['enableAssets', 'disableAssets']),
    getAssetIcon,
    getAssetName (asset) {
      return cryptoassets[asset]?.name || asset
    },
    // getAssetBalance (asset) {
    //   return prettyBalance(this.networkWalletBalances[asset], asset)
    // },
    isAssetEnabled (asset) {
      return this.networkAssets.includes(asset)
    },
    toggleAsset (asset, newValue) {
      const params = { network: this.activeNetwork, walletId: this.activeWalletId, assets: [asset] }
      newValue ? this.enableAssets(params) : this.disableAssets(params)
    },
    sortAssets () {
      const allAssets = Object.keys(cryptoassets)
      const assets = allAssets.sort((a, b) => this.isAssetEnabled(b) - this.isAssetEnabled(a))
      if (isEmpty(this.search)) {
        this.assets = assets
      } else {
        this.assets = assets.filter(
          asset => asset.toUpperCase().includes(
            this.search.toUpperCase()
          ) ||
        cryptoassets[asset]?.name.toLowerCase()
        .includes(this.search.toLowerCase())
        )
      }
    },
    clearSearch () {
      this.search = ''
      this.sortAssets()
    }
  },
  created () {
    this.sortAssets()
  },
  watch: {
    activeNetwork () {
      this.clearSearch()
    }
  }
}
</script>

<style lang="scss">
.manage-assets {
  display: flex;
  flex-direction: column;
  min-height: 0;

  &_customText {
    font-size: $font-size-lg;
  }

  &_bottomSection {
    width: 100%;
    position: absolute;
    bottom: 0;
  }

  &_search {
    border-bottom: 1px solid $hr-border-color;
    input {
      margin-right: 8px;
      padding-left: 20px;
    }

    a {
      padding-top: 20px;
    }

    .input-group {
      align-items: center;
      svg {
        position: absolute;
        left: 0;
        top: 5px;
      }
    }

    svg {
      width: 16px;
      margin-right: 8px;
    }
  }

  &_list {
    overflow-y: auto;
  }
}

.asset-item {
  width: 100%;
  border-bottom: 1px solid $hr-border-color;
  height: 60px;
  padding: 16px 20px;

  &_icon {
    margin-right: 8px;
  }

  &_toggle {
    text-align: right;
  }

  &_balance {
    display: block;
    font-size: $font-size-tiny;
    color: $text-muted;
  }
}
</style>
