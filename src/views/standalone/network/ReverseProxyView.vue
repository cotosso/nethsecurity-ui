<!--
  Copyright (C) 2024 Nethesis S.r.l.
  SPDX-License-Identifier: GPL-3.0-or-later
-->

<script setup lang="ts">
import { useI18n } from 'vue-i18n'
import {
  NeHeading,
  NeButton,
  NeInlineNotification,
  NeSkeleton,
  getAxiosErrorMessage,
  NeEmptyState
} from '@nethesis/vue-components'
import { onMounted, ref } from 'vue'
import { useUciPendingChangesStore } from '@/stores/standalone/uciPendingChanges'
import { ubusCall } from '@/lib/standalone/ubus'
import ProxyTable from '@/components/standalone/reverse_proxy/ProxyTable.vue'
import DeleteProxyModal from '@/components/standalone/reverse_proxy/DeleteProxyModal.vue'
import CreateOrEditProxyDrawer from '@/components/standalone/reverse_proxy/CreateOrEditProxyDrawer.vue'
import { FontAwesomeIcon } from '@fortawesome/vue-fontawesome'
import { faCirclePlus, faServer } from '@fortawesome/free-solid-svg-icons'
import { getStandaloneRoutePrefix } from '@/lib/router.ts'
import { useRouter } from 'vue-router'

export type ReverseProxy = {
  id: string
  type: string
  description: string
  location: string
  destination: string
  allow?: string[]
  certificate?: string
  domain?: string
  path?: string
}

const { t } = useI18n()
const router = useRouter()

const uciChangesStore = useUciPendingChangesStore()

const loading = ref(true)
const proxies = ref<ReverseProxy[]>([])
const selectedProxy = ref<ReverseProxy | null>(null)
const showCreateEditDrawer = ref(false)
const showDeleteModal = ref(false)
const portOpen = ref(false)

const error = ref({
  notificationTitle: '',
  notificationDescription: '',
  notificationDetails: ''
})

async function fetchProxies() {
  try {
    loading.value = true
    const data = await ubusCall('ns.reverseproxy', 'list-proxies')
    proxies.value = data.data.data
    portOpen.value = data.data.port_open
  } catch (err: any) {
    error.value.notificationTitle = t('error.cannot_retrieve_proxies')
    error.value.notificationDescription = t(getAxiosErrorMessage(err))
    error.value.notificationDetails = err.toString()
  } finally {
    loading.value = false
  }
}

function openCreateEditDrawer(itemToEdit: ReverseProxy | null) {
  selectedProxy.value = itemToEdit
  showCreateEditDrawer.value = true
}

function openDeleteModal(itemToDelete: ReverseProxy) {
  selectedProxy.value = itemToDelete
  showDeleteModal.value = true
}

function closeModalsAndDrawers() {
  selectedProxy.value = null
  showDeleteModal.value = false
  showCreateEditDrawer.value = false
}

function cleanError() {
  error.value = {
    notificationTitle: '',
    notificationDescription: '',
    notificationDetails: ''
  }
}

async function reloadProxies() {
  cleanError()
  await fetchProxies()
  await uciChangesStore.getChanges()
}

onMounted(() => {
  fetchProxies()
})
</script>

<template>
  <div class="flex flex-col gap-y-6">
    <NeHeading tag="h3" class="mb-7">{{ t('standalone.reverse_proxy.title') }}</NeHeading>
    <div class="flex flex-row items-center justify-between">
      <p class="max-w-2xl text-sm font-normal text-gray-500 dark:text-gray-400">
        {{ t('standalone.reverse_proxy.page_description') }}
      </p>
      <template v-if="proxies.length > 0">
        <div class="ml-2 flex shrink-0 flex-col gap-x-0 gap-y-2 sm:flex-row sm:gap-x-2 sm:gap-y-0">
          <NeButton kind="secondary" @click="openCreateEditDrawer(null)">
            <template #prefix>
              <FontAwesomeIcon :icon="faCirclePlus" class="h-4 w-4" aria-hidden="true" />
            </template>
            {{ t('standalone.reverse_proxy.add_reverse_proxy') }}
          </NeButton>
        </div>
      </template>
    </div>
    <NeInlineNotification
      v-if="error.notificationTitle"
      kind="error"
      :title="error.notificationTitle"
      :description="error.notificationDescription"
    >
      <template v-if="error.notificationDetails" #details>
        {{ error.notificationDetails }}
      </template>
    </NeInlineNotification>
    <NeSkeleton v-if="loading" :lines="10" />
    <template v-else-if="!error.notificationTitle">
      <NeInlineNotification
        v-if="!portOpen"
        kind="warning"
        :title="t('standalone.reverse_proxy.port_not_open')"
        :description="t('standalone.reverse_proxy.port_not_open_description')"
        :primary-button-label="t('standalone.reverse_proxy.open_firewall_settings')"
        @click="router.push(`${getStandaloneRoutePrefix()}/firewall/rules?tab=inputRules`)"
      />
      <NeEmptyState
        v-if="proxies.length == 0"
        :title="t('standalone.reverse_proxy.no_reverse_proxy_found')"
        :icon="faServer"
      >
        <NeButton kind="primary" @click="openCreateEditDrawer(null)">
          <template #prefix>
            <FontAwesomeIcon :icon="faCirclePlus" class="h-4 w-4" aria-hidden="true" />
          </template>
          {{ t('standalone.reverse_proxy.add_reverse_proxy') }}
        </NeButton>
      </NeEmptyState>
      <ProxyTable
        v-else
        :proxies="proxies"
        @delete="openDeleteModal"
        @edit="openCreateEditDrawer"
      />
    </template>
  </div>
  <DeleteProxyModal
    :visible="showDeleteModal"
    :item-to-delete="selectedProxy"
    @close="closeModalsAndDrawers"
    @proxy-deleted="reloadProxies"
  />
  <CreateOrEditProxyDrawer
    :is-shown="showCreateEditDrawer"
    :item-to-edit="selectedProxy"
    @close="closeModalsAndDrawers"
    @add-edit-proxy="reloadProxies"
  />
</template>
