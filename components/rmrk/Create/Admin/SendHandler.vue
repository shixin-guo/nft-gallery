<template>
  <div>
    <AddressParser v-model="parsedAddresses" />
    <BasicSlider v-model="distribution" label="action.distributionCount" />

    <BasicSwitch v-model="random" label="action.random" />
  </div>
</template>

<script lang="ts">
import {
  ProcessFunction,
  sendFunction,
  SendType,
  shuffleFunction,
} from '@/components/accounts/utils'
import { Debounce } from 'vue-debounce-decorator'
import Connector from '@kodadot1/sub-api'
import { Component, Emit, Vue, Watch } from 'nuxt-property-decorator'

const components = {
  AddressParser: () => import('@/components/accounts/AddressParser.vue'),
  BasicSwitch: () => import('@/components/shared/form/BasicSwitch.vue'),
  BasicSlider: () => import('@/components/shared/form/BasicSlider.vue'),
}

@Component({ components })
export default class SendHandler extends Vue {
  protected parsedAddresses: string[] = []
  protected random = false
  protected distribution = 100
  protected seed: number[] = []

  public created(): void {
    setTimeout(this.fetchRandomSeed, 3000)
  }

  public async fetchRandomSeed(): Promise<void> {
    const { api } = Connector.getInstance()
    const random = await api.query.babe.randomness()
    this.seed = Array.from(random)
  }

  @Watch('parsedAddresses')
  protected onParsedAddressedChanged(parsedAddresses: string[]): void {
    if (parsedAddresses) {
      this.onInput()
    }
  }

  @Watch('random')
  protected onRandomChange(random: boolean, oldVal: boolean): void {
    if (random !== oldVal) {
      this.onInput()
    }
  }

  @Watch('distribution')
  protected onDistributionChanged(distribution: number, oldVal: number): void {
    if (distribution !== oldVal) {
      this.onInput()
    }
  }

  @Emit('input')
  @Debounce(500)
  protected onInput(): { isValidMeta: boolean; metaFunction: ProcessFunction } {
    const self: SendType = {
      parsedAddresses: this.parsedAddresses,
      random: this.random,
      distribution: this.distribution,
    }
    console.log('SendType change', self)

    return {
      isValidMeta: self.parsedAddresses.length > 0,
      metaFunction: sendFunction(
        self.parsedAddresses,
        self.distribution,
        self.random ? shuffleFunction(this.seed) : undefined
      ),
    }
  }
}
</script>
