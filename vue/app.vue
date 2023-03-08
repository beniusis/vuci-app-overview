<template>
  <div>
    <div class="cards">
      <Card v-for="card in cards" :key="card.title" v-bind="card" />
    </div>
    <Drawer
      :visible="settingsVisible"
      @open="openSettings"
      @close="closeSettings"
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
      settingsVisible: false
    }
  },
  methods: {
    getEventsData (table, eventsCount) {
      return this.$log.get({ table: table, limit: eventsCount })
    },

    async renderSystemCard () {
      this.updateCpuUsage()
      const systemData = await this.$system.getInfo()
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
            title: 'Router uptime',
            data: '%t'.format(systemData.uptime)
          },
          {
            title: 'Local device time',
            data: new Date(systemData.localtime * 1000).toLocaleString('lt-LT')
          },
          {
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
            title: 'Firmware version',
            data: systemData.release.revision
          }
        ]
      }
      this.cards.push(systemCard)
    },

    updateCpuUsage () {
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

    async renderNetworkEventsCard () {
      const networkEventsData = await this.getEventsData('NETWORK', this.eventsLimit)
      console.log(networkEventsData)
    },

    openSettings () {
      this.settingsVisible = true
    },

    closeSettings () {
      this.settingsVisible = false
    }
  },

  created () {
    this.renderSystemCard()
    this.renderNetworkEventsCard()
  }
}
</script>

<style>
.cards {
  display: flex;
  flex-direction: row;
  gap: 20px;
  margin: 20px;
}
</style>
