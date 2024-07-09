<template>
  <div class="m-side-menu-header-levels">
    <NuxtLink v-if="submenuLink" :to="submenuLink">
      <AButton class="-primary -border-thin" @click.native="toggleSideMenu"
        ><span class="__title">{{ submenuTitle }}</span>
        <template #rightSvg>
          <ASvgIcon
            class="ml-4 -sm"
            path="solid/arrow-right"
          ></ASvgIcon> </template
      ></AButton>
    </NuxtLink>
    <ASvgIcon
      class="-sm __icon"
      path="solid/chevron-left"
      @click.native="closeChild"
    ></ASvgIcon>
  </div>
</template>

<script>
import { inject, defineComponent } from '@nuxtjs/composition-api'
import { string } from 'vue-types'
import useOverlap from '@/composition/useOverlap'

export default defineComponent({
  props: {
    submenuLink: string().isRequired,
    submenuTitle: string().isRequired
  },
  setup() {
    const { toggleSideMenu } = useOverlap

    const closeChild = inject('closeChild')
    return {
      toggleSideMenu,
      closeChild: closeChild ?? null
    }
  }
})
</script>

<style scoped lang="postcss">
.m-side-menu-header-levels {
  .__title {
    @apply text-base truncate block max-w-[160px];
  }

  .__icon {
    @apply border-gray-400 border-2 border-solid
        rounded-full;
  }
}
</style>
