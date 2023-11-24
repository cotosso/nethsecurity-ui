<script setup lang="ts">
import { useTimer } from '@/composables/useTimer'
import { ubusCall } from '@/lib/standalone/ubus'
import type { PackageUpdate } from '@/views/standalone/system/UpdateView.vue'
import {
  getAxiosErrorMessage,
  NeModal,
  NeInlineNotification,
  NeProgressBar
} from '@nethserver/vue-tailwind-lib'
import { ref } from 'vue'
import { useI18n } from 'vue-i18n'

const UPDATE_WAIT_TIME = 20000

defineProps<{
  packageUpdates: PackageUpdate[]
}>()

const error = ref({
  notificationDescription: '',
  notificationDetails: ''
})

const emit = defineEmits(['close', 'packages-updated'])

const { t } = useI18n()

const isSubmittingUpdateRequest = ref(false)
const isUpdatingPackages = ref(false)

const { startTimer, currentProgress, clearTimer } = useTimer({
  duration: UPDATE_WAIT_TIME,
  progressStep: 0.5,
  onTimerFinish: () => {
    emit('packages-updated')
    clearTimer()
    close()
  }
})

function close() {
  isUpdatingPackages.value = false
  error.value.notificationDescription = ''
  error.value.notificationDetails = ''
  emit('close')
}

async function updatePackages() {
  try {
    isSubmittingUpdateRequest.value = true
    await ubusCall('ns.update', 'install-package-updates')
    isUpdatingPackages.value = true
    startTimer()
  } catch (err: any) {
    error.value.notificationDescription = t(getAxiosErrorMessage(err))
    error.value.notificationDetails = err.toString()
  } finally {
    isSubmittingUpdateRequest.value = false
  }
}
</script>

<template>
  <NeModal
    :visible="packageUpdates.length > 0"
    kind="info"
    :title="t('standalone.update.bug_security_fixes_to_update')"
    :primary-label="t('standalone.update.title')"
    :primary-button-disabled="isUpdatingPackages"
    :primary-button-loading="isUpdatingPackages"
    :cancel-label="!isUpdatingPackages ? t('common.cancel') : ''"
    @close="!isUpdatingPackages ? close() : undefined"
    @primary-click="updatePackages"
  >
    <template v-if="!isUpdatingPackages"
      ><p v-for="item in packageUpdates" :key="item.package">
        {{ item.package }}
        <span class="text-gray-500 dark:text-gray-400">{{
          t('standalone.update.component_update_details', {
            versionFrom: item.currentVersion,
            versionTo: item.lastVersion
          })
        }}</span>
      </p>
      <NeInlineNotification
        v-if="error.notificationDescription"
        :title="t('error.cannot_update_packages')"
        :description="error.notificationDescription"
        class="my-6"
        kind="error"
        ><template #details v-if="error.notificationDetails">
          {{ error.notificationDetails }}
        </template></NeInlineNotification
      ></template
    >
    <template v-else>
      <p>
        {{ t('standalone.update.update_in_progress_message') }}
      </p>
      <NeProgressBar class="mt-4" :progress="currentProgress" />
    </template>
  </NeModal>
</template>