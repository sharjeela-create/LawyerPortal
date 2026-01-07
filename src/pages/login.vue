<script setup lang="ts">
import { computed, ref } from 'vue'
import { useRoute, useRouter } from 'vue-router'

import { useAuth } from '../composables/useAuth'

const router = useRouter()
const route = useRoute()
const auth = useAuth()

const email = ref('')
const password = ref('')
const errorMessage = ref<string | null>(null)

const redirectTo = computed(() => {
  const fromQuery = route.query.redirect
  return typeof fromQuery === 'string' && fromQuery.length ? fromQuery : '/dashboard'
})

const isBusy = computed(() => auth.state.value.loading)

const handleSubmit = async () => {
  errorMessage.value = null

  try {
    await auth.signInWithPassword(email.value.trim(), password.value)

    const isAdmin = auth.state.value.profile?.role === 'admin'
    const hasCenter = Boolean(auth.state.value.profile?.center_id)

    if (!isAdmin && !hasCenter) {
      await auth.signOut()
      errorMessage.value = 'Your account is not provisioned yet. Please contact an administrator.'
      return
    }

    await router.replace(redirectTo.value)
  } catch (err) {
    const msg = err instanceof Error ? err.message : 'Unable to sign in'
    errorMessage.value = msg
  }
}
</script>

<template>
  <div class="min-h-screen bg-[var(--ap-bg)] text-white">
    <div class="relative isolate min-h-screen overflow-hidden bg-gradient-to-b from-[var(--ap-surface)] via-[var(--ap-surface-soft)] to-[var(--ap-bg)]">
      <div class="absolute inset-0 pointer-events-none">
        <div class="absolute -top-32 left-1/2 h-72 w-72 -translate-x-1/2 rounded-full bg-[var(--ap-accent)]/40 blur-[120px]" />
        <div class="absolute top-16 right-8 h-56 w-56 rounded-full bg-[var(--ap-amber-soft)] blur-[110px]" />
      </div>

      <main class="mx-auto flex min-h-screen w-full max-w-6xl items-center px-6 py-10 lg:px-10">
        <section class="grid w-full gap-10 lg:grid-cols-[1.1fr_0.9fr] lg:items-center">
          <div class="space-y-8">
            <div class="space-y-4">
              <p class="text-xs uppercase tracking-[0.4em] text-white/60">Lawyer Portal</p>
              <h1 class="text-4xl font-semibold leading-tight text-white md:text-5xl">Secure entry for your MVA workspace.</h1>
              <p class="text-base leading-relaxed text-white/70">
                Sign in with your work email to access your dashboard, manage intake workflows, and keep handoffs moving.
              </p>
            </div>

            <div class="grid gap-4 sm:grid-cols-2">
              <div class="rounded-3xl border border-white/10 bg-white/5 p-5">
                <p class="text-xs uppercase tracking-[0.4em] text-white/50">Need an invite?</p>
                <p class="mt-3 text-sm leading-relaxed text-white/70">
                  Contact us and we will provision your workspace.
                </p>
              </div>
              <div class="rounded-3xl border border-white/10 bg-white/5 p-5">
                <p class="text-xs uppercase tracking-[0.4em] text-white/50">Security</p>
                <p class="mt-3 text-sm leading-relaxed text-white/70">
                  We enforce device-based MFA and session timeouts across all orgs.
                </p>
              </div>
            </div>
          </div>

          <div class="rounded-[32px] border border-white/10 bg-white/10 p-8 shadow-2xl shadow-black/50 backdrop-blur-xl">
            <div class="space-y-6">
              <div>
                <p class="text-xs uppercase tracking-[0.4em] text-white/60">Sign in</p>
                <h2 class="mt-2 text-3xl font-semibold text-white">Access your dashboard</h2>
              </div>

              <form class="space-y-4" @submit.prevent="handleSubmit">
                <label class="block space-y-2 text-sm">
                  <span class="text-white/80">Work email</span>
                  <input
                    v-model="email"
                    type="email"
                    placeholder="you@firm.com"
                    autocomplete="email"
                    class="w-full rounded-2xl border border-white/15 bg-white/5 px-4 py-3 text-white placeholder:text-white/40 focus:border-white/40 focus:outline-none"
                  >
                </label>

                <label class="block space-y-2 text-sm">
                  <span class="text-white/80">Password</span>
                  <input
                    v-model="password"
                    type="password"
                    placeholder="Enter password"
                    autocomplete="current-password"
                    class="w-full rounded-2xl border border-white/15 bg-white/5 px-4 py-3 text-white placeholder:text-white/40 focus:border-white/40 focus:outline-none"
                  >
                </label>

                <p v-if="errorMessage" class="rounded-2xl border border-red-500/30 bg-red-500/10 px-4 py-3 text-sm text-red-100">
                  {{ errorMessage }}
                </p>

                <button
                  type="submit"
                  class="w-full rounded-2xl bg-[var(--ap-accent)] px-4 py-3 text-sm font-semibold uppercase tracking-wide text-white shadow-lg"
                  style="box-shadow: 0 12px 24px var(--ap-accent-shadow);"
                  :disabled="isBusy"
                >
                  {{ isBusy ? 'Signing in…' : 'Continue' }}
                </button>
              </form>

              <p class="text-xs text-white/50">
                This login currently uses Supabase email/password. You can swap in magic links or SSO later.
              </p>
            </div>
          </div>
        </section>
      </main>
    </div>
  </div>
</template>
