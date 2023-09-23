<script lang="ts" setup>
import { OrgUserRoles } from 'nocodb-sdk'
import type { OrgUserReqType, RequestParams, Roles, UserType } from 'nocodb-sdk'
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

loadUsers()

const updateRole = async (userId: string, roles: Roles) => {
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

const deleteUser = async (userId: string) => {
  try {
    await api.orgUsers.delete(userId)
    message.success(t('msg.success.userDeleted'))
    await loadUsers()
    $e('a:org-user:user-deleted')
  } catch (e: any) {
    message.error(await extractSdkResponseErrorMsg(e))
  }
  // closing the modal
  isOpen.value = false
}

const resendInvite = async (user: User) => {
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

const copyPasswordResetUrl = async (user: User) => {
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
          <NcButton
            data-testid="nc-super-user-invite"
            size="small"
            type="primary"
            @click="
              () => {
                showUserModal = true
                userMadalKey++
              }
            "
          >
            <div class="flex items-center gap-1">
              <component :is="iconMap.plus" />
              Invite new user
            </div>
          </NcButton>
        </div>
      </div>
      <div class="w-[780px] mt-5 border-1 rounded-md h-[520px]">
        <div>
          <div class="flex w-full pl-5 bg-gray-50 border-b-1">
            <span class="py-3.5 text-gray-500 font-medium text-3.5 w-1/4 text-start">{{ $t('labels.email') }}</span>
            <span class="py-3.5 text-gray-500 font-medium text-3.5 w-2/4 text-start">{{ $t('objects.role') }}</span>
            <span class="py-3.5 pl-19 text-gray-500 font-medium text-1/4 w-2/9 text-start">Actions</span>
          </div>
        </div>
      </div>

      <LazyAccountUsersModal :key="userMadalKey" :show="showUserModal" @closed="showUserModal = false" @reload="loadUsers" />
    </div>
  </div>
</template>
