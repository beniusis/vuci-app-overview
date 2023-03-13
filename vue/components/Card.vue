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
    :hoverable="true"
    draggable
    @dragstart="drag($event, title)"
    @drop="drop($event, title)"
    @dragover.prevent
    @dragenter.prevent
  >
  <component v-if="head" slot="extra" :is="head.component" v-bind="head.props" />
  <div class="body">
    <div v-for="section in sections" :key="section.id" class="section">
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
  methods: {
    drag (event, title) {
      event.dataTransfer.setData('title', title)
    },

    drop (event, title) {
      const dropped = event.dataTransfer.getData('title')
      this.$emit('dragged', { dragged: dropped, after: title })
    }
  }
}
</script>

<style>
.card {
  width: 270px;
  height: 320px;
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
