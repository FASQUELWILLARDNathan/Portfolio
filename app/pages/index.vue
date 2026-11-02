<template>
  <ClientOnly>
    <BasementScene />
    <template #fallback><div style="min-height:100vh;background:#050708;color:#8dffac;padding:3rem;font-family:monospace">INITIALIZING TERMINAL...</div></template>
  </ClientOnly>
</template>