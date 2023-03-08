<template>
  <a-card
    size="small"
    :title="title"
    class="card"
    :headStyle="{
      textTransform: 'uppercase',
      fontWeight: 'bold',
      height: '50px',
      margin: '0 12px',
      padding: '0',
      color: 'rgba(0, 0, 0, 0.7)',
      borderColor: 'rgba(0, 0, 0, 0.5)'
    }"
    :bodyStyle="{
      color: 'rgba(0, 0, 0, 0.7)'
    }"
  >
  <component v-if="head" slot="extra" :is="head.component" v-bind="head.props" />
  <div class="body">
    <div v-for="section in sections" :key="section.title" class="section">
      <div class="title">{{ section.title }}</div>
      <div v-if="section.components" class="section-components">
        <component v-for="component in section.components"
          :is="component.name"
          :key="component.props.label"
          v-bind="component.props"
        />
      </div>
      <span v-else>{{ section.data }}</span>
    </div>
  </div>
  </a-card>
</template>

<script>
import StatusBar from './StatusBar.vue'

export default {
  name: 'Card',
  components: {
    StatusBar
  },
  props: {
    title: {
      type: String,
      default: null
    },
    head: {
      type: Object,
      default: null
    },
    sections: {
      type: Array,
      default: null
    }
  },
  data () {
    return {
    }
  },
  methods: {
  }
}
</script>

<style>
.card {
  width: 300px;
  height: 280px;
  border-radius: 5px;
}

.body span {
  font-size: 11px;
}

.section {
  padding: 5px 0 5px;
  border-bottom: 1px solid rgba(0, 0, 0, 0.1);
}

.section:nth-child(1) {
  padding-top: 0px;
}

.section:nth-last-child(1) {
  border: none;
}

.section-components {
  display: flex;
  flex-direction: row;
  gap: 10px;
}

.title {
  font-weight: bold;
  text-transform: uppercase;
}
</style>
