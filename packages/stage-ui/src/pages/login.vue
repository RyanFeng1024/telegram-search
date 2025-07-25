<script setup lang="ts">
import { useAuthStore, useWebsocketStore } from '@tg-search/client'
import { storeToRefs } from 'pinia'
import { ref, watch } from 'vue'
import { useRouter } from 'vue-router'
import { toast } from 'vue-sonner'

import Stepper from '../components/ui/Stepper.vue'

type LoginStep = 'phone' | 'code' | 'password' | 'complete'

const router = useRouter()

const authStore = useAuthStore()
const websocketStore = useWebsocketStore()
const { isLoggedIn } = storeToRefs(authStore)

const state = ref({
  currentStep: 'phone' as LoginStep,
  showAdvancedSettings: false,
  phoneNumber: websocketStore.getActiveSession()?.phoneNumber ?? '',
  verificationCode: '',
  twoFactorPassword: '',
})
authStore.auth.needCode = false
authStore.auth.needPassword = false
authStore.auth.isLoading = false

const {
  login,
  submitCode,
  submitPassword,
} = authStore.handleAuth()

watch(() => authStore.auth.needCode, (value) => {
  if (value) {
    authStore.auth.isLoading = false
    state.value.currentStep = 'code'
  }
})

watch(() => authStore.auth.needPassword, (value) => {
  if (value) {
    authStore.auth.isLoading = false
    state.value.currentStep = 'password'
  }
})

watch(isLoggedIn, (value) => {
  if (value) {
    authStore.auth.isLoading = false
    state.value.currentStep = 'complete'
  }
})

const steps = [
  { step: 1, value: 'phone', title: '手机号', description: '输入您的 Telegram 手机号' },
  { step: 2, value: 'code', title: '验证码', description: '输入 Telegram 发送的验证码' },
  { step: 3, value: 'password', title: '二次验证', description: '输入两步验证密码' },
  { step: 4, value: 'complete', title: '完成', description: '登录成功' },
]

function redirectRoot() {
  toast.success('登录成功')
  router.push('/')
}

async function handleLogin() {
  authStore.auth.isLoading = true

  try {
    switch (state.value.currentStep) {
      case 'phone':
        login(state.value.phoneNumber)
        break
      case 'code':
        submitCode(state.value.verificationCode)
        break
      case 'password':
        submitPassword(state.value.twoFactorPassword)
        break
    }
  }
  catch (error) {
    toast.error(error instanceof Error ? error.message : String(error))
  }
}
</script>

<template>
  <div class="min-h-screen flex items-center justify-center bg-background dark:bg-gray-900">
    <div class="max-w-md w-full rounded-2xl bg-card p-10 shadow-2xl dark:bg-gray-800">
      <h1 class="mb-6 text-center text-3xl text-gray-900 font-bold tracking-tight dark:text-gray-100">
        Telegram 登录
      </h1>
      <Stepper :steps="steps" :current-step="state.currentStep" />
      <p class="mb-8 text-center text-lg text-gray-600 font-medium dark:text-gray-400">
        {{ steps.find(s => s.value === state.currentStep)?.description }}
      </p>

      <!-- 手机号码表单 -->
      <form v-if="state.currentStep === 'phone'" class="space-y-6" @submit.prevent="handleLogin">
        <div>
          <label for="phoneNumber" class="mb-2 block text-base text-gray-900 font-semibold dark:text-gray-100">手机号码</label>
          <input
            id="phoneNumber"
            v-model="state.phoneNumber"
            type="tel"
            placeholder="+86 123 4567 8901"
            class="w-full border border-neutral-200 rounded-xl bg-neutral-100 px-5 py-4 text-xl text-gray-900 transition disabled:cursor-not-allowed dark:border-gray-600 dark:bg-gray-700 dark:text-gray-100 focus:outline-none focus:ring-2 focus:ring-primary dark:focus:ring-offset-gray-800"
            required
            :disabled="authStore.auth.isLoading"
          >
        </div>
        <button
          type="submit"
          class="w-full flex items-center justify-center rounded-xl bg-primary py-4 text-lg text-white font-bold transition disabled:cursor-not-allowed disabled:bg-gray-300 hover:bg-primary/90 dark:disabled:bg-gray-700"
          :disabled="authStore.auth.isLoading"
        >
          <span v-if="authStore.auth.isLoading" class="i-lucide-loader-2 mr-2 animate-spin" />
          {{ authStore.auth.isLoading ? '处理中...' : '登录' }}
        </button>
      </form>

      <!-- 验证码表单 -->
      <form v-if="state.currentStep === 'code'" class="space-y-6" @submit.prevent="handleLogin">
        <div>
          <label for="verificationCode" class="mb-2 block text-base text-gray-900 font-semibold dark:text-gray-100">验证码</label>
          <input
            id="verificationCode"
            v-model="state.verificationCode"
            type="text"
            placeholder="请输入 Telegram 发送的验证码"
            class="w-full border border-neutral-200 rounded-xl bg-neutral-100 px-5 py-4 text-xl text-gray-900 transition disabled:cursor-not-allowed dark:border-gray-600 dark:bg-gray-700 dark:text-gray-100 focus:outline-none focus:ring-2 focus:ring-primary dark:focus:ring-offset-gray-800"
            required
            :disabled="authStore.auth.isLoading"
          >
          <p class="mt-2 text-sm text-gray-600 dark:text-gray-400">
            请检查您的 Telegram 应用或短信
          </p>
        </div>
        <button
          type="submit"
          class="w-full flex items-center justify-center rounded-xl bg-primary py-4 text-lg text-white font-bold transition disabled:cursor-not-allowed disabled:bg-gray-300 hover:bg-primary/90 dark:disabled:bg-gray-700"
          :disabled="authStore.auth.isLoading"
        >
          <span v-if="authStore.auth.isLoading" class="i-lucide-loader-2 mr-2 animate-spin" />
          {{ authStore.auth.isLoading ? '处理中...' : '验证' }}
        </button>
      </form>

      <!-- 两步验证密码表单 -->
      <form v-if="state.currentStep === 'password'" class="space-y-6" @submit.prevent="handleLogin">
        <div>
          <label for="twoFactorPassword" class="mb-2 block text-base text-gray-900 font-semibold dark:text-gray-100">两步验证密码</label>
          <input
            id="twoFactorPassword"
            v-model="state.twoFactorPassword"
            type="password"
            placeholder="请输入您的两步验证密码"
            class="w-full border border-neutral-200 rounded-xl bg-neutral-100 px-5 py-4 text-xl text-gray-900 transition disabled:cursor-not-allowed dark:border-gray-600 dark:bg-gray-700 dark:text-gray-100 focus:outline-none focus:ring-2 focus:ring-primary dark:focus:ring-offset-gray-800"
            required
            :disabled="authStore.auth.isLoading"
          >
        </div>
        <button
          type="submit"
          class="w-full flex items-center justify-center rounded-xl bg-primary py-4 text-lg text-white font-bold transition disabled:cursor-not-allowed disabled:bg-gray-300 hover:bg-primary/90 dark:disabled:bg-gray-700"
          :disabled="authStore.auth.isLoading"
        >
          <span v-if="authStore.auth.isLoading" class="i-lucide-loader-2 mr-2 animate-spin" />
          {{ authStore.auth.isLoading ? '处理中...' : '登录' }}
        </button>
      </form>

      <!-- 登录完成 -->
      <div v-if="state.currentStep === 'complete'" class="text-center">
        <div class="mb-4 text-3xl">
          🎉
        </div>
        <h2 class="text-xl text-gray-900 font-bold dark:text-gray-100">
          登录成功！
        </h2>
        <p class="mt-2 text-lg text-gray-600 dark:text-gray-400">
          您已成功登录 Telegram 账号
        </p>
        <button
          class="mt-6 w-full rounded-xl bg-primary py-4 text-lg text-white font-bold transition hover:bg-primary/90"
          @click="redirectRoot"
        >
          进入主页
        </button>
      </div>
    </div>
  </div>
</template>
