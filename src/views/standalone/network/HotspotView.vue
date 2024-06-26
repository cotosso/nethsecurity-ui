<!--
  Copyright (C) 2024 Nethesis S.r.l.
  SPDX-License-Identifier: GPL-3.0-or-later
-->

<script lang="ts" setup>
import { NeTitle, NeTabs } from '@nethesis/vue-components'
import { useI18n } from 'vue-i18n'
import { onMounted, ref, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import StatusContent from '@/components/standalone/hotspot/StatusContent.vue'
import SettingsContent from '@/components/standalone/hotspot/SettingsContent.vue'

const { t } = useI18n()
const route = useRoute()
const router = useRouter()

// tabs management
const tabs = ref([
  {
    name: 'tab-status',
    label: t('standalone.hotspot.tabs.status')
  },
  {
    name: 'tab-settings',
    label: t('standalone.hotspot.tabs.settings')
  }
])
const selectedTab = ref('')

onMounted(() => {
  selectedTab.value = (route.query.tab as string) ?? tabs.value[0].name
})

watch(selectedTab, () => {
  router.push({ path: route.path, query: { tab: selectedTab.value } })
})
</script>

<template>
  <NeTitle>{{ t('standalone.hotspot.title') }}</NeTitle>

  <NeTabs
    :selected="selectedTab"
    :srSelectTabLabel="t('ne_tabs.select_a_tab')"
    :srTabsLabel="t('ne_tabs.tabs')"
    :tabs="tabs"
    class="mb-8"
    @selectTab="selectedTab = $event"
  />
  <StatusContent v-if="selectedTab == 'tab-status'" @go-to-setting="selectedTab = 'tab-settings'" />
  <SettingsContent v-if="selectedTab == 'tab-settings'" />
</template>
