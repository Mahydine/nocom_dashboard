<template>
  <section class="grid sm:grid-cols-2 lg:grid-cols-4 gap-3 md:gap-4">
    <div
      v-for="s in stats" :key="s.key"
      class="rounded-2xl border border-zinc-800 bg-zinc-900/40 p-4"
    >
      <div class="text-xs text-zinc-400">{{ s.label }}</div>
      <div class="mt-2 text-2xl font-semibold">{{ s.value }}</div>
      <p class="mt-1 text-[11px] text-zinc-500">{{ s.sub }}</p>
    </div>
  </section>
</template>

<script setup>
import { ref } from 'vue'

// chiffres plausibles (tu peux les brancher à une API plus tard)
const stats = ref([
  // demandés
  { key: 'base_blocks', label: 'Blocs de base téléchargés', value: '1 274 520', sub: 'Dernières 24h · caches multi-noeuds' },
  { key: 'recent_players', label: 'Joueurs détectés récemment', value: '312', sub: 'Fenêtre glissante 24h' },

  // +5 liés à NoCom
  { key: 'packets', label: 'Paquets interceptés', value: '8,4 M', sub: 'TCP/KeepAlive, mouvement, chunk load' },
  { key: 'chunks_profiled', label: 'Chunks profilés', value: '42 130', sub: 'Sélectionnés par heuristiques' },
  { key: 'coords_extracted', label: 'Coordonnées valides extraites', value: '1 823', sub: 'Confiance ≥ 0,92' },
  { key: 'false_positives', label: 'Faux positifs (24h)', value: '1,6 %', sub: 'Après filtrage de bruit' },
  { key: 'active_bases', label: 'Bases actives surveillées', value: '97', sub: '≥ 3 signaux concordants' },
])
</script>
