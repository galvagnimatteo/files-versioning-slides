<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps<{ repo: string; path: string; lines?: string; label?: string }>()

const REPOS: Record<string, { name: string; base: string; private: boolean }> = {
  ce: {
    name: 'files-ce',
    base: 'https://github.com/zextras/carbonio-files-ce/blob/a1da2eff79db4866e679d45f6cfd43ec235cac1b',
    private: false,
  },
  poc: {
    name: 'files-ce (PoC)',
    base: 'https://github.com/zextras/carbonio-files-ce/blob/2a8f2c74ccce84f6e77c5c0a488dfbbabb8bebc9',
    private: false,
  },
  pochead: {
    name: 'files-ce (PoC)',
    base: 'https://github.com/zextras/carbonio-files-ce/blob/9e3095090705ae16597246e94ebbf927f6e2534b',
    private: false,
  },
  commit: {
    name: 'files-ce commit',
    base: 'https://github.com/zextras/carbonio-files-ce/commit',
    private: false,
  },
  ext: {
    name: 'quarkus-extensions',
    base: 'https://github.com/zextras/carbonio-quarkus-extensions/blob/1c778ac875a32795fd4c86a3dc1cb5713d54497f',
    private: true,
  },
  jlc: {
    name: 'jenkins-lib-common',
    base: 'https://github.com/zextras/jenkins-lib-common/blob/2d89aa24e4a6b3846071cd83a5915ecdd5567895',
    private: true,
  },
  ui: {
    name: 'files-ui 3.2.3',
    base: 'https://github.com/zextras/carbonio-files-ui/blob/b16327349d577e86c099fb24301c3babb13d461b',
    private: false,
  },
}

const repo = computed(() => REPOS[props.repo])
const href = computed(() => {
  const anchor = props.lines ? '#L' + props.lines.replace('-', '-L') : ''
  return `${repo.value.base}/${props.path}${anchor}`
})
const text = computed(() => {
  if (props.label) return props.label
  if (props.repo === 'commit') return props.path.slice(0, 8)
  const file = props.path.split('/').pop()
  return props.lines ? `${file}:${props.lines}` : file
})
</script>

<template>
  <a class="src" :href="href" target="_blank" rel="noopener">
    <span class="src-repo">{{ repo.name }}</span>
    <span>{{ text }}</span>
    <span v-if="repo.private" class="src-private">privato</span>
  </a>
</template>
