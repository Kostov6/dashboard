<!--
SPDX-FileCopyrightText: 2023 SAP SE or an SAP affiliate company and Gardener contributors

SPDX-License-Identifier: Apache-2.0
-->

<template>
  <g-action-button-dialog
    ref="actionDialog"
    :shoot-item="shootItem"
    width="600"
    :caption="caption"
    :text="buttonText"
    confirm-button-text="Trigger now"
    icon="mdi-refresh"
    :disabled="!isMaintenancePreconditionSatisfied"
    @dialog-opened="onConfigurationDialogOpened"
  >
    <template #content>
      <v-card-text>
        <div class="text-subtitle-1 pt-4">
          Do you want to start the maintenance of your cluster outside of the configured maintenance time window?
        </div>
        <g-maintenance-components
          ref="maintenanceComponents"
          title="The following updates might be performed"
          :hide-os-updates="!hasShootWorkerGroups"
          :selectable="false"
        />
      </v-card-text>
    </template>
  </g-action-button-dialog>
</template>

<script>
import { mapActions } from 'pinia'

import { useAppStore } from '@/store/app'

import GActionButtonDialog from '@/components/dialogs/GActionButtonDialog.vue'
import GMaintenanceComponents from '@/components/ShootMaintenance/GMaintenanceComponents'

import { shootItem } from '@/mixins/shootItem'
import { errorDetailsFromError } from '@/utils/error'

import { get } from '@/lodash'

export default {
  components: {
    GActionButtonDialog,
    GMaintenanceComponents,
  },
  mixins: [shootItem],
  inject: ['api', 'logger'],
  props: {
    text: {
      type: Boolean,
      default: false,
    },
  },
  data () {
    return {
      maintenanceTriggered: false,
    }
  },
  computed: {
    isMaintenanceToBeScheduled () {
      return this.shootGardenOperation === 'maintain'
    },
    caption () {
      if (!this.isMaintenancePreconditionSatisfied) {
        return this.maintenancePreconditionSatisfiedMessage
      }
      if (this.isMaintenanceToBeScheduled) {
        return 'Requesting to schedule cluster maintenance'
      }
      return this.buttonTitle
    },
    updateKubernetesVersion () {
      return get(this.shootItem, 'spec.maintenance.autoUpdate.kubernetesVersion', false)
    },
    updateOSVersion () {
      return get(this.shootItem, 'spec.maintenance.autoUpdate.machineImageVersion', false)
    },
    buttonTitle () {
      return 'Schedule Maintenance'
    },
    buttonText () {
      if (!this.text) {
        return
      }
      return this.buttonTitle
    },
  },
  watch: {
    isMaintenanceToBeScheduled (maintenanceToBeScheduled) {
      const isMaintenanceScheduled = !maintenanceToBeScheduled && this.maintenanceTriggered
      if (!isMaintenanceScheduled) {
        return
      }
      this.maintenanceTriggered = false

      if (!this.shootName) { // ensure that notification is not triggered by shoot resource being cleared (e.g. during navigation)
        return
      }

      this.setSuccess(`Maintenance scheduled for ${this.shootName}`)
    },
  },
  methods: {
    ...mapActions(useAppStore, [
      'setSuccess',
    ]),
    async onConfigurationDialogOpened () {
      await this.reset()
      if (await this.$refs.actionDialog.waitForDialogClosed()) {
        this.startMaintenance()
      }
    },
    async startMaintenance () {
      this.maintenanceTriggered = true

      const maintain = { 'gardener.cloud/operation': 'maintain' }
      try {
        await this.api.addShootAnnotation({ namespace: this.shootNamespace, name: this.shootName, data: maintain })
      } catch (err) {
        const errorMessage = 'Could not start maintenance'
        const errorDetails = errorDetailsFromError(err)
        const detailedErrorMessage = errorDetails.detailedMessage
        this.$refs.actionDialog.setError({ errorMessage, detailedErrorMessage })
        this.logger.error(this.errorMessage, errorDetails.errorCode, errorDetails.detailedMessage, err)

        this.maintenanceTriggered = false
        this.currentGeneration = null
      }
    },
    reset () {
      this.$nextTick(() => {
        // need to wait until component has been rendered before we can trigger the function on the ref
        this.$refs.maintenanceComponents.setComponentUpdates({
          k8sUpdates: this.updateKubernetesVersion,
          osUpdates: this.updateOSVersion,
        })
      })
    },
  },
}
</script>
