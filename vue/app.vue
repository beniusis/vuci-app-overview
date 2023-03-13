<template>
  <div>
    <div class="cards">
      <Card
        v-for="card in positionedCards"
        :key="card.title"
        v-bind="card"
        @dragged="reorderCards"
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
      cardsVisibility: {},
      cardsOrder: {}
    }
  },
  computed: {
    positionedCards () {
      return [...this.cards].sort((a, b) => a.order - b.order).filter(c => c.visible === '1')
    }
  },
  timers: {
    renderSystemCard: { time: 1000, autostart: false, immediate: true, repeat: true }
  },
  methods: {
    getSystemData () {
      return this.$system.getInfo()
    },

    getInterfacesData () {
      return this.$network.load().then(() => this.$network.getInterfaces())
    },

    getEventsData (table, eventsCount) {
      return this.$log.get({ table: table, limit: eventsCount })
    },

    async getCardsVisibility () {
      await this.$uci.load('system_overview')
        .then(() => this.$uci.sections('system_overview').filter(s => s['.name'] === 'visibility'))
        .then(info => {
          this.cardsVisibility = {
            system: info[0].System,
            interfaces: [
              {
                name: 'lan',
                visibility: info[0].lan
              },
              {
                name: 'wan',
                visibility: info[0].wan
              }
            ],
            networkEvents: info[0].Recent_network_events,
            systemEvents: info[0].Recent_system_events
          }
        })
        .catch(err => this.$message.error(err.message))
    },

    async getCardsOrder () {
      await this.$uci.load('system_overview')
        .then(() => this.$uci.sections('system_overview').filter(s => s['.name'] === 'position'))
        .then(info => {
          this.cardsOrder = {
            system: info[0].System,
            interfaces: [
              {
                name: 'lan',
                order: info[0].lan
              },
              {
                name: 'wan',
                order: info[0].wan
              }
            ],
            networkEvents: info[0].Recent_network_events,
            systemEvents: info[0].Recent_system_events
          }
        })
    },

    async reorderCards ({ dragged, after }) {
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

    async renderSystemCard () {
      this.updateCpuUsage()
      const systemData = await this.getSystemData()
      const systemCard = {
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
        order: this.cardsOrder.system,
        visible: this.cardsVisibility.system
      }
      this.cards.push(systemCard)
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

    async renderInterfaceCards () {
      const interfacesData = await this.getInterfacesData()
      let interfaceCard = {}

      interfacesData.forEach((iface, idx) => {
        if (iface.name === 'lan' || iface.name === 'wan') {
          interfaceCard = {
            title: iface.name,
            sections: [
              {
                id: 0,
                title: 'Type',
                data: iface.status.device
              },
              {
                id: 1,
                title: 'IP Address',
                data: iface.getIPv4Addrs().join(', ')
              }
            ],
            order: this.cardsOrder.interfaces[idx].order,
            visible: this.cardsVisibility.interfaces[idx].visibility
          }
          this.cards.push(interfaceCard)
        }
      })
    },

    async renderNetworkEventsCard () {
      const networkEventsData = await this.getEventsData('NETWORK', this.eventsLimit)
      const networkEventsCard = {
        title: 'Recent network events',
        sections: Object.values(networkEventsData.log).map(event => {
          return {
            id: event.ID,
            title: new Date(event.TIME * 1000).toLocaleString('lt-LT'),
            data: event.TEXT
          }
        }),
        order: this.cardsOrder.networkEvents,
        visible: this.cardsVisibility.networkEvents
      }
      this.cards.push(networkEventsCard)
    },

    async renderSystemEventsCard () {
      const systemEventsData = await this.getEventsData('SYSTEM', this.eventsLimit)
      const systemEventsCard = {
        title: 'Recent system events',
        sections: Object.values(systemEventsData.log).map(event => {
          return {
            id: event.ID,
            title: new Date(event.TIME * 1000).toLocaleString('lt-LT'),
            data: event.TEXT
          }
        }),
        order: this.cardsOrder.systemEvents,
        visible: this.cardsVisibility.systemEvents
      }
      this.cards.push(systemEventsCard)
    },

    async renderCards () {
      await this.getCardsOrder()
        .then(() => this.getCardsVisibility())
        .then(() => this.renderSystemCard())
        .then(() => this.renderInterfaceCards())
        .then(() => this.renderNetworkEventsCard())
        .then(() => this.renderSystemEventsCard())
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

  created () {
    this.renderCards()
  }
}
</script>

<style>
.cards {
  display: flex;
  flex-direction: row;
  gap: 20px;
}
</style>
