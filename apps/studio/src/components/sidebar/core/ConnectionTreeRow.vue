<template>
  <a
    class="connection-row"
    :class="{ active }"
    @click.prevent="$emit('select', connection)"
    :title="connection.name"
  >
    <i class="material-icons connection-icon">link</i>
    <span class="connection-name truncate">{{ connection.name }}</span>
    <span class="active-dot" v-if="active" />
    <span class="connection-type">{{ connection.connectionType }}</span>
  </a>
</template>

<script>
export default {
  name: 'ConnectionTreeRow',
  props: {
    connection: { type: Object, required: true },
    active: { type: Boolean, default: false },
  },
}
</script>

<style lang="scss" scoped>
.connection-row {
  display: flex;
  align-items: center;
  gap: 0.45rem;
  padding: 0.35rem 0.6rem;
  cursor: pointer;
  font-size: 0.85rem;
  user-select: none;
  border-left: 2px solid transparent;

  &:hover {
    background: rgba(127, 127, 127, 0.08);
  }

  &.active {
    background: rgba(127, 127, 127, 0.18);
    border-left-color: currentColor;
    font-weight: 500;
  }
}

.active-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: #4caf50;
  flex-shrink: 0;
}

.connection-icon {
  font-size: 16px;
  opacity: 0.7;
  flex-shrink: 0;
}

.connection-name {
  flex: 1;
  min-width: 0;
}

.connection-type {
  font-size: 0.65rem;
  padding: 1px 6px;
  border-radius: 3px;
  background: rgba(127, 127, 127, 0.22);
  text-transform: lowercase;
  flex-shrink: 0;
  opacity: 0.85;
}
</style>
