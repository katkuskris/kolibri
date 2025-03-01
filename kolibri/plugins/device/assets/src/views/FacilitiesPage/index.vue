<template>

  <AppBarPage :title="pageTitle">

    <template #subNav>
      <DeviceTopNav />
    </template>

    <KPageContainer class="device-container">
      <HeaderWithOptions :headerText="coreString('facilitiesLabel')">
        <template #options>
          <!-- Margins to and bottom adds space when buttons are vertically stacked -->
          <KButtonGroup>
            <KButton
              :text="$tr('syncAllAction')"
              style="margin-top: 16px; margin-bottom: -16px;"
              @click="showSyncAllModal = true"
            />
            <KButton
              :text="getCommonSyncString('importFacilityAction')"
              primary
              style="margin-top: 16px; margin-bottom: -16px;"
              @click="showImportModal = true"
            />
          </KButtonGroup>
        </template>
      </HeaderWithOptions>

      <TasksBar
        v-if="activeFacilityTasks.length > 0"
        :tasks="activeFacilityTasks"
        :taskManagerLink="{ name: 'FACILITIES_TASKS_PAGE' }"
        @clearall="handleClickClearAll"
      />

      <CoreTable>
        <template #headers>
          <th>{{ coreString('facilityLabel') }}</th>
        </template>
        <template #tbody>
          <!-- On mobile, put buttons on a row of their own -->
          <tbody v-if="windowIsSmall">
            <template v-for="(facility, idx) in facilities">
              <tr
                :key="idx"
                style="border: none!important"
              >
                <td>
                  <FacilityNameAndSyncStatus
                    :facility="facility"
                    :isSyncing="facilityIsSyncing(facility)"
                    :isDeleting="facilityIsDeleting(facility)"
                    :syncHasFailed="facility.syncHasFailed"
                    :goToRoute="manageSync(facility.id)"
                  />
                </td>
              </tr>
              <!-- May cause error on device with > 10000 facilities... -->
              <tr :key="idx + 10000">
                <td style="padding: 0 0 16px 0;">
                  <!-- Gives most space possible to buttons and aligns them with text -->
                  <KButtonGroup style="margin-left: -16px; margin-right: -16px; max-width: 100%">
                    <KButton
                      :text="coreString('syncAction')"
                      :disabled="facilityIsSyncing(facility)"
                      appearance="flat-button"
                      @click="facilityForSync = facility"
                    />
                    <KButton
                      hasDropdown
                      appearance="flat-button"
                      :text="coreString('optionsLabel')"
                      :disabled="facilityIsSyncing(facility)"
                    >
                      <template #menu>
                        <KDropdownMenu
                          :options="facilityOptions(facility)"
                          @select="handleOptionSelect($event.value, facility)"
                        />
                      </template>
                    </KButton>
                  </KButtonGroup>
                </td>
              </tr>
            </template>
          </tbody>

          <!-- Non-mobile -->
          <tbody v-else>
            <tr
              v-for="(facility, idx) in facilities"
              :key="idx"
            >
              <td>
                <FacilityNameAndSyncStatus
                  :facility="facility"
                  :isSyncing="facilityIsSyncing(facility)"
                  :isDeleting="facilityIsDeleting(facility)"
                  :syncHasFailed="facility.syncHasFailed"
                  :goToRoute="manageSync(facility.id)"
                />
              </td>
              <td class="button-col">
                <KButtonGroup>
                  <KButton
                    :text="coreString('syncAction')"
                    :disabled="facilityIsSyncing(facility)"
                    appearance="flat-button"
                    @click="facilityForSync = facility"
                  />
                  <KButton
                    hasDropdown
                    appearance="flat-button"
                    :text="coreString('optionsLabel')"
                    :disabled="facilityIsSyncing(facility)"
                  >
                    <template #menu>
                      <KDropdownMenu
                        :options="facilityOptions()"
                        @select="handleOptionSelect($event.value, facility)"
                      />
                    </template>
                  </KButton>
                </KButtonGroup>
              </td>
            </tr>
          </tbody>
        </template>
      </CoreTable>

      <RemoveFacilityModal
        v-if="Boolean(facilityForRemoval)"
        :facility="facilityForRemoval"
        @success="handleRemoveSuccess"
        @cancel="facilityForRemoval = null"
      />

      <SyncAllFacilitiesModal
        v-if="showSyncAllModal"
        :facilities="facilities"
        @success="handleSyncAllSuccess"
        @cancel="showSyncAllModal = false"
      />

      <ImportFacilityModalGroup
        v-if="showImportModal"
        @success="handleStartImportSuccess"
        @cancel="showImportModal = false"
      />

      <!-- NOTE similar code for KDP Registration in SyncInterface -->
      <template v-if="Boolean(facilityForRegister)">
        <RegisterFacilityModal
          v-if="!kdpProject"
          :facility="facilityForRegister"
          :displaySkipOption="false"
          @success="handleValidateSuccess"
          @cancel="clearRegistrationState"
          @skip="handleKDPSync"
        />

        <ConfirmationRegisterModal
          v-else
          :targetFacility="facilityForRegister"
          :projectName="kdpProject.name"
          :token="kdpProject.token"
          @success="handleConfirmationSuccess"
          @cancel="clearRegistrationState"
        />
      </template>

      <SyncFacilityModalGroup
        v-if="Boolean(facilityForSync)"
        :facilityForSync="facilityForSync"
        @close="facilityForSync = null"
        @syncKDP="handleKDPSync"
        @syncPeer="handlePeerSync"
      />
    </KPageContainer>
  </AppBarPage>

</template>


<script>

  import commonCoreStrings from 'kolibri.coreVue.mixins.commonCoreStrings';
  import responsiveWindowMixin from 'kolibri.coreVue.mixins.responsiveWindowMixin';
  import commonSyncElements from 'kolibri.coreVue.mixins.commonSyncElements';
  import AppBarPage from 'kolibri.coreVue.components.AppBarPage';
  import CoreTable from 'kolibri.coreVue.components.CoreTable';
  import { FacilityResource } from 'kolibri.resources';
  import {
    FacilityNameAndSyncStatus,
    RegisterFacilityModal,
    ConfirmationRegisterModal,
    SyncFacilityModalGroup,
  } from 'kolibri.coreVue.componentSets.sync';
  import { TaskStatuses, TaskTypes } from 'kolibri.utils.syncTaskUtils';
  import { PageNames } from '../../constants';
  import { deviceString } from '../commonDeviceStrings';
  import TasksBar from '../ManageContentPage/TasksBar';
  import DeviceTopNav from '../DeviceTopNav';
  import HeaderWithOptions from '../HeaderWithOptions';
  import RemoveFacilityModal from './RemoveFacilityModal';
  import SyncAllFacilitiesModal from './SyncAllFacilitiesModal';
  import ImportFacilityModalGroup from './ImportFacilityModalGroup';
  import facilityTaskQueue from './facilityTasksQueue';

  const Options = Object.freeze({
    REGISTER: 'REGISTER',
    REMOVE: 'REMOVE',
    MANAGESYNC: 'MANAGE SYNC',
  });

  export default {
    name: 'FacilitiesPage',
    metaInfo() {
      return {
        title: this.coreString('facilitiesLabel'),
      };
    },
    components: {
      AppBarPage,
      ConfirmationRegisterModal,
      CoreTable,
      DeviceTopNav,
      HeaderWithOptions,
      FacilityNameAndSyncStatus,
      ImportFacilityModalGroup,
      RegisterFacilityModal,
      RemoveFacilityModal,
      SyncFacilityModalGroup,
      SyncAllFacilitiesModal,
      TasksBar,
    },
    mixins: [commonCoreStrings, commonSyncElements, facilityTaskQueue, responsiveWindowMixin],
    data() {
      return {
        showSyncAllModal: false,
        showImportModal: false,
        facilities: [],
        facilityForSync: null,
        facilityForRemoval: null,
        facilityForRegister: null,
        kdpProject: null, // { name, token }
        taskIdsToWatch: [],
      };
    },
    computed: {
      pageTitle() {
        return deviceString('deviceManagementTitle');
      },
    },
    watch: {
      // Update facilities whenever a watched task completes
      facilityTasks(newTasks) {
        for (const index in newTasks) {
          const task = newTasks[index];
          if (this.taskIdsToWatch.includes(task.id)) {
            if (task.status === TaskStatuses.COMPLETED) {
              this.fetchFacilites();
              if (task.type === TaskTypes.DELETEFACILITY) {
                this.showFacilityRemovedSnackbar(task.facility_name);
              }
              this.taskIdsToWatch = this.taskIdsToWatch.filter(x => x !== task.id);
            } else if (this.isSyncTask(task) && task.status === TaskStatuses.FAILED) {
              const match = this.facilities.find(({ id }) => id === task.facility);
              if (match) {
                this.$set(match, 'syncHasFailed', true);
              }
            }
          } else {
            // Add tasks that aren't being watched yet
            if (!task.clearable) {
              this.taskIdsToWatch.push(task.id);
            }
          }
        }
      },
    },
    beforeMount() {
      this.fetchFacilites();
    },
    methods: {
      facilityOptions() {
        return [
          {
            label: this.coreString('manageSyncAction'),
            value: Options.MANAGESYNC,
          },
          {
            label: this.coreString('registerAction'),
            value: Options.REGISTER,
          },
          {
            label: this.coreString('removeAction'),
            value: Options.REMOVE,
          },
        ];
      },
      handleOptionSelect(option, facility) {
        if (option === Options.REMOVE) {
          this.facilityForRemoval = facility;
        } else if (option === Options.REGISTER) {
          this.facilityForRegister = facility;
        } else if (option === Options.MANAGESYNC) {
          const route = this.manageSync(facility.id);
          this.$router.push(route);
        }
      },
      fetchFacilites() {
        return FacilityResource.fetchCollection({ force: true }).then(facilities => {
          this.facilities = [...facilities];
        });
      },
      handleClickClearAll() {
        Promise.all([this.fetchFacilites(), this.clearCompletedFacilityTasks()]);
      },
      handleValidateSuccess({ name, token }) {
        this.kdpProject = { name, token };
      },
      handleConfirmationSuccess() {
        this.fetchFacilites();
        this.clearRegistrationState();
      },
      clearRegistrationState() {
        this.facilityForRegister = null;
        this.kdpProject = null;
      },
      handleStartSyncSuccess(task) {
        this.taskIdsToWatch.push(task.id);
        this.facilityTasks.push(task);
        this.facilityForSync = null;
      },
      handleSyncAllSuccess() {
        this.showSyncAllModal = false;
      },
      handleStartImportSuccess() {
        this.$router.push({
          name: 'FACILITIES_TASKS_PAGE',
        });
        this.showImportModal = false;
      },
      manageSync(facilityId) {
        return {
          name: PageNames.MANAGE_SYNC_SCHEDULE,
          facilityId,
          params: {
            facilityId,
          },
        };
      },
      showFacilityRemovedSnackbar(facilityName) {
        this.$store.dispatch(
          'createSnackbar',
          this.$tr('facilityRemovedSnackbar', {
            facilityName,
          })
        );
      },
      handleRemoveSuccess(taskId) {
        this.taskIdsToWatch.push(taskId);
        this.facilityForRemoval = null;
      },
      handleKDPSync(facility) {
        this.facilityForSync = null;
        this.facilityForRegister = null;
        this.startKdpSyncTask(facility.id)
          .then(task => {
            this.handleStartSyncSuccess(task);
          })
          .catch(() => {
            this.$emit('failure');
          });
      },
      handlePeerSync(peerData, facility) {
        this.facilityForSync = null;
        this.startPeerSyncTask({
          facility: facility.id,
          device_id: peerData.id,
        })
          .then(task => {
            this.handleStartSyncSuccess(task);
          })
          .catch(() => {
            this.$emit('failure');
          });
      },
    },
    $trs: {
      syncAllAction: {
        message: 'Sync all',
        context:
          'Label for a button used to synchronize all facilities at once with the data portal',
      },
      facilityRemovedSnackbar: {
        message: "Removed '{facilityName}' from this device",
        context:
          "Notification that appears after a facility has been deleted. For example, \"Removed 'Zuk Village' from this device'.",
      },
    },
  };

</script>


<style lang="scss" scoped>

  @import '../../styles/definitions';

  .device-container {
    @include device-kpagecontainer;
  }

  .buttons {
    margin: auto;
  }

  .button-col {
    padding: 4px;
    padding-top: 8px;
    text-align: right;
    vertical-align: middle;

    .sync {
      margin-right: 0;
    }
  }

</style>
