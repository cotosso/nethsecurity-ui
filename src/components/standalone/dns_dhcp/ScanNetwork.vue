<!--
  Copyright (C) 2024 Nethesis S.r.l.
  SPDX-License-Identifier: GPL-3.0-or-later
-->

<script setup lang="ts">
import {
  NeHeading,
  NeInlineNotification,
  getAxiosErrorMessage,
  NeEmptyState
} from '@nethesis/vue-components'
import { onMounted } from 'vue'
import { ref } from 'vue'
import { useI18n } from 'vue-i18n'
import { useUciPendingChangesStore } from '@/stores/standalone/uciPendingChanges'
import { useFirewallStore } from '@/stores/standalone/firewall'
import { computed } from 'vue'
import { isEmpty } from 'lodash-es'
import ScanInterfaceCard from './ScanInterfaceCard.vue'
import ScanResultsTable from './ScanResultsTable.vue'
import CreateOrEditStaticLeaseDrawer from './CreateOrEditStaticLeaseDrawer.vue'
import CreateOrEditDnsRecordDrawer from './CreateOrEditDnsRecordDrawer.vue'
import { ubusCall } from '@/lib/standalone/ubus'

export interface ScanInterface {
  name: string
  device: string
}

export interface ScanResult {
  ip: string
  mac: string
  hostname?: string
  description?: string
}

const { t } = useI18n()
const uciChangesStore = useUciPendingChangesStore()
const firewallConfig = useFirewallStore()

const scanningInterface = ref<string>('')
const interfaces = ref<ScanInterface[]>([])
const scanResults = ref<ScanResult[]>([])
const scanRequested = ref(false)
const currentScanResult = ref<ScanResult | undefined>(undefined)
const isAddReservationDrawerShown = ref(false)
const isAddDnsRecordDrawerShown = ref(false)

const loading = ref({
  listInterfaces: true,
  scanNetwork: false
})
const error = ref({
  listInterfaces: '',
  listInterfacesDetails: '',
  scanNetwork: '',
  scanNetworkDetails: ''
})

const isLoadingData = computed(() => {
  return loading.value.listInterfaces || firewallConfig.loading
})

onMounted(() => {
  if (firewallConfig.loading || firewallConfig.error) {
    firewallConfig.fetch()
  }
  listInterfaces()
})

async function listInterfaces() {
  loading.value.listInterfaces = true

  try {
    const res = await ubusCall('ns.scan', 'list-interfaces')
    interfaces.value = res.data.interfaces.map((iface: any) => ({
      name: iface.interface,
      device: iface.device
    }))
  } catch (err: any) {
    error.value.listInterfaces = t(getAxiosErrorMessage(err))
    error.value.listInterfacesDetails = err.toString()
  } finally {
    loading.value.listInterfaces = false
  }
}

async function scanNetwork(iface: ScanInterface) {
  error.value.scanNetwork = ''
  error.value.scanNetworkDetails = ''
  scanningInterface.value = iface.name
  scanRequested.value = true
  loading.value.scanNetwork = true

  try {
    const res = await ubusCall('ns.scan', 'scan', { device: iface.device })
    scanResults.value = res.data.hosts.map((host: ScanResult) => ({
      ip: host.ip,
      mac: host.mac,
      hostname: host.hostname !== host.ip ? host.hostname : '',
      description: host.description
    }))
  } catch (err: any) {
    error.value.scanNetwork = t(getAxiosErrorMessage(err))
    error.value.scanNetworkDetails = err.toString()
  } finally {
    loading.value.scanNetwork = false
  }
}

function addIpReservation(scanResult: ScanResult) {
  currentScanResult.value = scanResult
  isAddReservationDrawerShown.value = true
}

function addDnsRecord(scanResult: ScanResult) {
  currentScanResult.value = scanResult
  isAddDnsRecordDrawerShown.value = true
}
</script>

<template>
  <div>
    <div class="space-y-8">
      <p class="max-w-2xl text-gray-500 dark:text-gray-400">
        {{ t('standalone.dns_dhcp.scan_network_description') }}
      </p>
      <!-- list interfaces error -->
      <NeInlineNotification
        v-if="error.listInterfaces"
        kind="error"
        :title="t('error.cannot_retrieve_interfaces')"
        :description="error.listInterfaces"
        class="mb-4"
      >
        <template v-if="error.listInterfacesDetails" #details>
          {{ error.listInterfacesDetails }}
        </template>
      </NeInlineNotification>
      <!-- interfaces -->
      <div>
        <NeHeading tag="h6" class="mb-4">{{ t('standalone.dns_dhcp.interfaces') }}</NeHeading>
        <!-- scan error -->
        <NeInlineNotification
          v-if="error.scanNetwork"
          kind="error"
          :title="t('error.cannot_scan_network')"
          :description="error.scanNetwork"
          class="mb-4"
        >
          <template v-if="error.scanNetworkDetails" #details>
            {{ error.scanNetworkDetails }}
          </template>
        </NeInlineNotification>
        <!-- empty state -->
        <NeEmptyState
          v-if="!isLoadingData && Object.keys(interfaces).length == 0"
          :title="t('standalone.dns_dhcp.no_interface_configured')"
          :icon="['fas', 'circle-info']"
        />
        <div v-else class="grid grid-cols-1 gap-4 md:grid-cols-2 xl:grid-cols-3 3xl:grid-cols-4">
          <!-- interface cards skeleton -->
          <template v-if="isLoadingData">
            <div v-for="index in 8" :key="index" class="flex animate-pulse">
              <div class="h-24 w-full bg-gray-300 sm:rounded-lg dark:bg-gray-700"></div>
            </div>
          </template>
          <!-- interface cards -->
          <template v-else>
            <ScanInterfaceCard
              v-for="(iface, ifaceName) in interfaces"
              :key="ifaceName"
              :iface="iface"
              :scan-button-loading="loading.scanNetwork && scanningInterface === iface.name"
              :scan-button-disabled="loading.scanNetwork && scanningInterface === iface.name"
              @start-scan="scanNetwork"
            />
          </template>
        </div>
      </div>
      <!-- scan results -->
      <div v-if="!isLoadingData">
        <NeHeading tag="h6" class="mb-4">{{ t('standalone.dns_dhcp.scan_results') }}</NeHeading>
        <!-- empty state -->
        <template v-if="!loading.scanNetwork && isEmpty(scanResults)">
          <!-- no results -->
          <NeEmptyState
            v-if="scanRequested"
            :title="
              t('standalone.dns_dhcp.scan_no_results_empty_state', { iface: scanningInterface })
            "
            :icon="['fas', 'circle-info']"
          />
          <!-- scan not requested yet -->
          <NeEmptyState
            v-else
            :title="t('standalone.dns_dhcp.scan_not_requested_empty_state')"
            :icon="['fas', 'network-wired']"
          />
        </template>
        <template v-else>
          <p
            class="z-0 -mb-1 table max-w-md rounded-ss-md rounded-se-md bg-indigo-300 p-2 text-sm dark:bg-indigo-800"
          >
            {{ t('standalone.dns_dhcp.interface_name', { name: scanningInterface }) }}
          </p>
          <ScanResultsTable
            :results="scanResults"
            :loading="loading.scanNetwork"
            @add-ip-reservation="addIpReservation"
            @add-dns-record="addDnsRecord"
          />
        </template>
      </div>
    </div>
    <!-- add reservation drawer -->
    <CreateOrEditStaticLeaseDrawer
      :is-shown="isAddReservationDrawerShown"
      :import-scan-result="currentScanResult"
      @close="isAddReservationDrawerShown = false"
      @add-edit-lease="uciChangesStore.getChanges()"
    />
    <!-- add dns record drawer -->
    <CreateOrEditDnsRecordDrawer
      :is-shown="isAddDnsRecordDrawerShown"
      :import-scan-result="currentScanResult"
      @close="isAddDnsRecordDrawerShown = false"
      @add-edit-record="uciChangesStore.getChanges()"
    />
  </div>
</template>
