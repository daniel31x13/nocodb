<script lang="ts" setup>
import { OrgUserRoles } from 'nocodb-sdk'
import type { OrgUserReqType, RequestParams, UserType } from 'nocodb-sdk'
import type { User } from '#imports'
import { extractSdkResponseErrorMsg, iconMap, useApi, useCopy, useDashboard, useNuxtApp } from '#imports'

const { api, isLoading } = useApi()

const { $e } = useNuxtApp()

const { t } = useI18n()

const { dashboardUrl } = useDashboard()

const { copy } = useCopy()

const users = ref<UserType[]>([])

const currentPage = ref(1)

const currentLimit = ref(10)

const showUserModal = ref(false)

const userMadalKey = ref(0)

const isOpen = ref(false)

const searchText = ref<string>('')

const pagination = reactive({
  total: 0,
  pageSize: 10,
  position: ['bottomCenter'],
})

const loadUsers = async (page = currentPage.value, limit = currentLimit.value) => {
  currentPage.value = page
  try {
    const response: any = await api.orgUsers.list({
      query: {
        limit,
        offset: searchText.value.length === 0 ? (page - 1) * limit : 0,
        query: searchText.value,
      },
    } as RequestParams)

    if (!response) return

    pagination.total = response.pageInfo.totalRows ?? 0

    pagination.pageSize = 10

    users.value = response.list as UserType[]
  } catch (e: any) {
    message.error(await extractSdkResponseErrorMsg(e))
  }
}

onMounted(() => {
  loadUsers()
})

const updateRole = async (userId: string, roles: string) => {
  try {
    await api.orgUsers.update(userId, {
      roles,
    } as OrgUserReqType)
    message.success(t('msg.success.roleUpdated'))

    $e('a:org-user:role-updated', { role: roles })
  } catch (e: any) {
    message.error(await extractSdkResponseErrorMsg(e))
  }
}

const deleteModalInfo = ref<UserType | null>(null)

const deleteUser = async () => {
  try {
    await api.orgUsers.delete(deleteModalInfo.value?.id as string)
    message.success(t('msg.success.userDeleted'))
    await loadUsers()
    $e('a:org-user:user-deleted')
  } catch (e: any) {
    message.error(await extractSdkResponseErrorMsg(e))
  } finally {
    // closing the modal
    isOpen.value = false
    deleteModalInfo.value = null
  }
}

const resendInvite = async (user: UserType) => {
  try {
    await api.orgUsers.resendInvite(user.id)

    // Invite email sent successfully
    message.success(t('msg.success.inviteEmailSent'))
    await loadUsers()
  } catch (e: any) {
    message.error(await extractSdkResponseErrorMsg(e))
  }

  $e('a:org-user:resend-invite')
}

const copyInviteUrl = async (user: User) => {
  if (!user.invite_token) return
  try {
    await copy(`${dashboardUrl.value}#/signup/${user.invite_token}`)

    // Invite URL copied to clipboard
    message.success(t('msg.success.inviteURLCopied'))
  } catch (e: any) {
    message.error(e.message)
  }
  $e('c:user:copy-url')
}

const copyPasswordResetUrl = async (user: UserType) => {
  try {
    const { reset_password_url } = await api.orgUsers.generatePasswordResetToken(user.id)

    await copy(reset_password_url!)

    // Invite URL copied to clipboard
    message.success(t('msg.success.passwordResetURLCopied'))
    $e('c:user:copy-url')
  } catch (e: any) {
    message.error(await extractSdkResponseErrorMsg(e))
  }
}

const openInviteModal = () => {
  showUserModal.value = true
  userMadalKey.value++
}

const openDeleteModal = (user: UserType) => {
  deleteModalInfo.value = user
  isOpen.value = true
}
</script>

<template>
  <div data-testid="nc-super-user-list">
    <div class="max-w-195 mx-auto">
      <div class="text-xl my-4 text-left font-weight-bold">User Management</div>
      <div class="py-2 flex gap-4 items-center justify-between">
        <a-input
          v-model:value="searchText"
          class="!max-w-90 !rounded-md"
          placeholder="Search members"
          @blur="() => loadUsers"
          @keydown.enter="() => loadUsers"
        >
          <template #prefix>
            <PhMagnifyingGlassBold class="!h-3.5 text-gray-500" />
          </template>
        </a-input>
        <div class="flex gap-3 items-center justify-center">
          <component :is="iconMap.reload" class="cursor-pointer" @click="() => loadUsers" />
          <NcButton data-testid="nc-super-user-invite" size="small" type="primary" @click="openInviteModal">
            <div class="flex items-center gap-1">
              <component :is="iconMap.plus" />
              Invite new user
            </div>
          </NcButton>
        </div>
      </div>
      <div class="w-[780px] mt-5 border-1 rounded-md h-[520px]">
        <div class="flex w-full bg-gray-50 border-b-1">
          <span class="py-3.5 text-gray-500 font-medium text-3.5 w-1/3 text-start pl-10">{{ $t('labels.email') }}</span>
          <span class="py-3.5 text-gray-500 font-medium text-3.5 w-1/3 text-start pl-20">{{ $t('objects.role') }}</span>
          <span class="py-3.5 text-gray-500 font-medium text-3.5 w-1/3 text-end pl-30">Actions</span>
        </div>
        <!-- if users are empty -->
        <div v-if="!users.length" class="">
          <a-empty :image="Empty.PRESENTED_IMAGE_SIMPLE" :description="$t('labels.noData')" />
        </div>

        <div v-for="el of users" v-else :key="el.id" data-testid="nc-token-list" class="flex border-b-1 py-3 justify-around px-5">
          <span class="text-3.5 text-start w-1/3 pl-5">
            {{ el.email }}
          </span>
          <span class="text-3.5 text-start w-1/3 pl-18">
            <div v-if="el?.roles?.includes('super')" class="font-weight-bold">Super Admin</div>
            <a-select
              v-else
              v-model:value="el.roles"
              class="w-[220px] nc-user-roles"
              :dropdown-match-select-width="false"
              @change="updateRole(el.id, el.roles as string)"
            >
              <a-select-option
                class="nc-users-list-role-option"
                :value="OrgUserRoles.CREATOR"
                :label="$t(`objects.roleType.orgLevelCreator`)"
              >
                <div>{{ $t(`objects.roleType.orgLevelCreator`) }}</div>
                <span class="text-gray-500 text-xs whitespace-normal">
                  {{ $t('msg.info.roles.orgCreator') }}
                </span>
              </a-select-option>

              <a-select-option
                class="nc-users-list-role-option"
                :value="OrgUserRoles.VIEWER"
                :label="$t(`objects.roleType.orgLevelViewer`)"
              >
                <div>{{ $t(`objects.roleType.orgLevelViewer`) }}</div>
                <span class="text-gray-500 text-xs whitespace-normal">
                  {{ $t('msg.info.roles.orgViewer') }}
                </span>
              </a-select-option>
            </a-select>
          </span>
          <span class="w-1/3 pl-35">
            <div class="flex items-center gap-2">
              <NcDropDown class="flex" placement="bottomRight" overlay-class-name="nc-dropdown-user-mgmt">
                <template #overlay>
                  <a-menu>
                    <a-menu-item>
                      <div class="flex flex-row items-center py-3" @click="copyPasswordResetUrl(el)">
                        <component :is="iconMap.copy" class="flex h-[1rem] text-gray-500" />
                        <div class="text-xs pl-2">{{ $t('activity.copyPasswordResetURL') }}</div>
                      </div>
                    </a-menu-item>
                    <a-menu-item>
                      <div class="flex flex-row items-center py-3" @click="isOpen = true">
                        <!-- Delete user modal -->

                        <component :is="iconMap.delete" data-testid="nc-super-user-delete" class="flex h-[1rem] text-gray-500" />
                        <div class="text-xs pl-2">{{ $t('general.delete') }}</div>
                      </div>
                    </a-menu-item>
                  </a-menu>
                </template>
              </NcDropDown>

              <NcDropdown :trigger="['click']">
                <MdiDotsVertical
                  class="border-1 !text-gray-600 h-5.5 w-5.5 rounded outline-0 p-0.5 nc-workspace-menu transform transition-transform !text-gray-400 cursor-pointer hover:(!text-gray-500 bg-gray-100)"
                />
                <template #overlay>
                  <NcMenu>
                    <template v-if="!el.roles?.includes('super')">
                      <!-- Resend invite Email -->
                      <NcMenuItem @click="resendInvite(el)">
                        <component :is="iconMap.email" class="flex h-[1rem] text-gray-500" />
                        <div class="text-sm pl-2">{{ $t('activity.resendInvite') }}</div>
                      </NcMenuItem>
                      <NcMenuItem @click="copyInviteUrl(el)">
                        <component :is="iconMap.copy" class="flex h-[1rem] text-gray-500" />
                        <div class="text-sm pl-2">{{ $t('activity.copyInviteURL') }}</div>
                      </NcMenuItem>
                    </template>
                    <NcDivider v-if="!el.roles?.includes('super')" />
                    <NcMenuItem class="!text-red-500 !hover:bg-red-50" @click="openDeleteModal(el)">
                      <MaterialSymbolsDeleteOutlineRounded />
                      Remove user
                    </NcMenuItem>
                  </NcMenu>
                </template>
              </NcDropdown>
            </div>
          </span>
        </div>
      </div>

      <GeneralDeleteModal v-model:visible="isOpen" entity-name="User" :on-delete="() => deleteUser()">
        <template #entity-preview>
          <span>
            <div class="flex flex-row items-center py-2.25 px-2.5 bg-gray-50 rounded-lg text-gray-700 mb-4">
              <GeneralIcon icon="account" class="nc-view-icon"></GeneralIcon>
              <div
                class="capitalize text-ellipsis overflow-hidden select-none w-full pl-1.75"
                :style="{ wordBreak: 'keep-all', whiteSpace: 'nowrap', display: 'inline' }"
              >
                {{ deleteModalInfo?.email }}
              </div>
            </div>
          </span>
        </template>
      </GeneralDeleteModal>

      <LazyAccountUsersModal :key="userMadalKey" :show="showUserModal" @closed="showUserModal = false" @reload="loadUsers" />
    </div>
  </div>
</template>
