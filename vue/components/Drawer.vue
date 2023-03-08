<template>
  <div>
    <a-button type="primary" class="settings-button" @click="$emit('open')">Settings</a-button>
    <a-drawer
      placement="right"
      :closable="false"
      :visible="visible"
      :width="400"
      @close="$emit('close')"
    >
      <vuci-form :uciConfig="config.name" @applied="$emit('saved')">
        <vuci-named-section :name="config.section" v-slot="{ s }">
          <vuci-form-item-switch
            v-for="card in cards"
            :uci-section="s"
            :key="card.title"
            :label="card.title"
            :name="card.title.replaceAll(' ', '_')"
            class="element"
          />
        </vuci-named-section>
      </vuci-form>
    </a-drawer>
  </div>
</template>

<script>
export default {
  name: 'Drawer',
  props: {
    visible: {
      type: Boolean,
      default: false
    },
    cards: {
      type: Array,
      default: null
    }
  },
  data () {
    return {
      config: {
        name: 'system_overview',
        section: 'visibility'
      }
    }
  }
}
</script>

<style>
.settings-button {
  position: fixed;
  right: 22.5px;
  top: 62.5px;
}

.element {
  display: flex;
  margin: 0;
}

.element .ant-form-item-label {
  width: 200px;
  font-weight: bold;
  text-align: right;
  text-transform: capitalize;
}
</style>
