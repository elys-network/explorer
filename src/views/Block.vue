<template>
  <div>
    <b-table-simple v-if="isUpcomingblock()">
      <b-tr>
        <b-td class="text-center">
          <flip-countdown :deadline="upgradeTime"  />
          <b-input-group prepend="Estimated by block time: ">
            <b-form-select v-model="blocktime">
              <b-form-select-option value="7">
                7s
              </b-form-select-option>
              <b-form-select-option value="6">
                6s
              </b-form-select-option>
              <b-form-select-option value="5">
                5s
              </b-form-select-option>
              <b-form-select-option value="4">
                4s
              </b-form-select-option>
              <b-form-select-option value="2">
                2s
              </b-form-select-option>
              <b-form-select-option value="1">
                1s
              </b-form-select-option>
            </b-form-select></b-input-group>
        </b-td>
      </b-tr>
    </b-table-simple>
    <b-row v-if="isUpcomingblock()">
      <b-col
        lg="4"
        sm="6"
      >
        <dashboard-card-horizontal
          icon="HashIcon"
          color="danger"

          :statistic="height"
          statistic-title="Countdown For Block:"
        />
      </b-col>
      <b-col
        lg="4"
        sm="6"
      >
        <dashboard-card-horizontal
          icon="HashIcon"
          color="success"
          :statistic="currentBlock"
          statistic-title="Current Block:"
        />
      </b-col>
      <b-col
        lg="4"
        sm="6"
      >
        <dashboard-card-horizontal
          icon="HashIcon"
          :statistic="remainingBlocks"
          statistic-title="Remaining Blocks:"
        />
      </b-col>
    </b-row>
    <b-card v-if="!isUpcomingblock()">
      <b-card-title class="d-flex justify-content-between">
        <span>#{{ height }}</span>
        <span><b-button size="sm" @click="goblock(height - 1)"> &lt; </b-button> <b-button size="sm"
            @click="goblock(Number(height) + 1)"> > </b-button></span>
      </b-card-title>
      <object-field-component :tablefield="block.block_id" />
    </b-card>

    <b-card title="Block Header" v-if="!isUpcomingblock()">
      <object-field-component :tablefield="block.block.header" />
    </b-card>

    <b-card v-if="block.block.data.txs && !isUpcomingblock()" title="Transaction">
      <b-table :items="txs" :fields="fields" responsive="sm">
        <template #cell(hash)="data">
          <router-link :to="`../tx/${data.value}`">
            {{ data.value }}
          </router-link>
        </template>
      </b-table>
    </b-card>

    <b-card v-if="block.block.evidence.evidence && !isUpcomingblock()" title="Evidence">
      <array-field-component :tablefield="block.block.evidence.evidence" />
    </b-card>

    <b-card title="Last Commit" v-if="!isUpcomingblock()">
      <object-field-component :tablefield="block.block.last_commit" :small="true" />
    </b-card>
  </div>
</template>

<script>
import {
  BCard, BTable, BCardTitle, BButton, BInputGroup, BFormSelect, BFormSelectOption, BTableSimple, BTr, BTd, BRow, BCol,
} from 'bootstrap-vue'
import FlipCountdown from 'vue2-flip-countdown'
import { fromBase64 } from '@cosmjs/encoding'
import { decodeTxRaw } from '@cosmjs/proto-signing'
import Tx from '@/libs/data/tx'
import dayjs from 'dayjs'
import { abbrMessage, tokenFormatter } from '@/libs/utils'
import ObjectFieldComponent from './components/ObjectFieldComponent.vue'
import ArrayFieldComponent from './components/ArrayFieldComponent.vue'
import DashboardCardHorizontal from './components/dashboard/DashboardCardHorizontal.vue'

export default {
  components: {
    BButton,
    BCard,
    BTable,
    BCardTitle,
    BInputGroup,
    BFormSelect,
    BFormSelectOption,
    BTableSimple,
    BTr,
    BTd,
    BRow,
    BCol,
    FlipCountdown,
    ObjectFieldComponent,
    ArrayFieldComponent,
    DashboardCardHorizontal,
  },
  beforeRouteUpdate(to, from, next) {
    const { height } = to.params
    if (height > 0 && height !== from.params.height) {
      this.initData(height)
      next()
    }
  },
  data() {
    return {
      block: { block: { header: {}, data: {}, evidence: {} } },
      txs: null,
      height: 0,
      blocktime: 6,
      latest: {},
      currentBlock: 0,
      remainingBlocks: 0,
      fields: [
        { key: 'hash' },
        { key: 'fee', formatter: v => tokenFormatter(v) },
        { key: 'messages', formatter: v => abbrMessage(v) },
        { key: 'memo' },
      ],
      timer: '',
    }
  },
  computed: {
    upgradeTime() {
      if (Number(this.height || 0) > 0 && this.latest?.block) {
        const blocks = Number(this.height) - Number(this.latest.block?.header?.height || 0)
        if (blocks > 0) {
          const endtime = dayjs().add(blocks * this.blocktime, 'second').format('YYYY-MM-DD HH:mm:ss')
          return endtime
        }
      }
      return '0001-01-01 00:00:00'
    },
  },
  created() {
    const { height } = this.$route.params
    this.initData(height)

    this.$http.getLatestBlock().then(res => {
      this.latest = res
      if (Number(this.height || 0) > 0 && res?.block) {
        this.currentBlock = Number(res.block?.header?.height || 0)
        this.remainingBlocks = Number(this.height) - this.currentBlock
      }
    })
    this.timer = setInterval(this.updateBlockHeights, 2000)
  },
  methods: {
    initData(height) {
      this.height = height
      this.$http.getBlockByHeight(height).then(res => {
        this.block = res
        const { txs } = res.block.data
        if (txs === null) return
        const array = []
        for (let i = 0; i < txs.length; i += 1) {
          let tx = new Tx()
          try {
            const origin = decodeTxRaw(fromBase64(txs[i]))
            tx = Tx.create(origin)
          } catch (e) {
            // catch errors
          }
          tx.setHash(txs[i])
          array.push(tx)
        }
        if (array.length > 0) this.txs = array
      })
    },
    goblock(height) {
      this.$router.push({ name: 'block', params: { height } })
    },
    isUpcomingblock() {
      if (Number(this.height || 0) > 0 && this.latest?.block) {
        const blocks = Number(this.height) - Number(this.latest.block?.header?.height || 0)
        return blocks > 0
      }
      return false
    },
    updateBlockHeights() {
      if (!this.isUpcomingblock()) {
        return
      }
      this.$http.getLatestBlock().then(res => {
        if (Number(this.height || 0) > 0 && res?.block) {
          this.currentBlock = Number(res.block?.header?.height || 0)
          this.remainingBlocks = Number(this.height) - this.currentBlock
        }
      })
    },
    clearAutoUpdate() {
      clearInterval(this.timer)
    },
  },
  beforeUnmount() {
    this.cancelAutoUpdate()
  },
}
</script>
