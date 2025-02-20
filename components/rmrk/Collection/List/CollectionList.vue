<template>
  <div class="collections">
    <Loader :value="isLoading" />
    <Search
      v-bind.sync="searchQuery"
      @resetPage="currentValue = 1"
      :sortOption="collectionSortOption">
      <b-field>
        <Pagination
          hasMagicBtn
          simple
          replace
          preserveScroll
          :total="total"
          v-model="currentValue"
          :per-page="first" />
      </b-field>
    </Search>

    <div>
      <div class="columns is-multiline">
        <div
          class="column is-4 column-padding"
          v-for="collection in results"
          :key="collection.id">
          <div class="card collection-card">
            <nuxt-link
              :to="`/rmrk/collection/${collection.id}`"
              tag="div"
              class="collection-card__skeleton">
              <div class="card-image">
                <BasicImage
                  :src="collection.image"
                  :alt="collection.name"
                  customClass="collection__image-wrapper" />
              </div>

              <div class="card-content">
                <nuxt-link :to="`/rmrk/collection/${collection.id}`">
                  <CollectionDetail
                    :nfts="collection.nfts.nodes"
                    :name="collection.name" />
                </nuxt-link>
                <b-skeleton :active="isLoading"> </b-skeleton>
              </div>
            </nuxt-link>
          </div>
        </div>
      </div>
    </div>
    <Pagination
      class="pt-5 pb-5"
      :total="total"
      :perPage="first"
      v-model="currentValue"
      replace />
  </div>
</template>

<script lang="ts">
import { Component, mixins, Vue, Watch } from 'nuxt-property-decorator'
import shouldUpdate from '~/utils/shouldUpdate'
import {
  CollectionWithMeta,
  Collection,
  Metadata,
  NFTMetadata,
} from '@/components/rmrk/service/scheme'
import { getSanitizer } from '@/components/rmrk/utils'
import { SearchQuery } from '@/components/rmrk/Gallery/Search/types'
import 'lazysizes'

import collectionListWithSearch from '@/queries/collectionListWithSearch.graphql'
import PrefixMixin from '~/utils/mixins/prefixMixin'
import { mapOnlyMetadata } from '~/utils/mappers'
import {
  getCloudflareImageLinks,
  processMetadata,
} from '~/utils/cachingStrategy'
import { CollectionMetadata } from '~/components/rmrk/types'
import { fastExtract } from '~/utils/ipfs'

interface Image extends HTMLImageElement {
  ffInitialized: boolean
}

const components = {
  GalleryCardList: () =>
    import('@/components/rmrk/Gallery/GalleryCardList.vue'),
  Search: () =>
    import('@/components/rmrk/Gallery/Search/SearchBarCollection.vue'),
  Money: () => import('@/components/shared/format/Money.vue'),
  Pagination: () => import('@/components/rmrk/Gallery/Pagination.vue'),
  CollectionDetail: () =>
    import('@/components/rmrk/Gallery/CollectionDetail.vue'),
  Loader: () => import('@/components/shared/Loader.vue'),
  BasicImage: () => import('@/components/shared/view/BasicImage.vue'),
}

@Component<CollectionList>({
  components,
})
export default class CollectionList extends mixins(PrefixMixin) {
  private collections: Collection[] = []
  private meta: Metadata[] = []
  public first = this.$store.state.preferences.collectionsPerPage
  private placeholder = '/placeholder.webp'
  private currentValue = parseInt((this.$route.query?.page as string) || '1')
  private total = 0
  private searchQuery: SearchQuery = {
    search: '',
    type: '',
    sortBy: 'BLOCK_NUMBER_DESC',
    listed: false,
  }

  private collectionSortOption: string[] = [
    'BLOCK_NUMBER_DESC',
    'BLOCK_NUMBER_ASC',
    'UPDATED_AT_DESC',
    'UPDATED_AT_ASC',
  ]

  get isLoading(): boolean {
    return this.$apollo.queries.collection.loading
  }

  get offset(): number {
    return this.currentValue * this.first - this.first
  }

  private buildSearchParam(): Record<string, unknown>[] {
    const params: any[] = []

    if (this.searchQuery.search) {
      params.push({
        name: { likeInsensitive: `%${this.searchQuery.search}%` },
      })
    }

    return params
  }

  public async created() {
    this.$apollo.addSmartQuery('collection', {
      query: collectionListWithSearch,
      manual: true,
      client: this.urlPrefix,
      loadingKey: 'isLoading',
      result: this.handleResult,
      variables: () => {
        return {
          orderBy: this.searchQuery.sortBy,
          search: this.buildSearchParam(),
          listed: this.searchQuery.listed
            ? [{ price: { greaterThan: '0' } }]
            : [],
          first: this.first,
          offset: this.offset,
        }
      },
      update: ({ collectionEntity }) => ({
        ...collectionEntity,
        nfts: collectionEntity.nfts.nodes,
      }),
    })
  }

  protected async handleResult({ data }: any) {
    this.total = data.collectionEntities.totalCount
    this.collections = data.collectionEntities.nodes.map((e: any) => ({
      ...e,
    }))

    const metadataList: string[] = this.collections.map(mapOnlyMetadata)
    const imageLinks = await getCloudflareImageLinks(metadataList)

    processMetadata<CollectionMetadata>(metadataList, (meta, i) => {
      Vue.set(this.collections, i, {
        ...this.collections[i],
        ...meta,
        image:
          imageLinks[fastExtract(this.collections[i].metadata)] ||
          getSanitizer(meta.image || '')(meta.image || ''),
      })
    })

    this.prefetchPage(this.offset + this.first, this.offset + 3 * this.first)
  }

  public async prefetchPage(offset: number, prefetchLimit: number) {
    try {
      const collections = this.$apollo.query({
        query: collectionListWithSearch,
        client: this.urlPrefix,
        variables: {
          first: this.first,
          offset,
        },
      })

      const {
        data: {
          collectionEntities: { nodes: collectionList },
        },
      } = await collections

      const metadataList: string[] = collectionList.map(mapOnlyMetadata)
      processMetadata<NFTMetadata>(metadataList)
    } catch (e: any) {
      console.warn('[PREFETCH] Unable fo fetch', offset, e.message)
    } finally {
      if (offset <= prefetchLimit) {
        this.prefetchPage(offset + this.first, prefetchLimit)
      }
    }
  }

  @Watch('$route.query.search')
  protected onSearchChange(val: string, oldVal: string) {
    if (shouldUpdate(val, oldVal)) {
      this.currentValue = 1
      this.searchQuery.search = val || ''
    }
  }

  get results() {
    return this.collections as CollectionWithMeta[]
  }

  onError(e: Event) {
    const target = e.target as Image
    target.src = this.placeholder
  }
}
</script>

<style lang="scss">
@import '@/styles/variables';
.card-image__burned {
  filter: blur(7px);
}

.collections {
  &__image-wrapper {
    position: relative;
    margin: auto;
    padding-top: 100%;
    overflow: hidden;
    cursor: pointer;
  }

  .card-image img {
    border-radius: 0px;
    top: 50%;
    transition: all 0.3s;
    display: block;
    width: 100%;
    height: auto;
    transform: scale(1);
  }

  .has-text-overflow-ellipsis {
    overflow: hidden;
    white-space: nowrap;
    text-overflow: ellipsis;
    color: #fff;
  }

  .card-image__emotes__count {
    vertical-align: text-bottom;
  }

  .is-float-right {
    float: right;
  }

  .is-absolute {
    position: absolute;
  }

  .collection-collection-counter {
    top: 5px;
    right: -5px;
  }

  .columns {
    padding-top: 10px;

    .card {
      border-radius: 0px;
      position: relative;
      overflow: hidden;

      &-image {
        &__emotes {
          position: absolute;
          background-color: $primary-light;
          border-radius: 4px;
          padding: 3px 8px;
          color: #fff;
          top: 10px;
          right: 10px;
          font-size: 14px;
          z-index: 3;
          transition: all 0.3s;
        }

        &__price {
          position: absolute;
          background-color: $grey-darker;
          border-radius: 4px;
          padding: 3px 8px;
          color: #fff;
          bottom: 10px;
          left: 10px;
          font-size: 14px;
          z-index: 3;
          transition: all 0.3s;
        }
      }

      .card-content {
        .heading--inline {
          display: inline-block;
          overflow: hidden;
          white-space: nowrap;
          text-overflow: ellipsis;
          color: #fff;
        }

        .level-item {
          padding: 0.1rem;
          text-align: left;
          line-height: initial;
        }

        .collection-title {
          overflow: hidden;
          white-space: nowrap;
          text-overflow: ellipsis;
          color: #fff;
          font-size: 1.5rem;
        }
        .collection-title-class {
          max-width: 100%;
          text-align: center;
        }
      }

      @media screen and (min-width: 1024px) {
        &-content {
          position: absolute;
          bottom: -45px;
          left: 0;
          width: 100%;
          transition: all 0.3s;
          background: #fff;
          opacity: 0;
        }

        &:hover .card-content {
          background: $frosted-glass-background;
          backdrop-filter: $frosted-glass-backdrop-filter;
          bottom: 0;
          opacity: 1;
          z-index: 2;
          padding-left: 1rem;
          padding-right: 1rem;
        }

        &:hover .collection__image-wrapper img {
          transform: scale(1.1);
          transition: transform 0.3s linear;
        }

        &:hover .card-image__emotes {
          top: 15px;
          right: 15px;
        }

        &:hover .card-image__price {
          bottom: 62px;
        }
      }
    }
  }
}
</style>
