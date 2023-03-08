<template>
  <div class="bar">
    <span class="bar-text">{{ label }}: {{ amount }}%</span>
    <a-progress :percent="amount" size="small" :status="'normal'" :showInfo="false" />
  </div>
</template>

<script>
export default {
  name: 'StatusBar',
  props: {
    label: {
      type: String,
      required: true
    },
    amount: {
      type: Number,
      required: true
    }
  }
}
</script>

<style>
.bar {
  display: flex;
  flex-direction: column;
  height: 30px;
}

.bar span {
  font-size: 11px;
  text-transform: none;
}

.bar-text {
  color: #1890ff;
}
</style>
