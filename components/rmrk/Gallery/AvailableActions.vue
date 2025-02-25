<template>
  <div class="actions-wrap">
    <Loader v-model="isLoading" :status="status" />
    <div v-if="accountId" class="buttons">
      <ShareNetwork
        v-if="identity && identity.twitter && this.isOwner"
        tag="button"
        class="button is-info is-dark is-outlined is-fullwidth twitter-btn"
        network="twitter"
        :hashtags="'KodaDot'"
        :url="realworldFullPath"
        :title="labelTwitter">
        <b-icon pack="fab" icon="twitter" />
        <span class="joy">SHARE JOY</span>
      </ShareNetwork>
      <template v-if="isOwner">
        <b-button
          v-for="action in actions"
          :key="action"
          :type="iconType(action)[0]"
          outlined
          @click="handleAction(action)"
          expanded>
          {{ action }}
        </b-button>
      </template>
      <template v-else>
        <b-tooltip
          :active="buyDisabled"
          :label="$i18n.t('tooltip.buyDisabled')">
          <b-button
            :type="iconType('BUY')[0]"
            :disabled="buyDisabled || !isAvailableToBuy"
            outlined
            @click="handleAction('BUY')"
            expanded>
            {{ replaceBuyNowWithYolo ? 'YOLO' : 'BUY' }}
          </b-button>
        </b-tooltip>
      </template>
    </div>
    <BalanceInput
      v-show="selectedAction === 'LIST'"
      ref="balanceInput"
      class="mb-4"
      empty-on-error
      @input="updateMeta" />
    <AddressInput
      v-show="selectedAction === 'SEND'"
      ref="addressInput"
      class="mb-4"
      @input="updateMeta" />
    <b-button
      v-if="showSubmit"
      type="is-primary"
      icon-left="paper-plane"
      @click="submit">
      Submit {{ selectedAction }}
    </b-button>
  </div>
</template>

<script lang="ts">
import { Component, mixins, Prop, Ref, Watch } from 'nuxt-property-decorator'
import Connector from '@kodadot1/sub-api'
import exec, { execResultValue, txCb } from '@/utils/transactionExecutor'
import { notificationTypes, showNotification } from '@/utils/notification'
import { unpin } from '@/utils/proxy'
import { GenericAccountId } from '@polkadot/types/generic/AccountId'
import RmrkVersionMixin from '@/utils/mixins/rmrkVersionMixin'
import { somePercentFromTX } from '@/utils/support'
import shouldUpdate from '@/utils/shouldUpdate'
import nftById from '@/queries/nftById.graphql'
import PrefixMixin from '~/utils/mixins/prefixMixin'
import KeyboardEventsMixin from '~/utils/mixins/keyboardEventsMixin'
import { get } from 'idb-keyval'
import { identityStore } from '@/utils/idbStore'
import { emptyObject } from '~/utils/empty'

type Address = string | GenericAccountId | undefined
type IdentityFields = Record<string, string>

const ownerActions = ['SEND', 'CONSUME', 'LIST']
const buyActions = ['BUY']

type DescriptionTuple = [string, string] | [string]
const iconResolver: Record<string, DescriptionTuple> = {
  SEND: ['is-info is-dark'],
  CONSUME: ['is-danger'],
  LIST: ['is-light'],
  BUY: ['is-success is-dark'],
}

type Action = 'SEND' | 'CONSUME' | 'LIST' | 'BUY' | ''

const components = {
  BalanceInput: () => import('@/components/shared/BalanceInput.vue'),
  AddressInput: () => import('@/components/shared/AddressInput.vue'),
  Loader: () => import('@/components/shared/Loader.vue'),
}

@Component({ components })
export default class AvailableActions extends mixins(
  RmrkVersionMixin,
  PrefixMixin,
  KeyboardEventsMixin
) {
  @Prop() public currentOwnerId!: string
  @Prop() public accountId!: string
  @Prop() public originialOwner!: string
  @Prop() public price!: string
  @Prop() public nftId!: string
  @Prop({ default: () => [] }) public ipfsHashes!: string[]
  @Prop({ default: false }) public buyDisabled!: boolean
  private selectedAction: Action = ''
  private meta: string | number = ''
  protected isLoading = false
  protected status = ''
  protected label = ''
  private identity: IdentityFields = emptyObject<IdentityFields>()
  private ownerIdentity: IdentityFields = emptyObject<IdentityFields>()

  @Ref('balanceInput') readonly balanceInput
  @Ref('addressInput') readonly addressInput

  public created() {
    this.initKeyboardEventHandler({
      a: this.bindActionEvents,
    })
  }

  private bindActionEvents(event) {
    const mappings = {
      b: 'BUY',
      s: 'SEND',
      c: 'CONSUME',
      l: 'LIST',
    }

    event.preventDefault()

    this.handleAction(mappings[event.key])
  }

  get actions() {
    return this.isOwner ? ownerActions : this.isAvailableToBuy ? buyActions : []
  }

  get showSubmit() {
    return (
      this.selectedAction && (!this.showMeta || this.metaValid) && !this.isBuy
    )
  }

  get metaValid() {
    if (typeof this.meta === 'number') {
      return this.meta >= 0
    }

    return this.meta
  }

  get showMeta() {
    return ['SEND', 'LIST'].includes(this.selectedAction)
  }

  get replaceBuyNowWithYolo(): boolean {
    return this.$store.getters['preferences/getReplaceBuyNowWithYolo']
  }

  get labelTwitter() {
    return `I'm sharing the joy from my recent purchase ${
      this.ownerIdentity?.twitter
        ? `made by ${this.ownerIdentity?.twitter}`
        : ''
    }`
  }

  protected iconType(value: string) {
    return iconResolver[value]
  }

  protected handleAction(action: Action) {
    if (shouldUpdate(action, this.selectedAction)) {
      this.selectedAction = action
      switch (action) {
        case 'BUY':
          this.submit()
          break
        case 'LIST':
          this.balanceInput?.focusInput()
          break
        case 'SEND':
          this.addressInput?.focusInput()
          break
        default:
          break
      }
    } else {
      this.selectedAction = ''
      this.meta = ''
    }
  }

  public async identityOf(account: Address) {
    const address: string = this.resolveAddress(account)
    const identity = await get(address, identityStore)
    return identity
  }

  private resolveAddress(account: Address): string {
    return account instanceof GenericAccountId
      ? account.toString()
      : account || ''
  }

  get isActionEmpty() {
    return this.selectedAction === ''
  }

  get isOwner() {
    console.log(
      '{ currentOwnerId, accountId }',
      this.currentOwnerId,
      this.accountId
    )

    return (
      this.currentOwnerId &&
      this.accountId &&
      this.currentOwnerId === this.accountId
    )
  }

  get isAvailableToBuy() {
    const { price, accountId } = this
    return accountId && Number(price) > 0
  }

  private handleSelect(value: Action) {
    this.selectedAction = value
    this.meta = ''
  }

  private constructRmrk(): string {
    const { selectedAction, version, meta, nftId } = this
    return `RMRK::${selectedAction}::${version}::${nftId}${
      this.metaValid ? '::' + meta : ''
    }`
  }

  get isBuy() {
    return this.selectedAction === 'BUY'
  }

  get isConsume() {
    return this.selectedAction === 'CONSUME'
  }

  get isList() {
    return this.selectedAction === 'LIST'
  }

  get isSend() {
    return this.selectedAction === 'SEND'
  }

  get realworldFullPath() {
    return `${window.location.origin}${this.$route.fullPath}`
  }

  @Watch('originialOwner', { immediate: true })
  async watchOwnerAddress(newAddress: Address, oldAddress: Address) {
    if (shouldUpdate(newAddress, oldAddress)) {
      this.identityOf(newAddress).then((id) => (this.ownerIdentity = id))
    }
  }

  @Watch('accountId', { immediate: true })
  async watchAddress(newAddress: Address, oldAddress: Address) {
    if (shouldUpdate(newAddress, oldAddress)) {
      this.identityOf(newAddress).then((id) => (this.identity = id))
    }
  }

  protected updateMeta(value: string | number) {
    console.log(typeof value, value)
    this.meta = value
  }

  protected async checkBuyBeforeSubmit() {
    const nft = await this.$apollo.query({
      query: nftById,
      client: this.urlPrefix,
      variables: {
        id: this.nftId,
      },
    })

    const {
      data: { nFTEntity },
    } = nft

    if (
      nFTEntity.currentOwner !== this.currentOwnerId ||
      nFTEntity.burned ||
      nFTEntity.price === 0 ||
      nFTEntity.price !== this.price
    ) {
      showNotification(
        `[RMRK::${this.selectedAction}] Owner changed or NFT does not exist`,
        notificationTypes.warn
      )
      throw new ReferenceError('NFT has changed')
    }
  }

  protected async submit() {
    const { api } = Connector.getInstance()
    const rmrk = this.constructRmrk()
    this.isLoading = true

    try {
      if (this.isActionEmpty) {
        throw new ReferenceError('No action selected')
      }
      showNotification(rmrk)
      console.log('submit', rmrk)
      const isBuy = this.isBuy
      const cb = isBuy ? api.tx.utility.batchAll : api.tx.system.remark
      const arg = isBuy
        ? [
            api.tx.system.remark(rmrk),
            api.tx.balances.transfer(this.currentOwnerId, this.price),
            somePercentFromTX(this.price),
          ]
        : rmrk

      if (isBuy) {
        await this.checkBuyBeforeSubmit()
      }

      const tx = await exec(
        this.accountId,
        '',
        cb,
        [arg],
        txCb(
          async (blockHash) => {
            execResultValue(tx)
            showNotification(blockHash.toString(), notificationTypes.info)
            if (this.isConsume) {
              this.unpinNFT()
            }

            showNotification(
              `[${this.selectedAction}] ${this.nftId}`,
              notificationTypes.success
            )
            this.selectedAction = ''
            this.isLoading = false
          },
          (err) => {
            execResultValue(tx)
            showNotification(`[ERR] ${err.hash}`, notificationTypes.danger)
            this.selectedAction = ''
            this.isLoading = false
          },
          (res) => {
            if (res.status.isReady) {
              this.status = 'loader.casting'
              return
            }

            if (res.status.isInBlock) {
              this.status = 'loader.block'
              return
            }

            if (res.status.isFinalized) {
              this.status = 'loader.finalized'
              return
            }

            this.status = ''
          }
        )
      )
    } catch (e) {
      showNotification(`[ERR] ${e}`, notificationTypes.danger)
      console.error(e)
      this.isLoading = false
    }
  }

  protected unpinNFT() {
    this.ipfsHashes.forEach(async (hash) => {
      if (hash) {
        try {
          await unpin(hash)
        } catch (e) {
          console.warn(`[ACTIONS] Cannot Unpin ${hash} because: ${e}`)
        }
      }
    })
  }

  unlistNft() {
    // change the selected action to list and change meta value to 0
    this.selectedAction = 'LIST'
    this.meta = 0
    this.submit()
  }
}
</script>
<style scoped lang="scss">
.joy {
  font-size: 16px;
  margin-top: 2px;
}
.twitter-btn {
  border-color: #1c9cef !important;
  color: #1c9cef !important;
  &:hover {
    color: #fff !important;
  }
}
.actions-wrap {
  .buttons {
    .b-tooltip {
      width: 100%;
    }
  }
}
</style>
