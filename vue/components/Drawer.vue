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
      <div class="sections">
        <div class="element">
          <span>System Card</span>
          <a-switch />
        </div>
        <div class="element">
          <span>LAN</span>
          <a-switch />
        </div>
        <div class="element">
          <span>WAN</span>
          <a-switch />
        </div>
        <div class="element">
          <span>Recent Network Events</span>
          <a-switch />
        </div>
        <div class="element">
          <span>Recent System Events</span>
          <a-switch />
        </div>
        <div class="element">
          <a-button type="primary" @click="$emit('close')">Submit</a-button>
        </div>
      </div>
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
    }
  }
}
</script>

<style>
.settings-button {
  position: fixed;
  right: 32.5px;
  top: 75px;
}

.sections {
  display: flex;
  flex-direction: column;
}

.element {
  display: flex;
  flex-direction: row;
  gap: 20px;
  justify-content: center;
  margin-top: 50px;
}
</style>
