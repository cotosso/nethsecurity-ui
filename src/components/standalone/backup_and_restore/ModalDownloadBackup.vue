<!--
  Copyright (C) 2024 Nethesis S.r.l.
  SPDX-License-Identifier: GPL-3.0-or-later
-->

<script setup lang="ts">
import { ref } from 'vue'
import { useI18n } from 'vue-i18n'
import { ubusCall } from '@/lib/standalone/ubus'
import { NeInlineNotification, getAxiosErrorMessage } from '@nethesis/vue-components'
import { NeModal } from '@nethesis/vue-components'
import { downloadFile } from '@/lib/standalone/fileUpload'
import { deleteFile } from '@/lib/standalone/fileUpload'

const { t } = useI18n()
const props = defineProps({
  showDownloadModal: {
    type: Boolean,
    required: true
  },
  isSetPassphrase: {
    type: Boolean,
    required: true
  },
  seletedBackup: {
    type: String,
    required: true
  },
  unitName: {
    type: String
  }
})

const emit = defineEmits(['success', 'close'])

let loadingDownload = ref(false)

let objNotification = {
  notificationTitle: '',
  notificationDescription: ''
}

let errorDownloadBackup = ref({ ...objNotification })

async function downloadBackup() {
  try {
    loadingDownload.value = true

    let payload = {}
    let methodCall = 'backup'
    if (props.seletedBackup) {
      methodCall = 'registered-download-backup'
      payload = {
        file: props.seletedBackup
      }
    }

    let res = await ubusCall('ns.backup', methodCall, payload)
    if (res?.data?.backup) {
      const file = await downloadFile(res.data.backup)
      const fileURL = URL.createObjectURL(file)

      let extension = '.tar.gz'
      if (props.isSetPassphrase) {
        extension += '.gpg'
      }
      let link = document.createElement('a')
      link.href = fileURL
      link.download = 'backup' + extension
      link.click()

      await deleteFile(res.data.backup)

      emit('close')
    }
  } catch (exception: any) {
    errorDownloadBackup.value.notificationTitle = t('error.cannot_download_backup')
    errorDownloadBackup.value.notificationDescription = t(getAxiosErrorMessage(exception))
  } finally {
    loadingDownload.value = false
  }
}
</script>

<template>
  <NeModal
    :primary-button-disabled="loadingDownload"
    :primary-button-loading="loadingDownload"
    :primary-label="t('standalone.backup_and_restore.backup.download')"
    :secondary-button-disabled="loadingDownload"
    :title="t('standalone.backup_and_restore.backup.modal_download_title')"
    :visible="showDownloadModal"
    kind="info"
    primary-button-kind="primary"
    :close-aria-label="t('common.close')"
    @close="$emit('close')"
    @primary-click="downloadBackup()"
  >
    <div>
      {{ t('standalone.backup_and_restore.backup.modal_download_description', { name: unitName }) }}
    </div>
    <NeInlineNotification
      v-if="errorDownloadBackup.notificationTitle"
      class="my-4"
      kind="error"
      :title="errorDownloadBackup.notificationTitle"
      :description="errorDownloadBackup.notificationDescription"
    />
  </NeModal>
</template>
