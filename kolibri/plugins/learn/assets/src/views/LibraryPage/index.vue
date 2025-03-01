<template>

  <LearnAppBarPage
    :appBarTitle="appBarTitle"
    :loading="getRootNodesLoading"
    :appearanceOverrides="{}"
    :deviceId="deviceId"
    :route="back"
  >
    <main
      class="main-grid"
      :style="gridOffset"
    >
      <div v-if="!windowIsLarge && !isLocalLibraryEmpty && !deviceId">
        <KButton
          icon="filter"
          data-test="filter-button"
          class="filter-button"
          :text="coreString('filter')"
          :primary="false"
          @click="toggleSidePanelVisibility"
        />
      </div>
      <!--
        - If search is loading, show loader.
        - If there are no search results, show channels and resumable
        content.
        - Otherwise, show search results.
      -->
      <KCircularLoader
        v-if="searchLoading"
        class="loader"
        type="indeterminate"
        :delay="false"
      />
      <div
        v-else-if="!displayingSearchResults && !getRootNodesLoading"
        data-test="channels"
      >
        <h1 class="channels-label">
          {{ channelsLabel }}
        </h1>
        <p
          v-if="isLocalLibraryEmpty"
          data-test="nothing-in-lib-label"
          class="nothing-in-lib-label"
        >
          {{ coreString('nothingInLibraryLearner') }}
        </p>
        <ChannelCardGroupGrid
          v-if="!isLocalLibraryEmpty"
          data-test="channel-cards"
          class="grid"
          :contents="rootNodes"
          :deviceId="deviceId"
        />
        <!-- ResumableContentGrid mostly handles whether it renders or not internally !-->
        <!-- but we conditionalize it based on whether we are on another device's library page !-->
        <ResumableContentGrid
          v-if="!deviceId"
          data-test="resumable-content"
          :currentCardViewStyle="currentCardViewStyle"
          @setCardStyle="style => currentCardViewStyle = style"
          @setSidePanelMetadataContent="content => metadataSidePanelContent = content"
        />
        <!-- Other Libraires -->
        <div
          v-if="!deviceId && isUserLoggedIn"
          data-test="other-libraries"
        >
          <KGrid gutter="12">
            <KGridItem
              :layout12="{ span: 6 }"
              :layout8="{ span: 4 }"
              :layout4="{ span: 4 }"
            >
              <h1>
                {{ $tr('otherLibraries') }}
              </h1>
            </KGridItem>
            <KGridItem
              :layout12="{ span: 6 }"
              :layout8="{ span: 4 }"
              :layout4="{ span: 4 }"
            >
              <div class="sync-status">
                <span
                  v-show="searching"
                  data-test="searching"
                >
                  <span data-test="searching-label">{{ $tr('searchingOtherLibrary') }}</span>
                  &nbsp;&nbsp;
                  <span>
                    <KCircularLoader
                      type="indeterminate"
                      :stroke="6"
                    />
                  </span>
                </span>
                <span
                  v-show="!searching && devicesWithChannelsExist"
                  data-test="showing-all"
                >
                  <span>
                    <KIcon
                      v-if="windowIsSmall"
                      icon="wifi"
                    />
                  </span>
                  &nbsp;&nbsp;
                  <span data-test="showing-all-label">{{ showingAllLibrariesLabel }}</span>
                  &nbsp;&nbsp;
                  <KButton
                    :text="coreString('refresh')"
                    appearance="basic-link"
                    @click="refreshDevices"
                  />
                  &nbsp;
                  <span>
                    <KIcon
                      v-if="!windowIsSmall"
                      icon="wifi"
                    />
                  </span>
                </span>
                <span
                  v-show="!searching && !devicesWithChannelsExist"
                  data-test="no-other"
                >
                  <span>
                    <KIcon icon="disconnected" />
                  </span>
                  &nbsp;&nbsp;
                  <span data-test="no-other-label">{{ $tr('noOtherLibraries') }}</span>
                  &nbsp;&nbsp;
                  <KButton
                    :text="coreString('refresh')"
                    appearance="basic-link"
                    @click="refreshDevices"
                  />
                </span>
              </div>
            </KGridItem>
          </KGrid>

          <h2
            v-if="pinnedDevicesExist && unpinnedDevicesExist"
            data-test="pinned-label"
          >
            {{ $tr('pinned') }}
          </h2>
          <PinnedNetworkResources
            v-if="pinnedDevicesExist"
            data-test="pinned-resources"
            :devices="pinnedDevices"
          />

          <!-- More  -->
          <h2
            v-if="pinnedDevicesExist && unpinnedDevicesExist"
            data-test="more-label"
          >
            {{ $tr('moreLibraries') }}
          </h2>
          <MoreNetworkDevices
            v-if="unpinnedDevicesExist"
            data-test="more-devices"
            :devices="unpinnedDevices"
          />
        </div>

      </div>

      <SearchResultsGrid
        v-else-if="displayingSearchResults"
        data-test="search-results"
        :results="results"
        :removeFilterTag="removeFilterTag"
        :clearSearch="clearSearch"
        :moreLoading="moreLoading"
        :searchMore="searchMore"
        :currentCardViewStyle="currentCardViewStyle"
        :searchTerms="searchTerms"
        :searchLoading="searchLoading"
        :more="more"
        @setCardStyle="style => currentCardViewStyle = style"
        @setSidePanelMetadataContent="content => metadataSidePanelContent = content"
      />
    </main>

    <!-- Side Panels for filtering and searching  -->
    <SidePanel
      v-if="!isLocalLibraryEmpty && !deviceId"
      ref="sidePanel"
      data-test="side-panel"
      :searchTerms="searchTerms"
      :mobileSidePanelIsOpen="mobileSidePanelIsOpen"
      @toggleMobileSidePanel="toggleSidePanelVisibility"
      @setSearchTerms="newTerms => searchTerms = newTerms"
      @setCategory="category => setCategory(category)"
    />

    <!-- Side Panel for metadata -->
    <SidePanelModal
      v-if="metadataSidePanelContent"
      data-test="side-panel-modal"
      alignment="right"
      :closeButtonIconType="close"
      @closePanel="metadataSidePanelContent = null"
      @shouldFocusFirstEl="findFirstEl()"
    >
      <template
        v-if="metadataSidePanelContent.learning_activities.length"
        #header
      >
        <!-- Flex styles tested in ie11 and look good. Ensures good spacing between
            multiple chips - not a common thing but just in case -->
        <div
          v-for="activity in metadataSidePanelContent.learning_activities"
          :key="activity"
          class="side-panel-chips"
          :class="$computedClass({ '::after': {
            content: '',
            flex: 'auto'
          } })"
        >
          <LearningActivityChip
            class="chip"
            style="margin-left: 8px; margin-bottom: 8px;"
            :kind="activity"
          />
        </div>
      </template>

      <BrowseResourceMetadata
        ref="resourcePanel"
        :content="metadataSidePanelContent"
        :showLocationsInChannel="true"
      />
    </SidePanelModal>
  </LearnAppBarPage>

</template>


<script>

  import { mapGetters, mapState } from 'vuex';
  import cloneDeep from 'lodash/cloneDeep';

  import { onMounted } from 'kolibri.lib.vueCompositionApi';
  import commonCoreStrings from 'kolibri.coreVue.mixins.commonCoreStrings';
  import useKResponsiveWindow from 'kolibri.coreVue.composables.useKResponsiveWindow';
  import { currentLanguage } from 'kolibri.utils.i18n';
  import SidePanelModal from '../SidePanelModal';
  import { KolibriStudioId } from '../../constants';
  import useCardViewStyle from '../../composables/useCardViewStyle';
  import useContentLink from '../../composables/useContentLink';
  import useDevices from '../../composables/useDevices';
  import usePinnedDevices from '../../composables/usePinnedDevices';
  import useSearch from '../../composables/useSearch';
  import useLearnerResources from '../../composables/useLearnerResources';
  import BrowseResourceMetadata from '../BrowseResourceMetadata';
  import commonLearnStrings from '../commonLearnStrings';
  import ChannelCardGroupGrid from '../ChannelCardGroupGrid';
  import LearningActivityChip from '../LearningActivityChip';
  import SearchResultsGrid from '../SearchResultsGrid';
  import LearnAppBarPage from '../LearnAppBarPage';
  import useChannels from './../../composables/useChannels';
  import ResumableContentGrid from './ResumableContentGrid';
  import PinnedNetworkResources from './PinnedNetworkResources';
  import MoreNetworkDevices from './MoreNetworkDevices';
  import SidePanel from './SidePanel';

  export default {
    name: 'LibraryPage',
    metaInfo() {
      return {
        title: this.learnString('learnLabel'),
      };
    },
    components: {
      BrowseResourceMetadata,
      ChannelCardGroupGrid,
      SidePanelModal,
      LearningActivityChip,
      ResumableContentGrid,
      SearchResultsGrid,
      SidePanel,
      LearnAppBarPage,
      PinnedNetworkResources,
      MoreNetworkDevices,
    },
    mixins: [commonLearnStrings, commonCoreStrings],
    setup() {
      const {
        searchTerms,
        displayingSearchResults,
        searchLoading,
        moreLoading,
        results,
        more,
        search,
        searchMore,
        removeFilterTag,
        clearSearch,
        setCategory,
        currentRoute,
      } = useSearch();
      const {
        resumableContentNodes,
        moreResumableContentNodes,
        fetchMoreResumableContentNodes,
      } = useLearnerResources();
      const {
        windowBreakpoint,
        windowIsLarge,
        windowIsMedium,
        windowIsSmall,
      } = useKResponsiveWindow();
      const { currentCardViewStyle } = useCardViewStyle();
      const { back } = useContentLink();
      const { baseurl, deviceName, fetchDevices } = useDevices();
      const { fetchChannels } = useChannels();
      const { fetchPinsForUser } = usePinnedDevices();

      onMounted(() => {
        const keywords = currentRoute().query.keywords;
        if (keywords && keywords.length) {
          search(keywords);
        }
      });

      return {
        displayingSearchResults,
        searchTerms,
        searchLoading,
        moreLoading,
        results,
        more,
        search,
        searchMore,
        removeFilterTag,
        clearSearch,
        setCategory,
        resumableContentNodes,
        moreResumableContentNodes,
        fetchMoreResumableContentNodes,
        windowBreakpoint,
        windowIsLarge,
        windowIsMedium,
        windowIsSmall,
        currentCardViewStyle,
        baseurl,
        fetchDevices,
        deviceName,
        fetchChannels,
        fetchPinsForUser,
        back,
      };
    },
    props: {
      deviceId: {
        type: String,
        default: null,
      },
    },
    data: function() {
      return {
        isLocalLibraryEmpty: false,
        metadataSidePanelContent: null,
        mobileSidePanelIsOpen: false,
        devices: [],
        searching: true,
        usersPins: [],
      };
    },
    computed: {
      ...mapState(['rootNodes']),
      ...mapGetters(['isSuperuser', 'isUserLoggedIn', 'getRootNodesLoading']),
      appBarTitle() {
        return this.learnString(this.deviceId ? 'exploreLibraries' : 'learnLabel');
      },
      channelsLabel() {
        if (this.deviceId) {
          if (this.deviceId === this.studioId) {
            return this.learnString('kolibriLibrary');
          } else {
            return this.$tr('libraryOf', { device: this.deviceName });
          }
        } else {
          return this.coreString('yourLibrary');
        }
      },
      channelsToDisplay() {
        return this.windowIsSmall ? 3 : 7;
      },
      devicesWithChannels() {
        //display Kolibri studio for superusers only
        return cloneDeep(this.devices).filter(device => {
          device['channels'] = device.channels?.slice(0, this.channelsToDisplay);
          return (
            device.channels?.length > 0 &&
            (device.instance_id !== this.studioId || this.isSuperuser)
          );
        });
      },
      devicesWithChannelsExist() {
        return this.devicesWithChannels.length > 0;
      },
      gridOffset() {
        const marginTop =
          !this.windowIsLarge && (this.isLocalLibraryEmpty || this.deviceId) ? '140px' : '110px';
        return this.isRtl
          ? { marginRight: `${this.sidePanelWidth + 24}px`, marginTop }
          : { marginLeft: `${this.sidePanelWidth + 24}px`, marginTop };
      },
      layoutSpan() {
        /**
         * The breakpoints below represent the window widths
         * 0: < 480px  | Small screen  | 4 columns
         * 1: < 600px  | Small screen  | 4 columns
         * 2: < 840px  | Medium screen | 8 columns
         * 3: < 960px  | Large screen  | 12 columns
         * 4: < 1280px | Large screen  | 12 columns
         * 5: < 1440px | Large screen  | 12 columns
         * 6: < 1600px | Large screen  | 12 columns
         *
         * On resize, display X cards per row where:
         * X = total columns in grid / column span for each card.
         * For example, if the total number of columns is 12, and
         * column span for each cards is 4, then X is 3.
         */
        let span = 3;
        if ([0, 1, 2, 6].includes(this.windowBreakpoint)) {
          span = 4;
        } else if ([3, 4, 5].includes(this.windowBreakpoint)) {
          span = 6;
        }
        return span;
      },
      pinnedDevices() {
        return this.devicesWithChannels.filter(device => {
          return (
            this.usersPinsDeviceIds.includes(device.instance_id) ||
            device.instance_id === this.studioId
          );
        });
      },
      pinnedDevicesExist() {
        return this.pinnedDevices.length > 0;
      },
      sidePanelWidth() {
        if (
          this.windowIsSmall ||
          this.windowIsMedium ||
          this.isLocalLibraryEmpty ||
          this.deviceId
        ) {
          return 0;
        } else if (this.windowBreakpoint < 4) {
          return 234;
        } else {
          return 346;
        }
      },
      showingAllLibrariesLabel() {
        let label = this.$tr('showingAllLibraries');
        if (label.slice(-1) === '.') {
          label = label.slice(0, -1);
        }
        return label;
      },
      studioId() {
        return KolibriStudioId;
      },
      unpinnedDevices() {
        return this.devicesWithChannels.filter(device => {
          return (
            !this.usersPinsDeviceIds.includes(device.instance_id) &&
            device.instance_id !== this.studioId
          );
        });
      },
      unpinnedDevicesExist() {
        return this.unpinnedDevices.length > 0;
      },
      usersPinsDeviceIds() {
        return this.usersPins.map(pin => pin.instance_id);
      },
    },
    provide() {
      return {
        $layoutSpan: () => this.layoutSpan,
      };
    },
    watch: {
      rootNodes(newNodes) {
        this.isLocalLibraryEmpty = !newNodes.length;
      },
      searchTerms() {
        this.mobileSidePanelIsOpen = false;
      },
      mobileSidePanelIsOpen() {
        if (this.mobileSidePanelIsOpen) {
          document.documentElement.style.position = 'fixed';
          return;
        }
        document.documentElement.style.position = '';
      },
    },
    created() {
      this.search();
      if (this.isUserLoggedIn) {
        this.refreshDevices();
      }
    },
    methods: {
      addDevice(device, channels) {
        this.devices.push(
          Object.assign(device, {
            channels: channels.sort(this.currentLanguageChannelsFirst),
            total_count: channels.length,
          })
        );
      },
      currentLanguageChannelsFirst(a, b) {
        return b['lang_code'].indexOf(currentLanguage) - a['lang_code'].indexOf(currentLanguage);
      },
      findFirstEl() {
        this.$refs.resourcePanel.focusFirstEl();
      },
      refreshDevices() {
        this.searching = true;
        this.devices = [];
        this.fetchPinsForUser().then(resp => {
          this.usersPins = resp.map(pin => {
            const instance_id = pin.instance_id.replace(/-/g, '');
            return { ...pin, instance_id };
          });
        });

        this.fetchDevices().then(devices => {
          this.searching = false;
          for (const device of devices) {
            const baseurl = device.base_url;
            this.fetchChannels({ baseurl })
              .then(channels => {
                this.addDevice(device, channels);
              })
              .catch(() => {
                this.addDevice(device, []);
              });
          }
        });
      },
      toggleSidePanelVisibility() {
        this.mobileSidePanelIsOpen = !this.mobileSidePanelIsOpen;
      },
    },
    $trs: {
      libraryOf: {
        message: 'Library of {device}',
        context: 'A header for a device Library',
      },
      otherLibraries: {
        message: 'Other libraries',
        context: 'Header for viewing other remote content Library',
      },
      searchingOtherLibrary: {
        message: 'Searching for libraries around you.',
        context: 'Connection state for showing other library',
      },
      /* eslint-disable kolibri/vue-no-unused-translations */
      noOtherLibraries: {
        message: 'No other libraries around you right now',
        context: 'Connection state when there is no other libraries around',
      },
      /* eslint-enable kolibri/vue-no-unused-translations */
      showingAllLibraries: {
        message: 'Showing all available libraries around you.',
        context: 'Connection state when the device is connected and shows other libraries',
      },
      moreLibraries: {
        message: 'More',
        context: 'Title section containing unpinned devices',
      },
      pinned: {
        message: 'Pinned',
        context: 'Sub heading for the pinned devices',
      },
    },
  };

</script>


<style lang="scss" scoped>

  @import '~kolibri-design-system/lib/styles/definitions';
  @import '../HybridLearningContentCard/card';

  $margin: 24px;

  .card-main-wrapper {
    @extend %dropshadow-1dp;

    position: relative;
    display: inline-flex;
    width: 100%;
    height: 130px;
    max-height: 258px;
    padding-bottom: $margin;
    text-decoration: none;
    vertical-align: top;
    border-radius: $radius;
    transition: box-shadow $core-time ease;

    &:hover {
      @extend %dropshadow-8dp;
    }

    &:focus {
      outline-width: 4px;
      outline-offset: 6px;
    }
  }

  .filter-button {
    margin-top: 30px;
  }

  .main-grid {
    margin-top: 110px;
    margin-right: 24px;
  }

  .channels-label {
    margin-bottom: 12px;
  }

  .nothing-in-lib-label {
    padding-top: 0;
    margin-top: 0;
  }

  .loader {
    margin-top: 60px;
  }

  .side-panel-chips {
    display: flex;
    flex-wrap: wrap;
    justify-content: flex-start;
    margin-bottom: -8px;
    margin-left: -8px;
  }

  .chip {
    margin-bottom: 8px;
    margin-left: 8px;
  }

  .sync-status {
    display: flex;
    justify-content: flex-end;
    margin: 30px 0 10px;

    span {
      display: inline-flex;
      vertical-align: bottom;
    }
  }

  .network-device-refresh {
    display: inline-block;
    margin: 0 4px;
  }

  .view-all-text {
    margin: auto;
    font-size: 16px;
  }

</style>
