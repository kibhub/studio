<template>

  <VList two-line>
    <!-- Select all checkbox -->
    <Checkbox
      v-model="selectAll"
      class="mt-0 mb-2 pa-2"
      :label="$tr('selectAllLabel')"
    />
    <VDivider />
    <EditListItem
      v-for="nodeId in nodeIds"
      :key="nodeId"
      v-model="selected"
      :nodeId="nodeId"
      @removed="handleRemoved"
    />

  </VList>

</template>

<script>

  import EditListItem from './EditListItem';
  import Checkbox from 'shared/views/form/Checkbox';

  export default {
    name: 'EditList',
    components: {
      EditListItem,
      Checkbox,
    },
    props: {
      value: {
        type: Array,
        default: () => [],
      },
      nodeIds: {
        type: Array,
        default: () => [],
      },
    },
    computed: {
      selected: {
        get() {
          return this.value;
        },
        set(items) {
          this.$emit('input', items);
        },
      },
      selectAll: {
        get() {
          return this.nodeIds.every(nodeId => this.selected.includes(nodeId));
        },
        set(value) {
          if (value) {
            this.selected = this.nodeIds;
          } else {
            this.selected = [];
          }
        },
      },
    },
    methods: {
      handleRemoved(nodeId) {
        let nodeIds = this.$route.params.detailNodeIds.split(',').filter(id => id !== nodeId);

        this.$router.push({
          name: this.$route.name,
          params: {
            nodeId: this.$route.params.nodeId,
            detailNodeIds: nodeIds.join(','),
          },
        });
        if (this.selected.includes(nodeId)) {
          if (this.selected.length === 1) {
            let viableNodes = this.nodeIds.filter(id => id !== nodeId);
            this.selected = [viableNodes[viableNodes.length - 1]];
          } else {
            this.selected = this.selected.filter(id => id !== nodeId);
          }
        }
      },
    },
    $trs: {
      selectAllLabel: 'Select all',
    },
  };

</script>

<style lang="less" scoped>

  .v-divider {
    margin-top: 0;
  }

</style>
