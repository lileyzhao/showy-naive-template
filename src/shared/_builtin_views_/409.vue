<script setup lang="ts" name="VIEWS-BUILTIN-403">
const router = useRouter()

const handleBackToHome = () => router.push({ path: '/' })
</script>

<template>
  <div class="size-full min-h-100vh flex-y-center flex-col gap-12 overflow-hidden pt-3">
    <div class="i-carbon:event-warning flex text-200px" />
    <div>conflict 409</div>
    <NButton type="primary" @click="handleBackToHome">
      {{ $t('common.backToHome') }}
    </NButton>
  </div>
</template>

<style scoped></style>
