<template>
  <v-card outlined class="mx-auto inspection-panel">
    <v-progress-linear indeterminate v-if="loading" color="primary" class="mb-2" />
    <v-alert v-if="error" type="error" tile dense>{{ error }}</v-alert>

    <h2 class="inspection-heading">Status</h2>
    <div v-if="!loading">
      <p>
        <strong>Current Status:</strong>
        {{ currentStatus.statusCodeDetail.display }}
        <br />
        <strong>Assigned To:</strong>
        {{ currentStatus.assignedTo ? currentStatus.assignedTo : 'N/A' }}
        <span
          v-if="currentStatus.assignedToEmail"
        >({{ currentStatus.assignedToEmail }})</span>
        <br />
        <strong>Effective Date:</strong>
        {{ actionDateDisplay }}
      </p>

      <v-form v-if="!error" ref="form" v-model="valid" lazy-validation>
        <v-row>
          <v-col cols="12" xl="8" offset-xl="2">
            <label>Update Status</label>
            <v-select
              block
              dense
              flat
              outlined
              solo
              single-line
              label="Select status to set"
              :items="items"
              item-text="display"
              item-value="code"
              v-model="statusToSet"
              data-test="select-inspection-statusToSet"
              :rules="[v => !!v || 'Status is required']"
              :disabled="!hasReviewer"
              @change="statusFields = true"
            />

            <div v-show="statusFields">
              <div v-if="showActionDate">
                <label>Effective Date (Optional)</label>
                <v-menu
                  v-model="actionDateMenu"
                  data-test="menu-inspection-actionDateMenu"
                  :close-on-content-click="true"
                  :nudge-right="40"
                  transition="scale-transition"
                  offset-y
                  min-width="290px"
                >
                  <template v-slot:activator="{ on }">
                    <v-text-field
                      v-model="actionDate"
                      data-test="text-inspection-actionDate"
                      placeholder="yyyy-mm-dd"
                      append-icon="event"
                      v-on:click:append="actionDateMenu=true"
                      readonly
                      v-on="on"
                      dense
                      flat
                      outlined
                      solo
                    />
                  </template>
                  <v-date-picker
                    v-model="actionDate"
                    data-test="picker-inspection-actionDate"
                    @input="actionDateMenu = false"
                  />
                </v-menu>
              </div>

              <div v-if="showInspector">
                <label>Assignee Name</label>
                <v-text-field
                  v-model="assignedTo"
                  data-test="text-inspection-assignedTo"
                  :rules="[v => !!v || 'Name is required']"
                  dense
                  flat
                  outlined
                  solo
                />

                <label>Assignee Email (Optional)</label>
                <v-text-field
                  v-model="assignedToEmail"
                  data-test="text-inspection-assignedToEmail"
                  dense
                  flat
                  outlined
                  solo
                />

                <div class="text-right">
                  <v-btn
                    text
                    small
                    color="primary"
                    class="pl-0 my-0 text-end"
                    @click="assignToCurrentUser"
                    data-test="btn-inspection-assign-self"
                  >
                    <v-icon class="mr-1">person</v-icon>
                    <span>ASSIGN TO ME</span>
                  </v-btn>
                </div>
              </div>

              <label>Note (Optional)</label>
              <v-textarea
                v-model="note"
                data-test="text-inspection-note"
                :rules="[v => v.length <= 4000 || 'Max 4000 characters']"
                rows="1"
                counter
                auto-grow
                dense
                flat
                outlined
                solo
              />
            </div>
          </v-col>
        </v-row>

        <v-row>
          <v-col cols="12" sm="6" xl="4" offset-xl="2">
            <v-dialog v-model="historyDialog" width="1200">
              <template v-slot:activator="{ on }">
                <v-btn
                  block
                  outlined
                  color="textLink"
                  data-test="btn-inspection-view-history"
                  v-on="on"
                >
                  <span>VIEW HISTORY</span>
                </v-btn>
              </template>

              <v-card v-if="historyDialog">
                <v-card-title class="headline grey lighten-3" primary-title>Status History</v-card-title>

                <StatusTable :submissionId="submissionId" :formName="formName" class="my-4" />

                <v-divider />
                <v-card-actions>
                  <v-spacer />
                  <v-btn
                    @click="historyDialog = false"
                    color="primary"
                    data-test="btn-inspection-close-status-table"
                    text
                  >
                    <span>CLOSE</span>
                  </v-btn>
                </v-card-actions>
              </v-card>
            </v-dialog>
          </v-col>

          <v-col cols="12" sm="6" xl="4" order="first" order-sm="last">
            <v-btn
              block
              color="primary"
              data-test="btn-inspection-update-status"
              :disabled="!hasReviewer"
              v-on="on"
              @click="updateStatus"
            >
              <span>UPDATE</span>
            </v-btn>
          </v-col>
        </v-row>
      </v-form>
    </div>
  </v-card>
</template>

<script>
import moment from 'moment';
import { mapGetters } from 'vuex';

import StatusTable from '@/components/common/admin/inspection/StatusTable.vue';
import commonFormService from '@/services/commonFormService';
import { AppRoles, getAppClient } from '@/utils/constants';

export default {
  name: 'InspectionPanel',
  components: {
    StatusTable
  },
  props: {
    formName: {
      type: String,
      required: true
    },
    submissionId: {
      required: true,
      type: String
    }
  },
  data() {
    return {
      on: false,
      error: '',
      currentStatus: {},
      historyDialog: false,
      actionDateMenu: false,
      loading: true,
      statusHistory: {},
      statusFields: false,
      statusToSet: '',
      valid: false,

      // Fields
      assignedTo: this.currentStatus ? this.currentStatus.assignedTo : '',
      assignedToEmail: this.currentStatus
        ? this.currentStatus.assignedToEmail
        : '',
      actionDate: '',
      note: ''
    };
  },
  computed: {
    ...mapGetters('auth', ['hasResourceRoles', 'email', 'token', 'fullName']),

    // State machine
    items() {
      if (
        this.currentStatus &&
        this.currentStatus.statusCodeDetail &&
        this.currentStatus.statusCodeDetail.nextCodes
      ) {
        return this.currentStatus.statusCodeDetail.nextCodes.filter(
          n => n.enabled
        );
      } else {
        return [];
      }
    },
    showInspector() {
      return ['ASSIGNED'].includes(this.statusToSet);
    },
    showActionDate() {
      return ['ASSIGNED', 'COMPLETED'].includes(this.statusToSet);
    },
    actionDateDisplay() {
      return this.currentStatus.actionDate
        ? moment(this.currentStatus.actionDate).format('MMMM D YYYY')
        : 'N/A';
    },
    hasReviewer() {
      return this.hasResourceRoles(getAppClient(this.formName), [
        AppRoles.REVIEWER
      ]);
    }
  },
  methods: {
    async getInspectionData() {
      this.loading = true;
      try {
        const statuses = await commonFormService.getSubmissionStatuses(
          this.formName,
          this.submissionId
        );
        this.statusHistory = statuses.data;
        if (!this.statusHistory.length || !this.statusHistory[0]) {
          this.error = 'No inspection statuses found';
        } else {
          // Statuses are returned in date precedence, the 0th item in the array is the current status
          this.currentStatus = this.statusHistory[0];
          const scRes = await commonFormService.getStatusCodes(this.formName);
          const statusCodes = scRes.data;
          if (!statusCodes.length) {
            throw new Error('error finding status codes');
          }
          this.currentStatus.statusCodeDetail = statusCodes.find(
            sc => sc.code === this.currentStatus.code
          );
        }
      } catch (error) {
        this.error = 'Error occured fetching status for this submission';
        console.error(`Error getting statuses: ${error.message}`); // eslint-disable-line no-console
      } finally {
        this.loading = false;
      }
    },
    assignToCurrentUser() {
      this.assignedTo = this.fullName;
      this.assignedToEmail = this.email;
    },
    resetForm() {
      this.statusFields = false;
      this.$refs.form.resetValidation();
      this.statusToSet = null;
      this.statusFields = false;
      this.note = '';
    },
    async updateStatus() {
      try {
        this.error = '';
        if (this.$refs.form.validate()) {
          if (!this.statusToSet) {
            throw new Error('No Status');
          }

          const statusBody = {
            code: this.statusToSet
          };
          if (this.showInspector) {
            if (this.assignedTo) {
              statusBody.assignedTo = this.assignedTo;
            }
            if (this.assignedToEmail) {
              statusBody.assignedToEmail = this.assignedToEmail;
            }
          }
          if (this.actionDate && this.showActionDate) {
            statusBody.actionDate = this.actionDate;
          }
          const statusResponse = await commonFormService.sendSubmissionStatuses(
            this.formName,
            this.submissionId,
            statusBody
          );
          if (!statusResponse.data) {
            throw new Error(
              'No response data from API while submitting status update form'
            );
          }
          if (this.note) {
            const submissionStatusId = statusResponse.data.submissionStatusId;
            const noteBody = {
              submissionId: this.submissionId,
              submissionStatusId: submissionStatusId,
              note: this.note
            };
            const response = await commonFormService.addNoteToStatus(
              this.formName,
              this.submissionId,
              submissionStatusId,
              noteBody
            );
            if (!response.data) {
              throw new Error(
                'No response data from API while submitting note for status update'
              );
            }
            // Update the parent if the note was updated
            this.$emit('note-updated');
          }
          this.resetForm();
          this.getInspectionData();
        }
      } catch (error) {
        console.error(`Error updating status: ${error}`); // eslint-disable-line no-console
        this.error = 'An error occured while trying to update the status';
      }
    }
  },
  created() {
    this.getInspectionData();
  }
};
</script>

<style lang="scss" scoped>
.inspection-panel {
  background-color: #f5f5f5;
  margin-bottom: 2em;
  padding: 1em 1.5em;
  .inspection-heading {
    font-size: 1.5em;
    margin-bottom: 0.5em;
    color: #003366;
  }
}
</style>
