<template>
  <div class="view-container">
    <div class="send" v-if="currentStep === 'inputs'">
      <NavBar
        showBack="true"
        :backPath="routeSource === 'assets' ? '/wallet' : `/accounts/${account.id}/${asset}`"
        :backLabel="routeSource === 'assets' ? 'Overview' : asset"
      >
        Send
      </NavBar>
      <div class="wrapper form">
        <div class="wrapper_top">
           <SendInput
            :asset="asset"
            :amount="amount"
            :account="account"
            :amount-fiat="amountFiat"
            @update:amount="(newAmount) => (amount = newAmount)"
            @toggle-max="toggleMaxAmount"
            @update:amountFiat="(amount) => (amountFiat = amount)"
            :max="available"
            :available="available"
            :max-fiat="prettyFiatBalance(available, fiatRates[asset])"
            :amount-error="amountError"
            :max-active="maxOptionActive"
          />
          <div class="form-group mt-40">
            <label for="address">Send to</label>
            <div class="input-group">
              <input
                type="text"
                :class="{ 'is-invalid': address && addressError }"
                v-model="address"
                class="form-control form-control-sm"
                id="address"
                placeholder="Address"
                autocomplete="off"
                required
              />
            </div>
            <small
              v-if="address && addressError"
              class="text-danger form-text text-right"
              >{{ addressError }}</small
            >
          </div>
          <div class="form-group mt-150">
          <DetailsContainer v-if="feesAvailable">
            <template v-slot:header>
              <span class="details-title">Network Speed/Fee</span>
              <span class="text-muted">
                ({{ selectedFeeLabel }} / {{ prettyFee }} {{ feeType }})
              </span>
            </template>
            <template v-slot:content>
              <ul class="selectors">
                <li>
                  <div class="send_fees">
                    <span class="selectors-asset">{{ assetChain }}</span>
                    <div class="custom-fees" v-if="customFee">
                    {{ currentFee }} {{ assetChain }} / {{ totalFeeInFiat }} USD
                    <button class="btn btn-link" @click="resetCustomFee">
                      Reset
                    </button>
                  </div>
                    <FeeSelector
                      v-else
                      :asset="asset"
                      v-model="selectedFee"
                      :fees="assetFees"
                      :totalFees="maxOptionActive ? maxSendFees : sendFees"
                      :fiatRates="fiatRates"
                      @custom-selected="onCustomFeeSelected"
                    />
                  </div>
                </li>
              </ul>
            </template>
          </DetailsContainer>
        </div>
        </div>
        <div class="wrapper_bottom">
          <div class="button-group">
            <router-link :to="routeSource === 'assets' ? '/wallet' : `/accounts/${this.account.id}/${asset}`">
              <button class="btn btn-light btn-outline-primary btn-lg">
                Cancel
              </button>
            </router-link>
            <button
              class="btn btn-primary btn-lg"
              @click="currentStep = null"
              :disabled="!canSend"
            >
              Review
            </button>
          </div>
        </div>
      </div>
    </div>
    <div class="send" v-else-if="currentStep === 'custom-fees'">
      <CustomFees
        @apply="applyCustomFee"
        @update="setCustomFee"
        @cancel="cancelCustomFee"
        :asset="assetChain"
        :selected-fee="selectedFee"
        :fees="assetFees"
        :totalFees="maxOptionActive ? maxSendFees : sendFees"
        :fiatRates="fiatRates"
      />
    </div>
    <div class="send" v-else>
      <NavBar
        :showBackButton="true"
        :backClick="back"
        backLabel="Back"
      >
        Send
      </NavBar>
      <div class="send-confirm wrapper form">
      <div class="wrapper_top form">
        <div>
          <label>
            Send
          </label>
          <div class="d-flex align-items-center justify-content-between mt-0">
            <div class="confirm-value" :style="getAssetColorStyle(asset)">
            {{ dpUI(amount) }} {{ asset }}
          </div>
          <div class="details-text">${{ amountInFiat }}</div>
          </div>
        </div>
        <div class="detail-group">
          <label class="text-muted">
            Network Fee
          </label>
          <div class="d-flex align-items-center justify-content-between mt-0">
            <div>
            ~{{ prettyFee }} {{ feeType }}
          </div>
          <div class="details-text">${{ totalFeeInFiat }}</div>
          </div>
        </div>
        <div class="detail-group">
          <label class="text-muted">
            Amount + Fees
          </label>
          <div class="d-flex align-items-center justify-content-between mt-0">
            <div class="font-weight-bold" v-if="asset === feeType">
              {{ dpUI(amountWithFee) }} {{ asset }}
            </div>
             <div class="font-weight-bold" v-else>
              {{ dpUI(amount) }} {{ asset }} + {{ prettyFee }} {{ feeType }}
            </div>
          <div class="font-weight-bold">${{ totalToSendInFiat }}</div>
          </div>
        </div>
        <div class="mt-40">
          <label>Send To</label>
          <p class="confirm-address">{{ this.address ? shortenAddress(this.address) : '' }}</p>
        </div>
      </div>
      <div class="wrapper_bottom">
        <div class="button-group">
          <button
            class="btn btn-light btn-outline-primary btn-lg"
            v-if="!loading"
            @click="currentStep = 'inputs'"
          >
            Edit
          </button>
          <button
            class="btn btn-primary btn-lg btn-icon"
            @click="send"
            :disabled="loading"
          >
            <SpinnerIcon class="btn-loading" v-if="loading" />
            <template v-else>Send {{ asset }}</template>
          </button>
        </div>
      </div>
    </div>
    </div>
     <!-- Modals for ledger prompts -->
    <OperationErrorModal :open="sendErrorModalOpen"
                         :account="account"
                         @close="closeSendErrorModal"
                         :error="sendErrorMessage" />
    <LedgerSignRequestModal :open="signRequestModalOpen"
                            @close="closeSignRequestModal" />
  </div>
</template>

<script>
import { mapState, mapActions, mapGetters } from 'vuex'
import _ from 'lodash'
import BN from 'bignumber.js'
import cryptoassets from '@/utils/cryptoassets'
import { chains, currencyToUnit, unitToCurrency } from '@liquality/cryptoassets'
import NavBar from '@/components/NavBar'
import FeeSelector from '@/components/FeeSelector'
import { prettyBalance, prettyFiatBalance, dpUI, fiatToCrypto } from '@/utils/coinFormatter'
import {
  getNativeAsset,
  getAssetColorStyle,
  getAssetIcon
} from '@/utils/asset'
import { shortenAddress } from '@/utils/address'
import {
  TX_TYPES,
  FEE_TYPES,
  getTxFee,
  getFeeLabel
} from '@/utils/fees'
import SpinnerIcon from '@/assets/icons/spinner.svg'
import DetailsContainer from '@/components/DetailsContainer'
import SendInput from './SendInput'
import LedgerSignRequestModal from '@/components/LedgerSignRequestModal'
import OperationErrorModal from '@/components/OperationErrorModal'
import CustomFees from '@/components/CustomFees'

export default {
  components: {
    NavBar,
    FeeSelector,
    SpinnerIcon,
    DetailsContainer,
    SendInput,
    OperationErrorModal,
    LedgerSignRequestModal,
    CustomFees
  },
  data () {
    return {
      sendFees: {},
      maxSendFees: {},
      stateAmount: 0,
      stateAmountFiat: 0,
      address: null,
      selectedFee: 'average',
      currentStep: 'inputs',
      loading: false,
      maxOptionActive: false,
      sendErrorModalOpen: false,
      signRequestModalOpen: false,
      sendErrorMessage: '',
      customFeeAssetSelected: null,
      customFee: null
    }
  },
  props: {
    asset: String,
    accountId: String
  },
  computed: {
    ...mapState([
      'activeNetwork',
      'activeWalletId',
      'fees',
      'fiatRates'
    ]),
    ...mapGetters([
      'accountItem',
      'client'
    ]),
    account () {
      return this.accountItem(this.accountId)
    },
    amount: {
      get () {
        return this.stateAmount
      },
      set (newValue) {
        if (newValue && !isNaN(newValue)) {
          this.stateAmount = newValue
        } else {
          this.stateAmount = 0.0
        }
        this.stateAmountFiat = prettyFiatBalance(
          this.stateAmount,
          this.fiatRates[this.asset]
        )
      }
    },
    amountFiat: {
      get () {
        return `$${this.stateAmountFiat}`
      },
      set (newValue) {
        const value = (newValue || '0').replace('$', '')
        this.stateAmountFiat = value
        this.stateAmount = fiatToCrypto(value, this.fiatRates[this.asset])
      }
    },
    balance () {
      return this.account.balances[this.asset] || 0
    },
    routeSource () {
      return this.$route.query.source || null
    },
    assetChain () {
      return getNativeAsset(this.asset)
    },
    assetFees () {
      const assetFees = {}
      if (this.customFee) {
        assetFees.custom = { fee: this.customFee }
      }

      const fees = this.fees[this.activeNetwork]?.[this.activeWalletId]?.[
        this.assetChain
      ]
      if (fees) {
        Object.assign(assetFees, fees)
      }

      return assetFees
    },
    feesAvailable () {
      return this.assetFees && Object.keys(this.assetFees).length
    },
    currentFee () {
      const fees = this.maxOptionActive ? this.maxSendFees : this.sendFees
      return (this.selectedFee in fees) ? fees[this.selectedFee] : BN(0)
    },
    isValidAddress () {
      return chains[cryptoassets[this.asset].chain].isValidAddress(this.address)
    },
    addressError () {
      if (!this.isValidAddress) {
        return 'Wrong format. Please check the address.'
      }
      return null
    },
    amountError () {
      const amount = BN(this.amount)
      if (amount.gt(this.available)) { return 'Lower amount. This exceeds available balance.' }
      return null
    },
    canSend () {
      if (!this.address || this.addressError) return false
      if (BN(this.amount).lte(0) || this.amountError) return false

      return true
    },
    prettyFee () {
      return this.currentFee.dp(6)
    },
    available () {
      if (cryptoassets[this.asset].type === 'erc20') {
        return unitToCurrency(cryptoassets[this.asset], this.balance)
      } else {
        const maxSendFee = (this.selectedFee in this.maxSendFees) ? this.maxSendFees[this.selectedFee] : BN(0)
        const fee = currencyToUnit(cryptoassets[this.assetChain], maxSendFee)
        const available = BN.max(BN(this.balance).minus(fee), 0)
        return unitToCurrency(cryptoassets[this.asset], available)
      }
    },
    amountInFiat () {
      return prettyFiatBalance(this.amount, this.fiatRates[this.asset])
    },
    totalFeeInFiat () {
      return prettyFiatBalance(this.currentFee, this.fiatRates[this.asset])
    },
    feeType () {
      return FEE_TYPES[this.assetChain]
    },
    includeFees () {
      return this.feeType === FEE_TYPES.BTC
    },
    selectedFeeLabel () {
      return getFeeLabel(this.selectedFee)
    },
    totalToSendInFiat () {
      const total = BN(this.amount).plus(BN(this.currentFee))
      return prettyFiatBalance(total, this.fiatRates[this.asset])
    },
    amountWithFee () {
      return BN(this.amount).plus(BN(this.currentFee))
    }
  },
  methods: {
    ...mapActions(['updateFees', 'sendTransaction']),
    prettyBalance,
    dpUI,
    prettyFiatBalance,
    getAssetIcon,
    getAssetColorStyle,
    shortenAddress,
    async _updateSendFees (amount) {
      const getMax = amount === undefined
      if (this.feesAvailable) {
        const sendFees = {}
        for (const [speed, fee] of Object.entries(this.assetFees)) {
          const feePrice = fee.fee
          sendFees[speed] = getTxFee(this.assetChain, TX_TYPES.SEND, feePrice)
        }
        if (this.asset === 'BTC') {
          const client = this.client(this.activeNetwork, this.activeWalletId, this.asset)
          const feePerBytes = Object.values(this.assetFees).map(fee => fee.fee)
          const value = getMax ? undefined : currencyToUnit(cryptoassets[this.asset], BN(amount))
          try {
            const totalFees = await client.getMethod('getTotalFees')({ value, feePerBytes, max: getMax })
            for (const [speed, fee] of Object.entries(this.assetFees)) {
              const totalFee = unitToCurrency(cryptoassets[this.asset], totalFees[fee.fee])
              sendFees[speed] = totalFee
            }
          } catch (e) {
            console.error(e)
          }
        }

        if (getMax) {
          this.maxSendFees = sendFees
        } else {
          this.sendFees = sendFees
        }
      }
    },
    updateSendFees: _.debounce(async function (amount) {
      await this._updateSendFees(amount)
    }, 800),
    async updateMaxSendFees () {
      await this._updateSendFees()
    },
    async send () {
      this.sendErrorMessage = ''
      this.loading = true
      if (this.account?.type.includes('ledger')) {
        this.signRequestModalOpen = true
      }

      try {
        const amountToSend = this.maxOptionActive ? this.available : this.amount

        const amount = currencyToUnit(cryptoassets[this.asset], amountToSend).toNumber()

        // validate for custom fees
        const fee = this.feesAvailable
          ? this.assetFees[this.selectedFee].fee
          : undefined

        await this.sendTransaction({
          network: this.activeNetwork,
          walletId: this.activeWalletId,
          asset: this.asset,
          to: this.address,
          accountId: this.account.id,
          amount,
          fee
        })

        this.$router.replace(`/accounts/${this.accountId}/${this.asset}`)
      } catch (error) {
        console.error(error)
        const { message } = error
        this.loading = false
        this.signRequestModalOpen = false
        this.sendErrorMessage = message || error
        this.sendErrorModalOpen = true
      }
    },
    toggleMaxAmount () {
      this.maxOptionActive = !this.maxOptionActive
      if (this.maxOptionActive) {
        this.amount = BN.min(
          BN(this.available),
          dpUI(this.available)
        )
      }
    },
    back () {
      this.currentStep = 'inputs'
    },
    closeSendErrorModal () {
      this.sendErrorModalOpen = false
      this.loading = false
    },
    closeSignRequestModal () {
      this.signRequestModalOpen = false
      this.loading = false
    },
    cancelCustomFee () {
      this.currentStep = 'inputs'
      this.selectedFee = 'average'
    },
    setCustomFee: _.debounce(async function ({ fee }) {
      this.customFee = fee
      if (this.maxOptionActive) {
        this.updateMaxSendFees()
      } else {
        this.updateSendFees(this.amount)
      }
    }, 800),
    applyCustomFee ({ fee }) {
      const presetFee = Object.entries(this.assetFees).find(([speed, speedFee]) => speed !== 'custom' && speedFee.fee === fee)
      if (presetFee) {
        const [speed] = presetFee
        this.selectedFee = speed
        this.customFee = null
      } else {
        this.updateMaxSendFees()
        this.updateSendFees(this.amount)
        this.customFee = fee
        this.selectedFee = 'custom'
      }
      this.currentStep = 'inputs'
    },
    onCustomFeeSelected () {
      this.currentStep = 'custom-fees'
    },
    resetCustomFee () {
      this.customFee = null
      this.selectedFee = 'average'
    }
  },
  async created () {
    await this.updateFees({ asset: this.assetChain })
    await this.updateSendFees(0)
    await this.updateMaxSendFees()
  },
  watch: {
    selectedFee: {
      handler () {
        if (this.maxOptionActive) {
          this.amount = BN.min(
            BN(this.available),
            dpUI(this.available)
          )
        }
      },
      deep: true
    },
    amount: function (val) {
      const amount = BN(val)
      const available = dpUI(this.available)
      if (!amount.eq(available)) {
        this.maxOptionActive = false
        this.updateSendFees(this.amount)
      }
    },
    available: function () {
      if (this.maxOptionActive) {
        this.amount = dpUI(this.available)
      }
    }
  }
}
</script>

<style lang="scss">
.send {
  &_asset {
    &.input-group {
      align-items: center;
    }

    &_icon {
      margin-right: 4px;
    }

    input {
      text-align: right;
      margin-left: 12px;
    }
  }
  &_fees {
    display: flex;
    align-items: center;
    font-weight: bold;
    margin: 6px 0;
    .fee-selector {
      margin-left: 6px;
    }
    .custom-fees {
      font-weight: normal;
    }

  }
}

/* Chrome, Safari, Edge, Opera */
input::-webkit-outer-spin-button,
input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

/* Firefox */
input[type="number"] {
  -moz-appearance: textfield;
}
</style>
