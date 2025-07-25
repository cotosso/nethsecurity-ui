<!--
  Copyright (C) 2024 Nethesis S.r.l.
  SPDX-License-Identifier: GPL-3.0-or-later
-->

<script setup lang="ts">
import { type ControllerAccount, useAccountsStore } from '@/stores/controller/accounts'
import {
  NeButton,
  type NeComboboxOption,
  NeEmptyState,
  NeHeading,
  NeInlineNotification,
  NeSkeleton,
  NeTextInput
} from '@nethesis/vue-components'
import { computed, onMounted, ref } from 'vue'
import { useI18n } from 'vue-i18n'
import UsersTable from '@/components/controller/users/UsersTable.vue'
import CreateOrEditUserDrawer from '@/components/controller/users/CreateOrEditUserDrawer.vue'
import DeleteUserModal from '@/components/controller/users/DeleteUserModal.vue'
import { useNotificationsStore } from '@/stores/notifications'
import { useUnitGroupsStore } from '@/stores/controller/unit_groups'

const accountsStore = useAccountsStore()
const notificationsStore = useNotificationsStore()
const unitGroupsStore = useUnitGroupsStore()

const { t } = useI18n()

const filter = ref('')
const selectedItem = ref<ControllerAccount>()
const showCreateEditDrawer = ref(false)
const showDeleteModal = ref(false)

const filteredAccounts = computed(() =>
  accountsStore.accounts.filter(
    (account) =>
      account.display_name.includes(filter.value) || account.username.includes(filter.value)
  )
)

const unitGroupsOptions = computed<NeComboboxOption[]>(() =>
  unitGroupsStore.unitGroups.map((group): NeComboboxOption => {
    return {
      id: String(group.id),
      label: group.name,
      description: group.description || ''
    }
  })
)

const unitGroupNameMap = computed(() => {
  const map: Record<string, string> = {}
  if (!Array.isArray(unitGroupsStore.unitGroups)) {
    return map
  }
  unitGroupsStore.unitGroups.forEach((group) => {
    map[group.id] = group.name
  })
  return map
})

function openCreateEditDrawer(itemToEdit?: ControllerAccount) {
  selectedItem.value = itemToEdit
  showCreateEditDrawer.value = true
}

function openDeleteModal(itemToEdit?: ControllerAccount) {
  selectedItem.value = itemToEdit
  showDeleteModal.value = true
}

onMounted(() => {
  accountsStore.loadAccounts()
  unitGroupsStore.loadUnitGroups()
})
</script>

<template>
  <NeHeading tag="h3" class="mb-7">{{ t('controller.users.title') }}</NeHeading>
  <div class="flex flex-col gap-y-6">
    <div class="flex flex-row items-center justify-between">
      <p class="max-w-2xl text-sm font-normal text-gray-500 dark:text-gray-400">
        {{ t('controller.users.description') }}
      </p>
      <div class="ml-2 shrink-0">
        <NeButton
          v-if="accountsStore.accounts.length > 0"
          kind="secondary"
          @click="openCreateEditDrawer()"
        >
          <template #prefix>
            <font-awesome-icon :icon="['fas', 'circle-plus']" class="h-4 w-4" aria-hidden="true" />
          </template>
          {{ t('controller.users.add_user') }}
        </NeButton>
      </div>
    </div>
    <!-- text filter -->
    <div class="flex items-center gap-4">
      <NeTextInput v-model="filter" class="max-w-xs" :placeholder="t('common.filter')" />
      <NeButton
        kind="tertiary"
        :disabled="accountsStore.listAccountsLoading || !filter"
        @click="filter = ''"
      >
        {{ t('common.clear_filter') }}
      </NeButton>
    </div>
    <NeInlineNotification
      v-if="accountsStore.listAccountsError.notificationDescription"
      kind="error"
      :title="t('error.cannot_retrieve_users')"
      :description="accountsStore.listAccountsError.notificationDescription"
    >
      <template v-if="accountsStore.listAccountsError.notificationDetails" #details>
        {{ accountsStore.listAccountsError.notificationDetails }}
      </template>
    </NeInlineNotification>
    <NeSkeleton v-if="accountsStore.listAccountsLoading" :lines="7" size="lg" />
    <template v-else>
      <NeEmptyState
        v-if="accountsStore.accounts.length == 0"
        :title="t('controller.users.no_users_found')"
        :icon="['fas', 'user-group']"
      >
        <NeButton kind="primary" @click="openCreateEditDrawer()"
          ><template #prefix>
            <font-awesome-icon
              :icon="['fas', 'circle-plus']"
              class="h-4 w-4"
              aria-hidden="true"
            /> </template
          >{{ t('controller.users.add_user') }}</NeButton
        >
      </NeEmptyState>
      <NeEmptyState
        v-else-if="filteredAccounts.length == 0"
        :title="t('controller.users.no_users_found')"
        :icon="['fas', 'user-group']"
      />
      <UsersTable
        v-else
        :users="filteredAccounts"
        :unit-groups-name-map="unitGroupNameMap"
        @edit="openCreateEditDrawer"
        @delete="openDeleteModal"
      />
    </template>
  </div>
  <CreateOrEditUserDrawer
    :is-shown="showCreateEditDrawer"
    :item-to-edit="selectedItem"
    :unit-groups-options="unitGroupsOptions"
    @add-user="
      () => {
        accountsStore.loadAccounts()
        notificationsStore.addNotification({
          kind: 'success',
          id: 'add-user',
          title: t('controller.users.user_added')
        })
      }
    "
    @edit-user="
      () => {
        accountsStore.loadAccounts()
        notificationsStore.addNotification({
          kind: 'success',
          id: 'edit-user',
          title: t('controller.users.user_edited')
        })
      }
    "
    @close="showCreateEditDrawer = false"
  />
  <DeleteUserModal
    :visible="showDeleteModal"
    :item-to-delete="selectedItem"
    @close="showDeleteModal = false"
    @user-deleted="
      () => {
        accountsStore.loadAccounts()
        notificationsStore.addNotification({
          kind: 'success',
          id: 'delete-user',
          title: t('controller.users.user_deleted')
        })
      }
    "
  />
</template>
