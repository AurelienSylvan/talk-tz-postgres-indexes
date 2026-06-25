<template>
  <div class="slidev-layout three-cols h-full flex flex-col">
    <slot />
    <div class="flex flex-row gap-4 flex-1 overflow-auto">
      <div class="flex-1 overflow-auto">
        <slot name="left" />
      </div>
      <div class="flex-1 overflow-auto">
        <slot name="middle" />
      </div>
      <div class="flex-1 overflow-auto">
        <slot name="right" />
      </div>
    </div>
  </div>
</template>
