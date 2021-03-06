<template>

  <div v-if="availableSpace !== null">
    <h2>{{ $tr('storagePercentageUsed', { qty: storageUsagePercentage.toString() }) }}</h2>
    <KLinearLoader :progress="storageUsagePercentage" type="determinate" class="loader" />
    <div>
      {{
        $tr('spaceUsedOfMax',
            { qty: formatFileSize(usedSpace), max: formatFileSize(totalSpace) }
        )
      }}
    </div>

    <KFixedGrid numCols="8" gutter="10">

      <!-- ContentKindsList returns lowercased strings for each kind -->
      <template v-for="kind in contentKinds">

        <KFixedGridItem :key="`${kind}1`" span="3" class="row">
          <span :style="{ backgroundColor: theme[kind], padding: '8px', marginRight: '8px' }">
            <KIcon :icon="kind" :color="$themeTokens.textInverted" />
          </span>

          <span>{{ translateConstant(kind) }}</span>
        </KFixedGridItem>

        <KFixedGridItem :key="`${kind}2`" span="5" class="row">
          <span>{{ formatFileSize(storageUseByKind[kind]) }}</span>
        </KFixedGridItem>

      </template>

    </KFixedGrid>

    <h2>{{ $tr('requestMoreSpaceHeading') }}</h2>

    <p>

      <span>{{ $tr('requestMoreSpaceMessage') }}</span>

      <KExternalLink
        style="display: inline;"
        :text="$tr('learnMoreAboutImportingContentFromChannels')"
        href="https://kolibri-studio.readthedocs.io/en/latest/add_content.html#import-content-from-other-channels"
        target="_blank"
      />

    </p>

    <KButton
      appearance="basic-link"
      :text="toggleText"
      data-test="toggle-link"
      @click="showRequestForm = !showRequestForm"
    />
    <VSlideYTransition>
      <RequestForm v-show="showRequestForm" />
    </VSlideYTransition>
  </div>
  <LoadingText v-else />

</template>


<script>

  import { mapGetters } from 'vuex';
  import RequestForm from './RequestForm';
  import { fileSizeMixin, constantsTranslationMixin } from 'shared/mixins';
  import { ContentKindsList } from 'shared/leUtils/ContentKinds';
  import theme from 'shared/vuetify/theme';
  import LoadingText from 'shared/views/LoadingText';

  export default {
    name: 'Storage',
    components: { LoadingText, RequestForm },
    mixins: [fileSizeMixin, constantsTranslationMixin],
    data() {
      return {
        showRequestForm: false,
      };
    },
    computed: {
      ...mapGetters(['availableSpace', 'totalSpace', 'storageUseByKind']),
      contentKinds() {
        // Remove topic and h5p apps from the list
        return ContentKindsList.filter(k => !['h5p', 'topic'].includes(k));
      },
      usedSpace() {
        return this.totalSpace - this.availableSpace;
      },
      storageUsagePercentage() {
        // parseInt to force the float to int
        return parseInt((this.usedSpace / this.totalSpace) * 100);
      },
      theme() {
        return theme();
      },
      toggleText() {
        return this.showRequestForm ? this.$tr('hideFormAction') : this.$tr('showFormAction');
      },
    },
    $trs: {
      spaceUsedOfMax: '{qty} of {max}',
      storagePercentageUsed: '{qty}% storage used',
      requestMoreSpaceHeading: 'Request more space',
      requestMoreSpaceMessage:
        'Please use this form to request additional uploading storage for your Kolibri Studio account. The resources you import from our public library to your channels do not count towards your storage limit.',
      learnMoreAboutImportingContentFromChannels:
        'Learn more about how to import resources from other channels',
      showFormAction: 'Open form',
      hideFormAction: 'Close form',
    },
  };

</script>


<style lang="less" scoped>

  h2 {
    margin-top: 32px;
  }

  .row {
    padding: 16px 0;
  }

  /deep/.ui-progress-linear {
    max-width: 75%;
    height: 8px !important;
    margin: 8px 0;
    background: #e9e9e9;

    /deep/.is-determinate {
      height: 8px !important;
    }
  }

</style>
