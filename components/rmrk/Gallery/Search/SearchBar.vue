<template>
  <div class="card mb-3 mt-5">
    <div class="columns mb-0">
      <b-field class="column is-6 mb-0" :class="searchColumnClass">
        <b-button
          v-if="!hideFilter"
          icon-left="filter"
          aria-controls="sortAndFilter"
          type="is-primary"
          class="is-hidden-mobile mr-2"
          @click="isVisible = !isVisible" />
        <b-autocomplete
          v-if="!hideSearchInput"
          class="gallery-search"
          v-model="name"
          :data="searchSuggestion"
          placeholder="Search..."
          icon="search"
          open-on-focus
          clearable
          max-height="350px"
          dropdown-position="is-bottom-left"
          expanded
          @keydown.native.enter="searchResult"
          @keydown.native.up="moveUp"
          @keydown.native.down="moveDown"
          @keydown.native.delete="makeInputDirty"
          @typing="updateSuggestion"
          @select="updateSelected"
          @input="makeInputDirty">
          <template slot-scope="props">
            <div v-if="props.option.type === 'Search'">
              <div class="media">
                <div class="media-left">
                  <b-icon icon="search" size="is-medium" />
                </div>
                <div class="media-content">{{ props.option.name }}</div>
              </div>
            </div>

            <div v-else-if="props.option.type === 'History'">
              <div class="media">
                <div class="media-left">
                  <b-icon icon="history" size="is-medium" />
                </div>
                <div class="media-content">{{ props.option.name }}</div>
                <div
                  class="media-right"
                  @click.stop.prevent="removeSearchHistory(props.option.name)">
                  <b-button type="is-text" icon-left="times" />
                </div>
              </div>
            </div>

            <div v-else>
              <nuxt-link
                :to="{
                  name: 'rmrk-detail-id',
                  params: { id: props.option.id },
                }"
                tag="div">
                <div class="media">
                  <div class="media-left">
                    <BasicImage
                      customClass="is-32x32"
                      :src="props.option.image || '/placeholder.webp'" />
                    <!-- <div class="preview-media-wrapper">
                    <PreviewMediaResolver
                      v-if="!props.option.image && props.option.animation_url"
                      :src="props.option.animation_url"
                      :metadata="props.option.metadata"
                      :mimeType="props.option.type"
                    />
                    </div> -->
                  </div>
                  <div class="media-content">
                    {{ props.option.name }}
                  </div>
                </div>
              </nuxt-link>
            </div>
          </template>
        </b-autocomplete>
        <div v-if="searchQuery" class="mt-3 ml-4">
          <span>Showing results for {{ searchQuery }}</span>
        </div>
      </b-field>
      <b-field
        expanded
        position="is-right"
        class="column is-6"
        v-if="!hideFilter">
        <b-button
          icon-left="filter"
          aria-controls="sortAndFilter"
          type="is-primary"
          class="is-hidden-tablet mr-2"
          @click="isVisible = !isVisible" />
        <slot />
      </b-field>
    </div>
    <b-collapse
      aria-id="sortAndFilter"
      animation="opacitySlide"
      v-model="isVisible">
      <div class="columns mb-0">
        <Sort class="column is-4 mb-0" :value="sortBy" @input="updateSortBy" />
        <BasicSwitch
          class="is-flex column is-4"
          v-model="vListed"
          :label="!replaceBuyNowWithYolo ? 'sort.listed' : 'YOLO'"
          size="is-medium"
          labelColor="is-success" />
      </div>
      <b-slider
        v-if="listed"
        class="column is-half"
        v-model="rangeSlider"
        :custom-formatter="(val) => `${val} KSM`"
        :max="30"
        :min="0"
        :step="1"
        ticks
        @change="sliderChange">
      </b-slider>
      <span v-if="sliderDirty"
        >Prices ranging from {{ this.query.priceMin / 1000000000000 }} to
        {{ this.query.priceMax / 1000000000000 }}</span
      >
    </b-collapse>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue, Emit, mixins } from 'nuxt-property-decorator'
import { Debounce } from 'vue-debounce-decorator'
import { exist } from './exist'
import nftListWithSearch from '@/queries/nftListWithSearch.graphql'
import { SearchQuery } from './types'
import { denyList } from '@/utils/constants'
import { NFT, NFTMetadata } from '../../service/scheme'
import { getSanitizer } from '../../utils'
import shouldUpdate from '~/utils/shouldUpdate'
import PrefixMixin from '~/utils/mixins/prefixMixin'
import KeyboardEventsMixin from '~/utils/mixins/keyboardEventsMixin'
import { mapNFTorCollectionMetadata } from '~/utils/mappers'
import {
  getCloudflareImageLinks,
  processMetadata,
} from '~/utils/cachingStrategy'
import { fastExtract } from '~/utils/ipfs'

@Component({
  components: {
    Sort: () => import('./SearchSortDropdown.vue'),
    TypeTagInput: () => import('./TypeTagInput.vue'),
    Pagination: () => import('@/components/rmrk/Gallery/Pagination.vue'),
    BasicSwitch: () => import('@/components/shared/form/BasicSwitch.vue'),
    BasicImage: () => import('@/components/shared/view/BasicImage.vue'),
    // PreviewMediaResolver: () => import('@/components/rmrk/Media/PreviewMediaResolver.vue'), // TODO: need to fix CSS for model-viewer
  },
})
export default class SearchBar extends mixins(
  PrefixMixin,
  KeyboardEventsMixin
) {
  @Prop(String) public search!: string
  @Prop(String) public type!: string
  @Prop(String) public sortBy!: string
  @Prop(String) public searchColumnClass!: string
  @Prop(Boolean) public listed!: boolean
  @Prop(Boolean) public hideFilter!: boolean
  @Prop(Boolean) public hideSearchInput!: boolean

  protected isVisible = false
  private query: SearchQuery = {
    search: '',
    type: '',
    sortBy: 'BLOCK_NUMBER_DESC',
    listed: true,
  }

  private first = 30
  private currentValue = 1
  private result: NFT[] = []
  private searchString = ''
  private name = ''
  private searched: NFT[] = []
  private highlightPos = 0
  private rangeSlider = [0, 5]
  private inputDirty = false
  private sliderDirty = false

  public mounted(): void {
    this.getSearchHistory()
    exist(this.$route.query.search, this.updateSearch)
    exist(this.$route.query.type, this.updateType)
    exist(this.$route.query.sort, this.updateSortBy)
    exist(this.$route.query.listed, this.updateListed)
  }

  public created() {
    this.initKeyboardEventHandler({
      f: this.bindFilterEvents,
    })
  }

  private bindFilterEvents(event) {
    switch (event.key) {
      case 'b':
        this.updateListed(!this.vListed)
        break
      case 'n':
        this.updateSortBy('BLOCK_NUMBER_DESC')
        break
      case 'o':
        this.updateSortBy('BLOCK_NUMBER_ASC')
        break
      case 'e':
        this.updateSortBy('PRICE_DESC')
        break
      case 'c':
        this.updateSortBy('PRICE_ASC')
        break
    }
  }

  get vListed(): boolean {
    this.query.listed = this.listed
    return this.listed
  }

  set vListed(listed: boolean) {
    if (this.inputDirty) {
      this.searchResult()
    }
    this.updateListed(listed)
  }

  get searchQuery(): string {
    return this.search
  }

  set searchQuery(value: string) {
    this.updateSearch(value)
  }

  get offset() {
    return this.currentValue * this.first - this.first
  }

  get searchSuggestion() {
    const nameInSearch: boolean = this.oldSearchResult(this.name)

    const suggestions = this.result.length
      ? this.filterSearch().concat(this.result)
      : this.filterSearch()
    return nameInSearch || !this.name
      ? suggestions
      : ([{ type: 'Search', name: this.name } as unknown] as NFT[]).concat(
          suggestions
        )
  }

  get replaceBuyNowWithYolo(): boolean {
    return this.$store.getters['preferences/getReplaceBuyNowWithYolo']
  }

  @Emit('update:listed')
  @Debounce(50)
  updateListed(value: string | boolean): boolean {
    const v = String(value)
    this.replaceUrl(v, undefined, 'listed')
    return v === 'true'
  }

  insertNewHistory() {
    const newResult = {
      type: 'History',
      name: this.searchString,
    } as unknown as NFT
    this.searched.push(newResult)
    localStorage.kodaDotSearchResult = JSON.stringify(this.searched)
  }
  //Invoked when "enter" key is pressed
  @Debounce(50)
  searchResult() {
    this.inputDirty = false
    const offset = this.oldSearchResult(this.searchString) ? 0 : 1

    //When an item from the autocomplete list is highlighted
    if (this.highlightPos >= 0) {
      const searchCache = this.filterSearch()
      //Highlighted item is NFT or search result from cache
      if (this.highlightPos == 0 && offset) {
        this.insertNewHistory()
        this.updateSearch(this.searchString)
      } else if (this.highlightPos >= searchCache.length + offset) {
        this.updateSelected(
          this.result[this.highlightPos - searchCache.length - offset]
        )
      } else this.updateSearch(searchCache[this.highlightPos - offset].name)
    } else {
      //Searching empty string
      if (!this.searchString) return

      //Current search string is not present in cache
      if (offset) {
        this.insertNewHistory()
      }
      this.updateSearch(this.searchString)
    }
    // this.searchString = ''
  }

  @Emit('update:type')
  @Debounce(50)
  updateType(value: string): string {
    this.replaceUrl(value, undefined, 'type')
    return value
  }

  @Emit('update:sortBy')
  @Debounce(400)
  updateSortBy(value: string): string {
    if (this.inputDirty) {
      this.searchResult()
    }
    this.replaceUrl(value, undefined, 'sort')
    return value
  }

  @Debounce(50)
  updateSelected(value: any) {
    //To handle clearing event
    if (!value) return
    if (value.type == 'History') {
      this.updateSearch(value.name)
    } else if (value.type == 'Search') {
      this.insertNewHistory()
      this.updateSearch(value.name)
    } else {
      this.$router.push({ name: 'rmrk-detail-id', params: { id: value.id } })
    }
    // this.searchString = ''
  }

  @Emit('update:search')
  @Debounce(50)
  updateSearch(value: string): string {
    shouldUpdate(value, this.searchQuery) && this.replaceUrl(value)
    return value
  }

  moveUp() {
    this.highlightPos = Math.max(0, this.highlightPos - 1)
  }

  moveDown() {
    const l =
      this.result.length +
      this.filterSearch().length +
      (1 - Number(this.oldSearchResult(this.name))) -
      1
    this.highlightPos = Math.min(l, this.highlightPos + 1)
  }

  async updateSuggestion(value: string) {
    this.searchString = value
    //To handle empty string
    if (!value) return

    this.query.search = value
    this.highlightPos = -1

    try {
      const nft = this.$apollo.query({
        query: nftListWithSearch,
        client: this.urlPrefix,
        variables: {
          first: this.first,
          offset: this.offset,
          denyList,
          orderBy: this.query.sortBy,
          search: this.buildSearchParam(),
        },
      })
      const {
        data: {
          nFTEntities: { nodes: nfts },
        },
      } = await nft

      this.result = nfts

      const metadataList: string[] = nfts.map(mapNFTorCollectionMetadata)
      const imageLinks = await getCloudflareImageLinks(metadataList)

      await processMetadata<NFTMetadata>(metadataList, (meta, i) => {
        Vue.set(this.result, i, {
          ...this.result[i],
          ...meta,
          image:
            (this.result[i]?.metadata &&
              imageLinks[fastExtract(this.result[i].metadata)]) ||
            getSanitizer(meta.image || '')(meta.image || ''),
          animation_url: getSanitizer(meta.animation_url || '')(
            meta.animation_url || ''
          ),
        })
      })
    } catch (e: any) {
      console.warn('[PREFETCH] Unable fo fetch', this.offset, e.message)
    }
  }

  @Debounce(100)
  replaceUrl(value: string, value2?, key = 'search', key2?): void {
    this.$router
      .replace({
        path: '/rmrk/gallery',
        query: {
          ...this.$route.query,
          search: this.searchQuery,
          page: '1',
          [key]: value,
          [key2]: value2,
        },
      })
      .catch(console.warn /*Navigation Duplicate err fix later */)
    // if searchbar request or filter is set, pagination should always revert to page 1
    this.$emit('resetPage')
  }
  private buildSearchParam(): Record<string, unknown>[] {
    const params: any[] = []

    if (this.query.search) {
      params.push({
        name: { likeInsensitive: `%${this.query.search}%` },
      })
    }

    if (this.query.listed) {
      params.push({
        price: { greaterThan: '0' },
      })
    }

    return params
  }
  private getSearchHistory() {
    // localStorage.kodaDotSearchResult = ''
    const cacheResult = localStorage.kodaDotSearchResult
    if (cacheResult) {
      this.searched = JSON.parse(cacheResult)
    }
  }

  private filterSearch() {
    if (!this.searched.length) return []
    return this.searched.filter((option) => {
      return (
        option.name
          .toString()
          .toLowerCase()
          .indexOf((this.searchString || '').toLowerCase()) >= 0
      )
    })
  }

  private oldSearchResult(value: string): boolean {
    const res = this.searched.filter((r) => r.name === value)
    return !!res.length
  }

  private removeSearchHistory(value: string): void {
    this.searched = this.searched.filter((r) => r.name !== value)
    localStorage.kodaDotSearchResult = JSON.stringify(this.searched)
  }

  @Debounce(50)
  private sliderChange([min, max]: [number, number]): void {
    if (!this.sliderDirty) {
      this.sliderDirty = true
    }
    this.sliderChangeMin(min * 1000000000000)
    this.sliderChangeMax(max * 1000000000000)
    const priceMin = String(min)
    const priceMax = String(max)
    this.replaceUrl(priceMin, priceMax, 'min', 'max')
  }

  @Emit('update:priceMin')
  @Debounce(50)
  private sliderChangeMin(min: number): void {
    this.query.priceMin = min
  }

  @Emit('update:priceMax')
  @Debounce(50)
  private sliderChangeMax(max: number): void {
    this.query.priceMax = max
  }
  private makeInputDirty() {
    if (!this.inputDirty) {
      return (this.inputDirty = true)
    }
  }
}
</script>

<style scoped lang="scss">
@import '@/styles/variables';

.b-skeleton {
  height: 32px !important;
  width: 32px !important;
  position: absolute;
  top: 0;
  left: 0;
}

.media {
  align-items: center;
}
.is-32x32 {
  font-size: 0;
}
.is-text {
  &:hover,
  &:focus {
    background-color: $primary !important;
    transition: all 3s !important;
  }
}
</style>
