<template>
  <div>
    <div class="cards">
      <Card
        v-for="card in positionedCards"
        :key="card.title"
        v-bind="card"
        @dragged="repositionCards"
      />
    </div>
    <Drawer
      :visible="settingsVisible"
      :cards="cards"
      @open="openSettings"
      @close="closeSettings"
      @saved="saveSettings"
    />
  </div>
</template>

<script>
import Card from './components/Card.vue'
import Drawer from './components/Drawer.vue'

export default {
  components: {
    Card,
    Drawer
  },
  data () {
    return {
      cards: [],
      cpuPercentage: 0,
      eventsLimit: 4,
      lastCPUTime: null,
      settingsVisible: false,
      cardsVisibility: [],
      cardsOrder: []
    }
  },
  computed: {
    positionedCards () {
      return [...this.cards].sort((a, b) => a.order - b.order).filter(c => c.visibility === '1')
    }
  },
  timers: {
    updateSystemCard: { time: 1000, autostart: true, immediate: false, repeat: true },
    updateConfigFileData: { time: 10000, autostart: true, immediate: false, repeat: true }
  },
  methods: {
    async updateConfigFileData () {
      this.remove(await this.getVisibilitySectionInfo(), 'visibility')
      this.remove(await this.getPositionSectionInfo(), 'position')
    },

    remove (info, section) {
      Object.keys(info[0]).map((opt) => {
        if (!opt[0].startsWith('.')) {
          const index = this.cards.findIndex(card => card.title.replaceAll(' ', '_') === opt)
          if (index === -1) {
            this.$spin()
            this.$uci.load('system_overview')
              .then(() => this.$uci.del('system_overview', section, opt))
              .then(() => this.$uci.save())
              .then(() => this.$uci.apply())
              .catch(err => this.$message.error(err.message))
              .finally(() => this.$spin(false))
          }
        }
      })
    },

    getSystemData () {
      return this.$system.getInfo()
    },

    getInterfacesData () {
      return this.$network.load().then(() => this.$network.getInterfaces())
    },

    getEventsData (table, eventsCount) {
      return this.$log.get({ table: table, limit: eventsCount })
    },

    getLastPositionOfOrderedCards () {
      this.cardsOrder.sort((a, b) => a[1] - b[1])
      return Number(this.cardsOrder.at(-1)[1])
    },

    getVisibilitySectionInfo () {
      return this.$uci.load('system_overview')
        .then(() => this.$uci.sections('system_overview')
          .filter(s => s['.name'] === 'visibility'))
    },

    getPositionSectionInfo () {
      return this.$uci.load('system_overview')
        .then(() => this.$uci.sections('system_overview')
          .filter(s => s['.name'] === 'position'))
    },

    async getCardsVisibility () {
      const visibilityInfo = await this.getVisibilitySectionInfo()

      if (visibilityInfo.length < 1) return

      Object.entries(visibilityInfo[0]).map((opt) => {
        if (!opt[0].startsWith('.')) {
          this.cardsVisibility.push(opt)
        }
      })
    },

    async getCardsPositions () {
      const positionInfo = await this.getPositionSectionInfo()

      if (positionInfo.length < 1) return

      Object.entries(positionInfo[0]).map((opt) => {
        if (!opt[0].startsWith('.')) {
          this.cardsOrder.push(opt)
        }
      })
    },

    getCardOrder (name) {
      let order = 0
      const card = this.cardsOrder.filter(order => order[0] === name)
      if (card.length > 0) {
        order = card[0][1]
      } else {
        order = this.getLastPositionOfOrderedCards() + 1
      }
      return order
    },

    getCardVisibility (name) {
      let visibility = 0
      const card = this.cardsVisibility.filter(visible => visible[0] === name)
      if (card.length > 0) {
        visibility = card[0][1]
      }
      return visibility
    },

    async repositionCards ({ dragged, after }) {
      const afterCardIndex = this.cards.findIndex(card => card.title === after)
      const draggedCardIndex = this.cards.findIndex(card => card.title === dragged)

      const afterOrder = this.cards[draggedCardIndex].order
      const previousOrder = this.cards[afterCardIndex].order

      this.cards[afterCardIndex].order = afterOrder
      this.cards[draggedCardIndex].order = previousOrder

      this.$spin()
      await this.$uci.load('system_overview')
        .then(() => {
          this.$uci.set('system_overview', 'position', dragged.replaceAll(' ', '_'), previousOrder)
          this.$uci.set('system_overview', 'position', after.replaceAll(' ', '_'), afterOrder)
        })
        .then(() => this.$uci.save())
        .then(() => this.$uci.apply())
        .catch(err => this.$message.error(err.message))
        .finally(() => this.$spin(false))
    },

    updateCpuUsage () {
      // method from vuci-app-home application
      this.$rpc.call('system', 'cpu_time').then(times => {
        if (!this.lastCPUTime) {
          this.cpuPercentage = 0
          this.lastCPUTime = times
          return
        }

        const idle1 = this.lastCPUTime[3]
        const idle2 = times[3]

        let total1 = 0
        let total2 = 0

        this.lastCPUTime.forEach(t => {
          total1 += t
        })

        times.forEach(t => {
          total2 += t
        })

        this.cpuPercentage = Math.floor(((total2 - total1 - (idle2 - idle1)) / (total2 - total1)) * 100)
        this.lastCPUTime = times
      })
    },

    async generateSystemCardData () {
      this.updateCpuUsage()
      const systemData = await this.getSystemData()
      return {
        title: 'System',
        head: {
          component: 'StatusBar',
          props: {
            label: 'CPU load',
            amount: this.cpuPercentage
          }
        },
        sections: [
          {
            id: 0,
            title: 'Router uptime',
            data: '%t'.format(systemData.uptime)
          },
          {
            id: 1,
            title: 'Local device time',
            data: new Date(systemData.localtime * 1000).toLocaleString('lt-LT')
          },
          {
            id: 2,
            title: 'Memory usage',
            components: [
              {
                name: 'StatusBar',
                props: {
                  label: 'RAM',
                  amount: Math.floor(((systemData.memory.total - systemData.memory.free) / systemData.memory.total) * 100)
                }
              },
              {
                name: 'StatusBar',
                props: {
                  label: 'FLASH',
                  amount: Math.floor(((systemData.disk.root.total - systemData.disk.root.free) / systemData.disk.root.total) * 100)
                }
              }
            ]
          },
          {
            id: 3,
            title: 'Firmware version',
            data: systemData.release.revision
          }
        ],
        visibility: this.getCardVisibility('System'),
        order: this.getCardOrder('System')
      }
    },

    async renderSystemCard () {
      this.cards.push(await this.generateSystemCardData())
    },

    findCardIndex (title) {
      return this.cards.findIndex(card => card.title === title)
    },

    async updateSystemCard () {
      const index = this.findCardIndex('System')
      const updatedSystemCard = await this.generateSystemCardData()
      this.cards[index].head.props.amount = this.cpuPercentage
      this.cards[index].sections = updatedSystemCard.sections
    },

    async generateInterfaceCardData () {
      const interfaces = await this.getInterfacesData()
      return interfaces.map((iface) => {
        return {
          title: iface.name,
          sections: [
            {
              id: 0,
              title: 'Type',
              data: iface.status.l3_device || 'Interface is down'
            },
            {
              id: 1,
              title: 'IP Address',
              data: iface.getIPv4Addrs().join(', ') || 'Interface is down'
            }
          ],
          visibility: this.getCardVisibility(iface.name),
          order: this.getCardOrder(iface.name)
        }
      })
    },

    async renderInterfaceCard () {
      const interfacesData = await this.generateInterfaceCardData()
      interfacesData.forEach((iface) => {
        this.cards.push(iface)
      })
    },

    async generateEventsCardData (type) {
      const eventsData = await this.getEventsData(type, this.eventsLimit)
      return {
        title: `Recent ${type.toLowerCase()} events`,
        sections: Object.values(eventsData.log).map(event => {
          return {
            id: event.ID,
            title: new Date(event.TIME * 1000).toLocaleString('lt-LT'),
            data: event.TEXT
          }
        }),
        visibility: this.getCardVisibility(`Recent_${type.toLowerCase()}_events`),
        order: this.getCardOrder(`Recent_${type.toLowerCase()}_events`)
      }
    },

    async renderEventsCard () {
      this.cards.push(await this.generateEventsCardData('SYSTEM'))
      this.cards.push(await this.generateEventsCardData('NETWORK'))
    },

    async renderCards () {
      await this.getCardsVisibility()
        .then(() => this.getCardsPositions())
        .then(() => this.renderSystemCard())
        .then(() => this.renderInterfaceCard())
        .then(() => this.renderEventsCard())
    },

    openSettings () {
      this.settingsVisible = true
    },

    closeSettings () {
      this.settingsVisible = false
    },

    saveSettings () {
      this.settingsVisible = false
      setTimeout(() => {
        this.$router.go(0)
      }, 1000)
    }
  },

  async created () {
    await this.renderCards()
  }
}
</script>

<style>
.cards {
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
  justify-content: center;
  gap: 40px;
  margin: 20px;
  max-width: 1400px;
}
</style>
