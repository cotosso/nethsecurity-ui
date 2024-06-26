<!--
  Copyright (C) 2024 Nethesis S.r.l.
  SPDX-License-Identifier: GPL-3.0-or-later
-->

<script setup lang="ts">
import { NeCard, NeLink, NeTitle } from '@nethesis/vue-components'
import { useI18n } from 'vue-i18n'
import RealTimeTrafficCard from '@/components/standalone/dashboard/RealTimeTrafficCard.vue'
import SystemInfoCard from '@/components/standalone/dashboard/SystemInfoCard.vue'
import ServiceCard from '@/components/standalone/dashboard/ServiceCard.vue'
import TrafficSummaryChart from '@/components/standalone/dashboard/TrafficSummaryChart.vue'
import WanTrafficCard from '@/components/standalone/dashboard/WanTrafficCard.vue'
import { useTrafficSummary } from '@/composables/useTrafficSummary'
import { getStandaloneRoutePrefix } from '@/lib/router'
import router from '@/router'
import { useRoute } from 'vue-router'

const { t } = useI18n()
const route = useRoute()
const {
  clientsLabels,
  clientsDatasets,
  protocolsLabels,
  protocolsDatasets,
  appsLabels,
  appsDatasets,
  loadingTrafficSummary,
  errorTitle,
  errorDescription
} = useTrafficSummary()

function goTo(path: string) {
  router.push(`${getStandaloneRoutePrefix(route)}${path}`)
}
</script>

<template>
  <div class="flex flex-col justify-between md:flex-row md:items-center">
    <NeTitle>{{ t('standalone.dashboard.title') }}</NeTitle>
    <div class="mb-6 text-sm text-gray-500 dark:text-gray-400">
      {{ t('standalone.dashboard.data_updated_every_seconds', { seconds: 10 }) }}
    </div>
  </div>

  <div class="grid grid-cols-1 gap-x-6 gap-y-6 sm:grid-cols-2 xl:grid-cols-4 3xl:grid-cols-6">
    <!-- system -->
    <SystemInfoCard class="sm:col-span-2 xl:row-span-2" />
    <!-- internet connection -->
    <ServiceCard
      serviceName="internet"
      hasStatus
      :title="t('standalone.dashboard.internet_connection')"
      :icon="['fas', 'earth-americas']"
    />
    <!-- multiwan -->
    <ServiceCard serviceName="mwan" hasStatus :icon="['fas', 'earth-americas']">
      <template #title>
        <NeLink @click="goTo('/network/multi-wan')">
          {{ t('standalone.dashboard.multiwan') }}
        </NeLink>
      </template>
    </ServiceCard>
    <!-- dpi-core -->
    <ServiceCard
      serviceName="netifyd"
      hasStatus
      :title="t('standalone.dashboard.dpi_core')"
      :icon="['fas', 'bolt']"
    />
    <!-- openvpn rw -->
    <ServiceCard serviceName="openvpn_rw" hasStatus :icon="['fas', 'globe']">
      <template #title>
        <NeLink @click="goTo('/vpn/openvpn-rw')">
          {{ t('standalone.dashboard.openvpn_rw') }}
        </NeLink>
      </template>
    </ServiceCard>
    <!-- hotspot -->
    <ServiceCard serviceName="dedalo" hasStatus :icon="['fas', 'wifi']">
      <template #title>
        <NeLink @click="goTo('/network/hotspot')">
          {{ t('standalone.dashboard.hotspot') }}
        </NeLink>
      </template>
    </ServiceCard>
    <!-- threat shield / banIP -->
    <ServiceCard serviceName="banip" hasStatus :icon="['fas', 'shield']">
      <template #title>
        <NeLink @click="goTo('/security/threat-shield')">
          {{ t('standalone.dashboard.thread_shield_ip') }}
        </NeLink>
      </template>
    </ServiceCard>
    <!-- known hosts -->
    <ServiceCard
      serviceName="hosts"
      hasCounter
      :title="t('standalone.dashboard.known_hosts')"
      :icon="['fas', 'circle-info']"
    />
    <!-- column spacer (only from 3xl screen) -->
    <div class="col-span-1 hidden 3xl:block"></div>
    <WanTrafficCard class="sm:col-span-2 3xl:col-span-3" />
    <!-- realtime traffic -->
    <RealTimeTrafficCard class="sm:col-span-2 3xl:col-span-3" />
    <!-- top hosts -->
    <NeCard
      :title="t('standalone.dashboard.today_top_hosts')"
      :description="
        clientsDatasets[0]?.data.length
          ? t('standalone.dashboard.today_top_hosts_description')
          : t('standalone.dashboard.no_hosts')
      "
      :skeletonLines="6"
      :loading="loadingTrafficSummary"
      :errorTitle="errorTitle"
      :errorDescription="errorDescription"
      class="sm:col-span-2 xl:row-span-2"
    >
      <TrafficSummaryChart
        v-if="clientsDatasets[0]?.data.length"
        :labels="clientsLabels"
        :datasets="clientsDatasets"
      />
    </NeCard>
    <!-- top applications -->
    <NeCard
      :title="t('standalone.dashboard.today_top_applications')"
      :description="
        appsDatasets[0]?.data.length
          ? t('standalone.dashboard.today_top_applications_description')
          : t('standalone.dashboard.no_applications')
      "
      :skeletonLines="6"
      :loading="loadingTrafficSummary"
      :errorTitle="errorTitle"
      :errorDescription="errorDescription"
      class="sm:col-span-2 xl:row-span-2"
    >
      <TrafficSummaryChart
        v-if="appsDatasets[0]?.data.length"
        :labels="appsLabels"
        :datasets="appsDatasets"
      />
    </NeCard>
    <!-- top protocols -->
    <NeCard
      :title="t('standalone.dashboard.today_top_protocols')"
      :description="
        protocolsDatasets[0]?.data.length
          ? t('standalone.dashboard.today_top_protocols_description')
          : t('standalone.dashboard.no_protocols')
      "
      :skeletonLines="6"
      :loading="loadingTrafficSummary"
      :errorTitle="errorTitle"
      :errorDescription="errorDescription"
      class="sm:col-span-2 xl:row-span-2"
    >
      <TrafficSummaryChart
        v-if="protocolsDatasets[0]?.data.length"
        :labels="protocolsLabels"
        :datasets="protocolsDatasets"
      />
    </NeCard>
  </div>
</template>
