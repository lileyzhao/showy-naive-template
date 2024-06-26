<script setup lang="ts" name="Layout-Index">
import { useDebounceFn } from '@vueuse/core'
import MainSidebar from './components/MainSidebar.vue'
import SubSidebar from './components/SubSidebar.vue'
import ThemeDrawer from './components/ThemeDrawer.vue'
import MobileDrawer from './components/MobileDrawer.vue'
import TopBar from './components/TopBar.vue'
import { useAppStore } from '@/store'
import { MenuButtonEnum, MenuPositionEnum, ScreenEnum } from '@/shared'
import { getFullRoutes } from '@/utils'

const app = useAppStore()
const route = useRoute()
const fullRoutes = getFullRoutes()

const mountTimeout = ref()
const mainSidebarRef = ref<InstanceType<typeof MainSidebar>>()
const topBarRef = ref<InstanceType<typeof TopBar>>()
const mobileDrawerRef = ref<InstanceType<typeof MobileDrawer>>()
const themeDrawerRef = ref<InstanceType<typeof ThemeDrawer>>()

// Get menu settings from the app store. 从应用存储中获取菜单设置。
const menuSetting = computed(() => app.MenuSetting)

/** Whether the menu is in the top bar layout. 是否顶栏菜单布局。 */
const isTopBar = computed(() => menuSetting.value.menuPosition === MenuPositionEnum.TOP_BAR)

/** Selected item in the main menu. 主栏菜单选中项。 */
const mainMenuKey = ref<string>()

/** Root route name of the main menu. 主菜单的根路由名。 */
const mainMenuRootKey = computed(() => {
  const findRootRoute = (parentName: string): string | undefined => {
    const parentRoute = fullRoutes.find(r => r.name === parentName)
    if (!parentRoute || parentRoute.name === 'root' || parentRoute.meta.parentName === 'root')
      return parentName

    return findRootRoute(parentRoute.meta.parentName as string)
  }

  const currRoute = fullRoutes.find(r => r.name === mainMenuKey.value)
  if (!currRoute)
    return route.matched[1].name as string
  else if (currRoute.meta.parentName === 'root')
    return currRoute.name as string
  else return findRootRoute(currRoute.meta.parentName as string)
})

/** Main menu selected item changed. 主栏菜单选中项改变。 */
const handleMainMenuKeyChange = (key: string) => {
  if (mainMenuKey.value === key)
    return
  mainMenuKey.value = key
  clearTimeout(mountTimeout.value)
}

/** Restore the sub-menu when the mouse enters the content area. 复原副栏菜单(当鼠标移入内容区时)。 */
const restoreSubMenu = useDebounceFn(() => {
  mountTimeout.value = setTimeout(() => {
    if (!menuSetting.value.subMenu.collapsed) {
      // 刷新主栏菜单
      if (!app.isMobile && !isTopBar.value)
        mainSidebarRef.value?.refreshMainMenu()
      else if (!app.isMobile)
        topBarRef.value?.refreshTopMenu()
    }
  }, 650)
}, 500)

/** Cancel the restore of the sub-menu. 取消复原副栏菜单。 */
const cancelRestoreSubMenu = () => {
  clearTimeout(mountTimeout.value)
}

/** Sub-menu collapse state changed. 副栏菜单折叠状态改变。 */
const handleSubCollapsed = (collSubMenu: boolean) => {
  // Update the main menu. 更新主栏菜单。
  mainMenuKey.value = (collSubMenu ? route.name : route.matched[1].name) as string
}

/** Trigger mobile mode. 触发移动端模式。 */
const triggerMobileMode = () => {
  if (document.body.clientWidth <= ScreenEnum.MD)
    app.isMobile = true
  else app.isMobile = false
}

onMounted(async () => {
  // Trigger mobile detection and add a listener. 触发移动端检测并添加监听事件。
  triggerMobileMode()
  window.addEventListener('resize', triggerMobileMode)

  // Update the main menu. 更新主栏菜单。
  mainMenuKey.value = (menuSetting.value.subMenu.collapsed ? route.name : route.matched[1].name) as string
})

onUnmounted(() => {
  window.removeEventListener('resize', triggerMobileMode)
})

/** Handle various actions like toggling drawers. 处理各种操作，如切换抽屉。 */
const handleAction = (op: string, _val: any) => {
  if (op === 'toggleMobileDrawer')
    mobileDrawerRef.value?.show()
  else if (op === 'toggleThemeDrawer')
    themeDrawerRef.value?.show()
}
</script>

<template>
  <n-layout has-sider position="absolute">
    <!-- Sidebar (Desktop): Main Sidebar. 侧边栏(电脑端):主栏。 -->
    <MainSidebar
      v-if="!app.isMobile && !isTopBar" ref="mainSidebarRef" :menu-key="mainMenuKey"
      @key-change="handleMainMenuKeyChange"
    />
    <!-- Sidebar (Desktop): Sub Sidebar. 侧边栏(电脑端):副栏。 -->
    <SubSidebar
      v-if="!app.isMobile && (!isTopBar || menuSetting.topMenu.showSubMenu)"
      :parent-menu-key="mainMenuRootKey" @collapsed="handleSubCollapsed"
    />

    <!-- Right main area. 右侧主体区。 -->
    <n-layout content-class="flex flex-col">
      <!-- Top bar area. 头部横栏区。 -->
      <TopBar
        ref="topBarRef" @action="handleAction" @key-change="handleMainMenuKeyChange"
        @mouseenter="cancelRestoreSubMenu"
      />
      <!-- Content area. 内容区。 -->
      <n-layout has-sider @mouseenter="restoreSubMenu" @mouseleave="cancelRestoreSubMenu">
        <n-layout-content :native-scrollbar="false" flex-1 :style="app.IsDarkMode ? 'background-color: #18181c;' : ''">
          <div p-8px :style="{ backgroundColor: app.IsDarkMode ? '#26262a' : '#f7fafc' }">
            <router-view v-slot="{ Component, route: r }">
              <transition name="fade">
                <keep-alive :max="25">
                  <component :is="Component" :key="r.fullPath" />
                </keep-alive>
              </transition>
            </router-view>
          </div>
          <n-back-top :right="40" />
        </n-layout-content>
      </n-layout>
    </n-layout>

    <!-- Drawer (Mobile). 抽屉栏(手机端)。 -->
    <MobileDrawer v-if="app.isMobile" ref="mobileDrawerRef" />

    <!-- Theme settings drawer. 主题设置抽屉栏。 -->
    <ThemeDrawer v-if="!app.isMobile && app.hasMenuButton(MenuButtonEnum.ThemeDrawer)" ref="themeDrawerRef" />
  </n-layout>
</template>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.1s;
}

.fade-enter,
.fade-leave-to {
  opacity: 0;
}
</style>
