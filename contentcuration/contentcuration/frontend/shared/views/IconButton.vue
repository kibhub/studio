<template>

  <VTooltip maxWidth="150px" bottom color="darkGrey">
    <template #activator="{ on }">
      <VBtn
        flat
        icon
        :small="small"
        v-bind="$attrs"
        v-on="on"
        @click.stop="$event => $emit('click', $event)"
        @keydown.enter="$emit('mousedown')"
        @keyup.enter="$emit('mouseup')"
        @mousedown="$emit('mousedown')"
        @mouseup="$emit('mouseup')"
        @mouseenter="$emit('mouseenter')"
        @mouseleave="$emit('mouseleave')"
      >
        <slot name="icon">
          <Icon :small="small">
            <slot>
              {{ icon }}
            </slot>
          </Icon>
        </slot>
      </VBtn>
    </template>
    <span class="text-xs-center">{{ text }}</span>
  </VTooltip>

</template>

<script>

  export default {
    name: 'IconButton',
    props: {
      text: {
        type: String,
        required: true,
      },
      icon: {
        type: String,
        required: true,
      },
      small: {
        type: Boolean,
        default: false,
      },
    },
  };

</script>
