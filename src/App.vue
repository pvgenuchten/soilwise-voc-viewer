<script setup>
import { computed, nextTick, onMounted, onBeforeUnmount, ref } from 'vue'

const DATA_URL = import.meta.env.VITE_VOCABULARY_URL || 'https://soilwise-he.github.io/soil-vocabs/assets/soilvoc_data.json'
const loading = ref(true)
const loadError = ref('')
const concepts = ref([])
const byId = ref(new Map())
const roots = ref([])
const selectedId = ref('')
const query = ref('')
const expanded = ref(new Set())
const mobileSidebarOpen = ref(false)
const treeWrap = ref(null)

function asArray(v) {
  if (v == null) return []
  return Array.isArray(v) ? v : [v]
}
function valueText(v) {
  if (v == null) return ''
  if (typeof v === 'string' || typeof v === 'number') return String(v)
  if (Array.isArray(v)) return valueText(v[0])
  if (typeof v === 'object') return String(v['@value'] ?? v.value ?? v.text ?? v.label ?? v.uri ?? v['@id'] ?? '')
  return ''
}
function values(v) { return asArray(v).map(valueText).filter(Boolean) }
function first(obj, keys) {
  for (const k of keys) if (obj?.[k] != null) return obj[k]
  return null
}
function idOf(x) {
  if (typeof x === 'string') return x
  if (!x || typeof x !== 'object') return ''
  return String(x['@id'] ?? x.id ?? x.uri ?? x.value ?? '')
}
function localId(id) {
  if (!id) return ''
  const hash = id.lastIndexOf('#')
  if (hash >= 0 && hash < id.length - 1) return id.slice(hash + 1)
  return id.replace(/\/+$/, '').split('/').pop() || id
}
function normalize(raw) {
  // The published EU Soil Vocabulary JSON wraps a nested hierarchy in
  // vocabulary.top_concepts, with children in narrower and procedures.
  const isEusoilVocabulary = Array.isArray(raw?.vocabulary?.top_concepts)
  let rows = []
  if (isEusoilVocabulary) rows = raw.vocabulary.top_concepts
  else if (Array.isArray(raw)) rows = raw
  else if (Array.isArray(raw?.['@graph'])) rows = raw['@graph']
  else if (Array.isArray(raw?.concepts)) rows = raw.concepts
  else if (Array.isArray(raw?.items)) rows = raw.items
  else if (Array.isArray(raw?.data)) rows = raw.data
  else if (raw && typeof raw === 'object') {
    const maybe = Object.values(raw).find(v => Array.isArray(v) && v.some(x => x && typeof x === 'object' && (x['@id'] || x.id || x.uri || x.prefLabel || x.label)))
    rows = maybe || []
  }

  // Flatten nested concepts and retain the explicit parent-child structure.
  const flat = []
  const seenObjects = new Set()
  const parentById = new Map()
  const explicitRootIds = []
  function add(item, parentId = '') {
    if (!item || typeof item !== 'object' || Array.isArray(item) || seenObjects.has(item)) return
    seenObjects.add(item)
    const id = idOf(item)
    const type = values(first(item, ['@type', 'type', 'rdf:type'])).join(' ')
    const looksLikeConcept = id || item.prefLabel || item['skos:prefLabel'] || item.label || /Concept/i.test(type)
    if (looksLikeConcept) {
      flat.push(item)
      if (parentId && id && !parentById.has(id)) parentById.set(id, parentId)
    }
    const currentId = id || parentId
    // `procedures` in this schema is a collection of procedure records attached
    // to a concept; include them in navigation as children as well.
    for (const key of ['narrower', 'procedures', 'member', 'members', 'concepts', 'items', 'hasTopConcept', 'skos:member']) {
      for (const child of asArray(item[key])) {
        if (child && typeof child === 'object') add(child, currentId)
      }
    }
  }
  rows.forEach(item => {
    const id = idOf(item)
    if (id && isEusoilVocabulary) explicitRootIds.push(id)
    add(item)
  })

  const unique = new Map()
  for (const x of flat) {
    const id = idOf(x)
    if (id) unique.set(id, x)
  }
  const result = [...unique.entries()].map(([id, x]) => {
    const label = values(first(x, ['prefLabel', 'skos:prefLabel', 'label', 'http://www.w3.org/2004/02/skos/core#prefLabel']))[0] || localId(id)
    const altLabels = values(first(x, ['altLabel', 'skos:altLabel', 'alternativeLabel']))
    // definitions[] entries use { text, source }; use their text if the
    // convenience `definition` string is missing, and avoid rendering objects.
    const definitionRaw = first(x, ['definition', 'skos:definition', 'description', 'comment', 'scopeNote', 'skos:scopeNote', 'http://www.w3.org/2004/02/skos/core#definition'])
    const definition = values(definitionRaw).length
      ? values(definitionRaw)
      : asArray(x.definitions).map(d => valueText(d?.text ?? d)).filter(Boolean)
    const broader = asArray(first(x, ['broader', 'skos:broader', 'http://www.w3.org/2004/02/skos/core#broader'])).map(idOf).filter(Boolean)
    const narrower = [
      ...asArray(first(x, ['narrower', 'skos:narrower', 'http://www.w3.org/2004/02/skos/core#narrower'])),
      ...asArray(x.procedures)
    ].map(idOf).filter(Boolean)
    const types = values(first(x, ['@type', 'type', 'kind', 'category']))
    const procedure = x.isProcedure === true || x['soilvoc:procedure'] === true ||
      /procedure/i.test([label, localId(id), ...types].join(' '))
    return { id, key: localId(id), label, altLabels, definition, broader, narrower, types, procedure, raw: x, children: [] }
  })

  const map = new Map(result.map(c => [c.id, c]))
  const aliases = new Map()
  for (const c of result) {
    aliases.set(c.id, c.id)
    aliases.set(c.key, c.id)
    aliases.set(c.id.toLowerCase(), c.id)
    aliases.set(c.key.toLowerCase(), c.id)
  }
  function resolve(id) {
    const rawId = String(id || '')
    return aliases.get(rawId) || aliases.get(localId(rawId)) || aliases.get(rawId.toLowerCase()) || ''
  }
  for (const c of result) {
    c.broader = c.broader.map(resolve).filter(id => id && id !== c.id)
    c.narrower = c.narrower.map(resolve).filter(id => id && id !== c.id)
    const nestedParent = parentById.get(c.id)
    const resolvedParent = nestedParent ? resolve(nestedParent) : ''
    if (resolvedParent && resolvedParent !== c.id && !c.broader.includes(resolvedParent)) c.broader.push(resolvedParent)
  }
  // Fill missing inverse edges so both explicit broader and nested hierarchy work.
  for (const c of result) {
    for (const parentId of c.broader) {
      const parent = map.get(parentId)
      if (parent && !parent.narrower.includes(c.id)) parent.narrower.push(c.id)
    }
    for (const childId of c.narrower) {
      const child = map.get(childId)
      if (child && !child.broader.includes(c.id)) child.broader.push(c.id)
    }
  }
  // Procedures are represented as leaf nodes in the explorer.
  for (const c of result) {
    if (c.procedure) c.narrower = []
    c.children = c.narrower.map(id => map.get(id)).filter(Boolean)
  }
  const childIds = new Set(result.flatMap(c => c.narrower))
  let top = result.filter(c => !childIds.has(c.id))
  if (!top.length) top = result.slice(0, 1)
  if (explicitRootIds.length) {
    const explicitRoots = explicitRootIds.map(resolve).map(id => map.get(id)).filter(Boolean)
    if (explicitRoots.length) top = explicitRoots
  } else {
    const explicitTop = result.filter(c => c.raw.topConcept === true || c.raw['skos:topConceptOf'] || values(c.raw['@type']).some(t => /TopConcept/i.test(t)))
    if (explicitTop.length) top = explicitTop
  }
  return { concepts: result, map, roots: top, resolve }
}

const selected = computed(() => byId.value.get(selectedId.value) || null)
const searchResults = computed(() => {
  const q = query.value.trim().toLocaleLowerCase()
  if (!q) return []
  return concepts.value.filter(c => [c.label, c.key, c.id, ...c.altLabels, ...c.definition].join(' ').toLocaleLowerCase().includes(q))
    .sort((a, b) => a.label.localeCompare(b.label)).slice(0, 100)
})
const isSearching = computed(() => query.value.trim().length > 0)
function childrenOf(c) { return (c?.narrower || []).map(id => byId.value.get(id)).filter(Boolean) }
function isExpanded(id) { return expanded.value.has(id) }
function toggle(id) {
  const next = new Set(expanded.value)
  next.has(id) ? next.delete(id) : next.add(id)
  expanded.value = next
}
function expandAncestors(id) {
  const next = new Set(expanded.value)
  const visited = new Set()
  function visit(current) {
    if (!current || visited.has(current)) return
    visited.add(current)
    const c = byId.value.get(current)
    for (const p of c?.broader || []) { next.add(p); visit(p) }
  }
  visit(id)
  expanded.value = next
}
function scrollSelectedIntoView(id) {
  nextTick(() => {
    const rows = treeWrap.value?.querySelectorAll('[data-concept-id]') || []
    const row = [...rows].find(el => el.dataset.conceptId === id)
    if (row) row.scrollIntoView({ block: 'nearest', inline: 'nearest', behavior: 'smooth' })
  })
}
function openConcept(id, updateUrl = true) {
  const c = byId.value.get(id) || byId.value.get(resolveId(id))
  if (!c) return
  selectedId.value = c.id
  expandAncestors(c.id)
  mobileSidebarOpen.value = false
  if (updateUrl) {
    const hash = encodeURIComponent(c.key || c.id)
    history.pushState({ concept: c.id }, '', `${location.pathname}${location.search}#${hash}`)
  }
  scrollSelectedIntoView(c.id)
}
function openSearchResult(id) {
  query.value = ''
  openConcept(id)
}
function resolveId(id) {
  const raw = decodeURIComponent(String(id || '').replace(/^#/, ''))
  return byId.value.has(raw) ? raw : concepts.value.find(c => c.key === raw || c.id.endsWith('#' + raw) || c.id.endsWith('/' + raw))?.id || ''
}
function openFromHash() {
  const id = resolveId(location.hash.slice(1))
  if (id) openConcept(id, false)
}
function conceptUrl(c) { return `${location.pathname}${location.search}#${encodeURIComponent(c.key || c.id)}` }
function onPopState() { openFromHash() }
function ancestorsOf(c) {
  // Some procedures are shared by multiple properties, so their broader
  // relations form a graph rather than a single tree. Breadcrumbs should show
  // one concise route, not concatenate every possible parent path.
  const rootIds = new Set(roots.value.map(root => root.id))
  const queue = [[c.id]]
  const visited = new Set([c.id])
  let path = null

  while (queue.length) {
    const currentPath = queue.shift()
    const currentId = currentPath[currentPath.length - 1]
    if (rootIds.has(currentId)) {
      path = currentPath
      break
    }
    const current = byId.value.get(currentId)
    const parents = (current?.broader || [])
      .map(id => byId.value.get(id))
      .filter(Boolean)
      .sort((a, b) => a.label.localeCompare(b.label))
    for (const parent of parents) {
      if (visited.has(parent.id)) continue
      visited.add(parent.id)
      queue.push([...currentPath, parent.id])
    }
  }

  // Reverse the selected -> root path into root -> parent order.
  if (path) return path.slice(1).reverse().map(id => byId.value.get(id)).filter(Boolean)
  return []
}
function displayId(c) { return c?.key || c?.id || '' }
function shareConcept(c) {
  const url = window.location.origin + conceptUrl(c)
  if (navigator.clipboard?.writeText) navigator.clipboard.writeText(url).catch(() => window.prompt('Copy concept link', url))
  else window.prompt('Copy concept link', url)
}
function reloadPage() { window.location.reload() }
function labelFor(id) { return byId.value.get(id)?.label || localId(id) }
function typeLabel(c) { return c.procedure ? 'Procedure' : 'Soil concept' }
function safeHref(value) {
  const text = valueText(value).trim()
  return /^https?:\/\//i.test(text) ? text : ''
}
function sourceReferences(c) {
  const rows = asArray(c?.raw?.definitions).map((definition) => {
    const source = definition?.source ?? definition?.['dct:source'] ?? definition?.['prov:wasDerivedFrom']
    const href = safeHref(source)
    return href ? { href, label: valueText(source) } : null
  }).filter(Boolean)
  return [...new Map(rows.map(row => [row.href, row])).values()]
}
function matchEntries(c, kind) {
  const keys = kind === 'exact'
    ? ['exactMatch', 'skos:exactMatch', 'http://www.w3.org/2004/02/skos/core#exactMatch']
    : ['closeMatch', 'skos:closeMatch', 'http://www.w3.org/2004/02/skos/core#closeMatch']
  const entries = asArray(first(c?.raw, keys)).map((entry) => {
    const href = safeHref(entry?.uri ?? entry?.['@id'] ?? entry?.url ?? entry)
    if (!href) return null
    return { href, label: valueText(entry?.label) || href }
  }).filter(Boolean)
  return [...new Map(entries.map(entry => [entry.href, entry])).values()]
}

onMounted(async () => {
  window.addEventListener('popstate', onPopState)
  window.addEventListener('hashchange', openFromHash)
  try {
    const response = await fetch(DATA_URL)
    if (!response.ok) throw new Error(`Vocabulary request failed (${response.status})`)
    const raw = await response.json()
    const parsed = normalize(raw)
    concepts.value = parsed.concepts
    byId.value = parsed.map
    roots.value = parsed.roots
    if (!concepts.value.length) throw new Error('No concepts were recognized in the JSON. Check the source data structure.')
    // Start with a compact hierarchy: only top concepts are visible.
    expanded.value = new Set()
    const requested = resolveId(location.hash.slice(1))
    if (requested) openConcept(requested, false)
    else if (roots.value[0]) selectedId.value = roots.value[0].id
  } catch (e) {
    loadError.value = `${e.message}. If this is a CORS restriction, download the JSON into public/soilvoc_data.json and change DATA_URL in src/App.vue to '/soilvoc_data.json'.`
  } finally { loading.value = false }
})
onBeforeUnmount(() => {
  window.removeEventListener('popstate', onPopState)
  window.removeEventListener('hashchange', openFromHash)
})
</script>

<template>
  <div class="app-shell">
    <header class="topbar">
      <button class="mobile-menu icon-button" aria-label="Toggle vocabulary navigation" @click="mobileSidebarOpen = !mobileSidebarOpen">☰</button>
      <a class="brand" href="./" @click.prevent="selectedId = roots[0]?.id || ''; history.pushState({}, '', location.pathname + location.search)">
        <span class="brand-mark">S</span>
        <span><strong>Soil Vocabulary</strong><small>Concept explorer</small></span>
      </a>
      <div class="topbar-meta"><span class="status-dot"></span> SKOS vocabulary <span class="meta-divider">·</span> {{ concepts.length ? concepts.length.toLocaleString() + ' concepts' : 'Loading concepts' }}</div>
    </header>

    <div class="workspace">
      <div v-if="mobileSidebarOpen" class="mobile-scrim" @click="mobileSidebarOpen = false"></div>
      <aside class="sidebar" :class="{ 'sidebar-open': mobileSidebarOpen }">
        <div class="sidebar-heading">
          <div><span class="eyebrow">BROWSE</span><h2>Concepts</h2></div>
          <span class="count-pill">{{ concepts.length }}</span>
        </div>
        <label class="search-box">
          <span class="search-icon">⌕</span>
          <input v-model="query" type="search" placeholder="Search concepts…" aria-label="Search concepts and descriptions" />
          <kbd v-if="query" @click="query = ''">Esc</kbd>
        </label>
        <div class="search-caption" v-if="isSearching">{{ searchResults.length }} matching concepts</div>
        <div ref="treeWrap" class="tree-wrap">
          <div v-if="isSearching" class="search-list">
            <button v-for="c in searchResults" :key="c.id" class="search-result" :class="{ active: selectedId === c.id }" @click="openSearchResult(c.id)">
              <span class="result-glyph" :class="{ procedure: c.procedure }">{{ c.procedure ? '↗' : '◇' }}</span>
              <span class="result-text"><strong>{{ c.label }}</strong><small>{{ c.definition[0] || c.key }}</small></span>
            </button>
            <p v-if="!searchResults.length" class="empty-state">No concepts match “{{ query }}”. Try another term.</p>
          </div>
          <template v-else>
            <TreeNode v-for="root in roots" :key="root.id" :node="root" :depth="0" :selected-id="selectedId" :expanded="expanded" :map="byId" @toggle="toggle" @open="openConcept" />
          </template>
        </div>
        <div class="sidebar-footer"><span class="footer-leaf">↗</span> Procedures are shown as leaf concepts</div>
      </aside>

      <main class="content">
        <div v-if="loading" class="center-state"><div class="loader"></div><p>Loading the soil vocabulary…</p></div>
        <div v-else-if="loadError" class="error-panel"><div class="error-icon">!</div><h2>Couldn’t load the vocabulary</h2><p>{{ loadError }}</p><button class="primary-button" @click="reloadPage()">Try again</button></div>
        <template v-else-if="selected">
          <div class="breadcrumbs">
            <span>Vocabulary</span><span class="crumb-sep">/</span>
            <template v-for="(p, i) in ancestorsOf(selected)" :key="p.id">
              <button @click="openConcept(p.id)">{{ p.label }}</button><span class="crumb-sep">/</span>
            </template>
            <span class="current-crumb">{{ selected.label }}</span>
          </div>
          <section class="concept-hero" :class="{ 'procedure-hero': selected.procedure }">
            <div class="hero-topline">
              <span class="type-chip" :class="{ 'procedure-chip': selected.procedure }"><span>{{ selected.procedure ? '↗' : '◇' }}</span>{{ typeLabel(selected) }}</span>
              <button class="share-button" title="Copy or open this concept link" @click="shareConcept(selected)">↗ <span>Share concept</span></button>
            </div>
            <div class="hero-title-row">
              <div class="hero-symbol" :class="{ 'procedure-symbol': selected.procedure }">{{ selected.procedure ? '↗' : '◇' }}</div>
              <div><h1>{{ selected.label }}</h1><p class="concept-uri">{{ selected.id }}</p></div>
            </div>
            <div v-if="selected.altLabels.length" class="alt-labels"><span class="field-label">ALSO KNOWN AS</span><span v-for="alt in selected.altLabels" :key="alt" class="alt-chip">{{ alt }}</span></div>
          </section>

          <div class="detail-grid">
            <section class="detail-card description-card">
              <div class="section-heading"><span class="section-icon">≡</span><div><h2>{{ selected.procedure ? 'Procedure description' : 'About this concept' }}</h2><p>{{ selected.procedure ? 'What this procedure covers' : 'Definition and explanatory notes' }}</p></div></div>
              <div v-if="selected.definition.length" class="definition-list">
                <p v-for="(d, i) in selected.definition" :key="i">{{ d }}</p>
              </div>
              <div v-else class="muted-empty">No description is available for this concept.</div>
              <div v-if="selected.raw.example || selected.raw['skos:example']" class="extra-field"><span class="field-label">EXAMPLE</span><p>{{ values(selected.raw.example || selected.raw['skos:example']).join(' · ') }}</p></div>
              <div v-if="selected.raw.note || selected.raw['skos:note'] || selected.raw.scopeNote || selected.raw['skos:scopeNote']" class="extra-field"><span class="field-label">NOTE</span><p>{{ values(selected.raw.note || selected.raw['skos:note'] || selected.raw.scopeNote || selected.raw['skos:scopeNote']).join(' · ') }}</p></div>
              <div v-if="sourceReferences(selected).length" class="extra-field reference-field">
                <span class="field-label">SOURCE REFERENCES</span>
                <a v-for="source in sourceReferences(selected)" :key="source.href" class="external-reference" :href="source.href" target="_blank" rel="noopener noreferrer">{{ source.label }} <span>↗</span></a>
              </div>
              <div v-if="matchEntries(selected, 'exact').length" class="extra-field reference-field">
                <span class="field-label">EXACT MATCH</span>
                <a v-for="match in matchEntries(selected, 'exact')" :key="match.href" class="external-reference" :href="match.href" target="_blank" rel="noopener noreferrer">{{ match.label }} <span>↗</span></a>
              </div>
              <div v-if="matchEntries(selected, 'close').length" class="extra-field reference-field">
                <span class="field-label">CLOSE MATCH</span>
                <a v-for="match in matchEntries(selected, 'close')" :key="match.href" class="external-reference" :href="match.href" target="_blank" rel="noopener noreferrer">{{ match.label }} <span>↗</span></a>
              </div>
            </section>
            <section class="detail-card relations-card">
              <div class="section-heading"><span class="section-icon">⌘</span><div><h2>Concept relations</h2><p>Navigate the vocabulary network</p></div></div>
              <div class="relation-group">
                <div class="relation-label"><span class="relation-dot broader-dot"></span><span>Broader concepts</span><span class="relation-count">{{ selected.broader.length }}</span></div>
                <button v-for="id in selected.broader" :key="id" class="relation-link" @click="openConcept(id)"><span class="relation-symbol">↑</span><span>{{ labelFor(id) }}</span><span class="link-arrow">↗</span></button>
                <p v-if="!selected.broader.length" class="relation-empty">This is a top-level concept.</p>
              </div>
              <div class="relation-group">
                <div class="relation-label"><span class="relation-dot narrower-dot"></span><span>{{ selected.procedure ? 'Related narrower concepts' : 'Narrower concepts' }}</span><span class="relation-count">{{ selected.narrower.length }}</span></div>
                <button v-for="id in selected.narrower" :key="id" class="relation-link" :class="{ 'relation-procedure': byId.get(id)?.procedure }" @click="openConcept(id)"><span class="relation-symbol">{{ byId.get(id)?.procedure ? '↗' : '↓' }}</span><span class="relation-name">{{ labelFor(id) }}</span><span v-if="byId.get(id)?.procedure" class="relation-type">Procedure</span><span class="link-arrow">↗</span></button>
                <p v-if="!selected.narrower.length" class="relation-empty">{{ selected.procedure ? 'Procedures are leaf concepts.' : 'No narrower concepts are listed.' }}</p>
              </div>
            </section>
          </div>
          <section v-if="!selected.procedure && childrenOf(selected).length" class="children-section">
            <div class="children-heading"><div><span class="eyebrow">EXPLORE FURTHER</span><h2>Narrower concepts</h2></div><span class="count-pill">{{ childrenOf(selected).length }}</span></div>
            <div class="children-grid">
              <button v-for="child in childrenOf(selected)" :key="child.id" class="child-card" @click="openConcept(child.id)">
                <span class="child-icon" :class="{ 'procedure-symbol': child.procedure }">{{ child.procedure ? '↗' : '◇' }}</span>
                <span class="child-info"><strong>{{ child.label }}</strong><small>{{ child.procedure ? 'Procedure' : 'Concept' }}</small><span>{{ child.definition[0] || 'Explore this concept and its relations.' }}</span></span>
                <span class="child-arrow">↗</span>
              </button>
            </div>
          </section>
          <footer class="content-footer"><span>SOIL VOCABULARY EXPLORER</span><span>Concept ID: {{ displayId(selected) }}</span></footer>
        </template>
      </main>
    </div>
  </div>
</template>

<script>
import { defineComponent, h } from 'vue'
const TreeNode = defineComponent({
  name: 'TreeNode',
  props: { node: Object, depth: Number, selectedId: String, expanded: Object, map: Object },
  emits: ['toggle', 'open'],
  setup(props, { emit }) {
    return () => {
      const node = props.node
      const children = (node.narrower || []).map(id => props.map.get(id)).filter(Boolean)
      const expanded = props.expanded.has(node.id)
      return h('div', { class: 'tree-node' }, [
        h('div', { class: ['tree-row', { 'tree-active': props.selectedId === node.id, 'tree-procedure': node.procedure }], style: { '--depth': props.depth }, 'data-concept-id': node.id }, [
          children.length
            ? h('button', { class: 'tree-toggle', 'aria-label': expanded ? 'Collapse' : 'Expand', onClick: () => emit('toggle', node.id) }, [h('span', { class: { rotated: expanded } }, '›')])
            : h('span', { class: 'tree-toggle-placeholder' }),
          h('button', { class: 'tree-label', onClick: () => emit('open', node.id), title: node.label }, [
            h('span', { class: ['tree-glyph', { 'tree-glyph-procedure': node.procedure }] }, node.procedure ? '↗' : '◇'),
            h('span', { class: 'tree-label-text' }, node.label)
          ]),
          node.procedure ? h('span', { class: 'procedure-mini', title: 'Procedure' }, 'P') : null
        ]),
        expanded && children.length ? h('div', { class: 'tree-children' }, children.map(child => h(TreeNode, { key: child.id, node: child, depth: props.depth + 1, selectedId: props.selectedId, expanded: props.expanded, map: props.map, onToggle: id => emit('toggle', id), onOpen: id => emit('open', id) }))) : null
      ])
    }
  }
})
export default { components: { TreeNode } }
</script>