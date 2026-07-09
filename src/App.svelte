<script>
  import { onMount } from 'svelte'
  import QRCode from 'qrcode'
  import {
    Check,
    ChevronLeft,
    ChevronRight,
    Copy,
    Grid2X2,
    Home,
    Maximize2,
    Monitor,
    MonitorPlay,
    Moon,
    Share2,
    Sun,
    X,
  } from '@lucide/svelte'
  import { sections, slides } from './lib/slides.js'

  const themeStorageKey = 'star-membership-theme'

  const themeModes = [
    { value: 'system', label: 'System' },
    { value: 'dark', label: 'Dark' },
    { value: 'light', label: 'Light' },
  ]

  const themeLabels = Object.fromEntries(themeModes.map((mode) => [mode.value, mode.label]))

  const tonePalette = {
    gold: 'var(--tone-gold)',
    forest: 'var(--tone-forest)',
    clay: 'var(--tone-clay)',
    teal: 'var(--tone-teal)',
    plum: 'var(--tone-plum)',
    leaf: 'var(--tone-leaf)',
  }

  let current = 0
  let overview = false
  let plateContentMotion = true
  let shareDialog
  let shareUrl = ''
  let shareQrCode = ''
  let shareError = ''
  let copiedShareUrl = false
  let copyResetTimer
  let shareDialogClosing = false
  let shareCloseTimer
  let themePreference = 'system'
  let activeTheme = 'light'
  let themeMediaQuery

  $: slide = slides[current]
  $: progress = ((current + 1) / slides.length) * 100
  $: toneStyle = `--accent: ${tonePalette[slide.tone] ?? tonePalette.gold};`
  $: nextThemeModeOption =
    themeModes[(Math.max(themeModes.findIndex((mode) => mode.value === themePreference), 0) + 1) % themeModes.length]
  $: currentThemeLabel =
    themePreference === 'system'
      ? `System (${themeLabels[activeTheme].toLowerCase()} active)`
      : themeLabels[themePreference]
  $: themeToggleLabel = `${currentThemeLabel} theme. Switch to ${nextThemeModeOption.label.toLowerCase()} theme`

  function plateHashFor(index) {
    return `plate-${String(slides[index].number).padStart(2, '0')}`
  }

  function plateIndexFromUrl() {
    if (typeof window === 'undefined') return -1

    const match = window.location.hash.toLowerCase().match(/^#plate-(\d{1,3})$/)
    if (!match) return -1

    const plateNumber = Number(match[1])
    return slides.findIndex((option) => option.number === plateNumber)
  }

  function syncPlateUrl(index) {
    if (typeof window === 'undefined') return

    const url = new URL(window.location.href)
    url.hash = plateHashFor(index)

    if (url.href !== window.location.href) {
      window.history.replaceState({ plate: slides[index].number }, '', url)
    }
  }

  function goTo(index, { animate = true, syncUrl = true } = {}) {
    plateContentMotion = animate
    current = Math.max(0, Math.min(slides.length - 1, index))
    overview = false

    if (syncUrl) {
      syncPlateUrl(current)
    }
  }

  function next(options = {}) {
    goTo(current + 1, options)
  }

  function previous(options = {}) {
    goTo(current - 1, options)
  }

  function toggleFullscreen() {
    if (!document.fullscreenElement) {
      document.documentElement.requestFullscreen?.()
    } else {
      document.exitFullscreen?.()
    }
  }

  function activeSection(number) {
    return [...sections].reverse().find((section) => number >= section.start)?.label
  }

  function normalizeThemePreference(value) {
    return themeModes.some((mode) => mode.value === value) ? value : 'system'
  }

  function systemTheme() {
    return themeMediaQuery?.matches ? 'dark' : 'light'
  }

  function resolveTheme(preference) {
    return preference === 'system' ? systemTheme() : preference
  }

  function applyThemePreference(preference = themePreference) {
    themePreference = normalizeThemePreference(preference)
    activeTheme = resolveTheme(themePreference)

    if (typeof document === 'undefined') return

    const root = document.documentElement
    root.dataset.themePreference = themePreference
    root.dataset.theme = activeTheme
    root.style.colorScheme = activeTheme
  }

  function readThemePreference() {
    try {
      return normalizeThemePreference(window.localStorage.getItem(themeStorageKey))
    } catch (error) {
      return 'system'
    }
  }

  function persistThemePreference(preference) {
    try {
      window.localStorage.setItem(themeStorageKey, preference)
    } catch (error) {
      // Private browsing and locked-down embeds can reject storage writes.
    }
  }

  function setThemePreference(preference) {
    applyThemePreference(preference)
    persistThemePreference(themePreference)
  }

  function toggleThemePreference() {
    setThemePreference(nextThemeModeOption.value)
  }

  function straplineBadges(value) {
    return value?.split('|').map((item) => item.trim()).filter(Boolean) ?? []
  }

  async function refreshShareCode() {
    syncPlateUrl(current)
    shareUrl = window.location.href
    shareQrCode = ''
    shareError = ''

    try {
      shareQrCode = await QRCode.toDataURL(shareUrl, {
        errorCorrectionLevel: 'M',
        margin: 2,
        scale: 8,
        color: {
          dark: '#243040',
          light: '#ffffff',
        },
      })
    } catch (error) {
      shareError = 'QR code unavailable'
    }
  }

  async function openShareDialog() {
    copiedShareUrl = false
    shareDialogClosing = false
    clearTimeout(shareCloseTimer)
    await refreshShareCode()
    if (!shareDialog?.open) {
      shareDialog?.showModal()
    }
  }

  function closeShareDialog() {
    if (!shareDialog?.open || shareDialogClosing) return

    shareDialogClosing = true
    clearTimeout(shareCloseTimer)
    shareCloseTimer = setTimeout(() => {
      shareDialog?.close()
    }, 220)
  }

  function finishShareDialogClose() {
    clearTimeout(shareCloseTimer)
    shareDialog?.close()
  }

  function handleSharePanelAnimationEnd(event) {
    if (shareDialogClosing && event.target === event.currentTarget) {
      finishShareDialogClose()
    }
  }

  function handleShareDialogCancel(event) {
    event.preventDefault()
    closeShareDialog()
  }

  function handleShareDialogClosed() {
    shareDialogClosing = false
    copiedShareUrl = false
    clearTimeout(shareCloseTimer)
  }

  function handleShareDialogClick(event) {
    if (event.target === shareDialog) {
      closeShareDialog()
    }
  }

  async function copyShareLink() {
    if (!shareUrl) return

    try {
      await navigator.clipboard.writeText(shareUrl)
      copiedShareUrl = true
      clearTimeout(copyResetTimer)
      copyResetTimer = setTimeout(() => {
        copiedShareUrl = false
      }, 1800)
    } catch (error) {
      shareError = 'Copy unavailable'
    }
  }

  onMount(() => {
    themeMediaQuery = window.matchMedia?.('(prefers-color-scheme: dark)')
    applyThemePreference(readThemePreference())

    function handleSystemThemeChange() {
      applyThemePreference(themePreference)
    }

    if (themeMediaQuery?.addEventListener) {
      themeMediaQuery.addEventListener('change', handleSystemThemeChange)
    } else {
      themeMediaQuery?.addListener?.(handleSystemThemeChange)
    }

    const linkedPlateIndex = plateIndexFromUrl()
    if (linkedPlateIndex >= 0) {
      goTo(linkedPlateIndex, { animate: false })
    } else {
      syncPlateUrl(current)
    }

    function handleHashChange() {
      const linkedIndex = plateIndexFromUrl()
      if (linkedIndex >= 0) {
        goTo(linkedIndex, { animate: false })
      } else {
        syncPlateUrl(current)
      }
    }

    function handleKeydown(event) {
      if (event.defaultPrevented) return

      const tagName = event.target?.tagName
      if (tagName === 'INPUT' || tagName === 'SELECT' || tagName === 'TEXTAREA') return

      if (event.key === 'ArrowRight' || event.key === 'PageDown' || event.key === ' ') {
        event.preventDefault()
        next({ animate: false })
      }

      if (event.key === 'ArrowLeft' || event.key === 'PageUp') {
        event.preventDefault()
        previous({ animate: false })
      }

      if (event.key === 'Home') {
        event.preventDefault()
        goTo(0, { animate: false })
      }

      if (event.key === 'End') {
        event.preventDefault()
        goTo(slides.length - 1, { animate: false })
      }

      if (event.key.toLowerCase() === 'o') {
        event.preventDefault()
        overview = !overview
      }

      if (event.key === 'Escape' && overview) {
        event.preventDefault()
        overview = false
      }
    }

    window.addEventListener('hashchange', handleHashChange)
    window.addEventListener('keydown', handleKeydown)
    return () => {
      window.removeEventListener('hashchange', handleHashChange)
      window.removeEventListener('keydown', handleKeydown)
      if (themeMediaQuery?.removeEventListener) {
        themeMediaQuery.removeEventListener('change', handleSystemThemeChange)
      } else {
        themeMediaQuery?.removeListener?.(handleSystemThemeChange)
      }
      clearTimeout(copyResetTimer)
      clearTimeout(shareCloseTimer)
    }
  })
</script>

<svelte:head>
  <title>PICPA STAR Membership Blueprint</title>
  <meta
    name="description"
    content="A Svelte and Tailwind presentation redesign for the PICPA STAR Membership Blueprint."
  />
</svelte:head>

<main class="presentation-shell">
  <div class="presentation-grid">
    <section class="deck-area">
      <header class="deck-toolbar">
        <div class="plate-identity">
          <img src="/picpa-logo.png" alt="PICPA logo" />
          <div>
            <span class="toolbar-kicker">PICPA STAR Region</span>
            <p>Membership blueprint / sheet {String(slide.number).padStart(2, '0')}</p>
          </div>
        </div>

        <nav class="section-strip" aria-label="Slide sections">
          {#each sections as section}
            <button
              type="button"
              class:active={activeSection(slide.number) === section.label}
              on:click={() => goTo(section.start - 1)}
            >
              <span>{section.label}</span>
              <small>{section.start}</small>
            </button>
          {/each}
        </nav>

        <div class="toolbar-actions" aria-label="Presentation controls">
          <button
            type="button"
            class="theme-toggle"
            on:click={toggleThemePreference}
            aria-label={themeToggleLabel}
            title={themeToggleLabel}
          >
            {#if themePreference === 'system'}
              <Monitor size={18} strokeWidth={1.8} />
            {:else if themePreference === 'dark'}
              <Moon size={18} strokeWidth={1.8} />
            {:else}
              <Sun size={18} strokeWidth={1.8} />
            {/if}
          </button>
          <button type="button" on:click={openShareDialog} aria-label="Share current link" title="Share">
            <Share2 size={18} strokeWidth={1.8} />
          </button>
          <button type="button" on:click={() => goTo(0)} aria-label="Go to first slide" title="First slide">
            <Home size={18} strokeWidth={1.8} />
          </button>
          <button
            type="button"
            on:click={() => (overview = !overview)}
            aria-label={overview ? 'Close overview' : 'Open overview'}
            title={overview ? 'Close overview' : 'Overview'}
          >
            {#if overview}
              <X size={18} strokeWidth={1.8} />
            {:else}
              <Grid2X2 size={18} strokeWidth={1.8} />
            {/if}
          </button>
          <button type="button" on:click={toggleFullscreen} aria-label="Toggle fullscreen" title="Fullscreen">
            <Maximize2 size={18} strokeWidth={1.8} />
          </button>
          <button type="button" on:click={previous} aria-label="Previous slide" title="Previous">
            <ChevronLeft size={20} strokeWidth={1.8} />
          </button>
          <button type="button" on:click={next} aria-label="Next slide" title="Next">
            <ChevronRight size={20} strokeWidth={1.8} />
          </button>
        </div>
      </header>

      <div class="mobile-picker">
        <MonitorPlay size={18} strokeWidth={1.8} />
        <select aria-label="Choose slide" bind:value={current} on:change={(event) => goTo(+event.target.value)}>
          {#each slides as option, index}
            <option value={index}>{option.number}. {option.title}</option>
          {/each}
        </select>
      </div>

      <div class="stage-wrap" style={toneStyle}>
        {#if overview}
          <div class="overview-panel">
            {#each slides as option, index}
              <button type="button" class="overview-card" on:click={() => goTo(index)}>
                <span>{String(option.number).padStart(2, '0')}</span>
                <strong>{option.title}</strong>
                <small>{option.section}</small>
              </button>
            {/each}
          </div>
        {:else}
          {#key current}
            <article
              class={`deck-slide deck-slide-${slide.kind}${slide.emphasis ? ` deck-slide-${slide.emphasis}` : ''}`}
              class:deck-slide-finale={slide.emphasis === 'finale'}
              class:plate-motion-off={!plateContentMotion}
            >
            <div class="blueprint-grid" aria-hidden="true"></div>
            <div class="plate-mark plate-mark-nw" aria-hidden="true"></div>
            <div class="plate-mark plate-mark-ne" aria-hidden="true"></div>
            <div class="plate-mark plate-mark-sw" aria-hidden="true"></div>
            <div class="plate-mark plate-mark-se" aria-hidden="true"></div>
            <div class="slide-watermark" aria-hidden="true">STAR</div>

            <header class="slide-header">
              <div>
                <span>Plate {String(slide.number).padStart(2, '0')}</span>
                <strong>{slide.section}</strong>
              </div>
            </header>

            <div class="slide-body">
              {#if slide.kind === 'title'}
                <div class="title-layout">
                  <p class="eyebrow">{slide.eyebrow}</p>
                  <h1>{slide.title}</h1>
                  <p class="lead-title">{slide.subtitle}</p>
                  {#if slide.strapline}
                    <ul class="strapline" aria-label="Membership blueprint values">
                      {#each straplineBadges(slide.strapline) as badge}
                        <li>{badge}</li>
                      {/each}
                    </ul>
                  {/if}
                </div>
              {:else}
                <div class="content-head" class:finale-head={slide.emphasis === 'finale'}>
                  <p class="eyebrow">{String(slide.number).padStart(2, '0')} / {slide.section}</p>
                  <h2>{slide.title}</h2>
                  {#if slide.lead}
                    <p class="lead">{slide.lead}</p>
                  {/if}
                  {#if slide.body && slide.kind !== 'hero'}
                    <p class="copy">{slide.body}</p>
                  {/if}
                </div>

                {#if slide.kind === 'reflection'}
                  <div class="reflection-layout">
                    <p class="reflection-prompt">{slide.prompt}</p>
                    <div>
                      <ul class="star-list compact-list reflection-list" aria-label={slide.lead}>
                        {#each slide.items as item}
                          <li>{item}</li>
                        {/each}
                      </ul>
                    </div>
                  </div>
                {:else if slide.kind === 'contrast'}
                  <div class="comparison-grid">
                    <section class="comparison-panel quiet-panel">
                      <h3>{slide.left.title}</h3>
                      <ul class="star-list">
                        {#each slide.left.items as item}
                          <li>{item}</li>
                        {/each}
                      </ul>
                    </section>
                    <section class="comparison-panel active-panel">
                      <h3>{slide.right.title}</h3>
                      <ul class="star-list">
                        {#each slide.right.items as item}
                          <li>{item}</li>
                        {/each}
                      </ul>
                    </section>
                  </div>
                {:else if slide.kind === 'flow'}
                  {#if slide.emphasis === 'journey'}
                    <ol class="journey-route" aria-label={slide.title}>
                      <svg
                        class="journey-path"
                        viewBox="0 0 1000 150"
                        preserveAspectRatio="none"
                        aria-hidden="true"
                      >
                        <polyline points="32,58 188,98 344,48 500,90 656,42 812,88 968,54" />
                        <path d="M968 54 l-28 -16 M968 54 l-24 22" />
                      </svg>
                      {#each slide.flow as step, index}
                        <li style={`--journey-offset: ${index % 2 === 0 ? '0rem' : '3.05rem'}`}>
                          <span class="journey-node">{String(index + 1).padStart(2, '0')}</span>
                          <strong>{step}</strong>
                        </li>
                      {/each}
                    </ol>
                  {:else if slide.emphasis === 'visibility-growth'}
                    <div class="visibility-growth-layout">
                      <section class="visibility-proof">
                        <span>Visibility engine</span>
                        <strong>{slide.thesis.title}</strong>
                        <p>{slide.thesis.body}</p>
                      </section>
                      <ol class="growth-ladder" aria-label={slide.title}>
                        {#each slide.flow as step, index}
                          <li class:final-growth={step.outcome}>
                            <span>{String(index + 1).padStart(2, '0')}</span>
                            <div>
                              <small>{step.label}</small>
                              <h3>{step.title}</h3>
                              <p>{step.text}</p>
                            </div>
                          </li>
                        {/each}
                      </ol>
                    </div>
                  {:else}
                    <div class="flow-line">
                      {#each slide.flow as step}
                        <span>{step}</span>
                      {/each}
                    </div>
                  {/if}
                {:else if slide.kind === 'statement'}
                  {#if slide.emphasis === 'expertise-service'}
                    <div class="expertise-service-layout">
                      <section class="expertise-service-emblem" aria-label={slide.focus.label}>
                        <span>{slide.focus.label}</span>
                        <strong>{slide.focus.title}</strong>
                        <p>{slide.focus.body}</p>
                      </section>

                      <div class="expertise-service-panel">
                        <div class="expertise-service-statements">
                          {#each slide.statements as statement, index}
                            <article class="expertise-service-statement">
                              <span>{String(index + 1).padStart(2, '0')}</span>
                              <div>
                                <small>{statement.label}</small>
                                <p>{statement.text}</p>
                              </div>
                            </article>
                          {/each}
                        </div>

                        <ul class="expertise-service-modes" aria-label="Professional service modes">
                          {#each slide.serviceModes as mode}
                            <li>{mode}</li>
                          {/each}
                        </ul>
                      </div>
                    </div>
                  {:else if slide.emphasis === 'standard-bearer'}
                    <div class="bearer-layout">
                      <section class="bearer-emblem" aria-label="Standard bearer role">
                        <span>Visible</span>
                        <strong>Ambassador</strong>
                      </section>
                      <div class="bearer-callouts">
                        {#each slide.statements as statement, index}
                          <article class="bearer-callout">
                            <span>{String(index + 1).padStart(2, '0')} / {statement.label}</span>
                            <p>{statement.text}</p>
                          </article>
                        {/each}
                      </div>
                    </div>
                  {:else if slide.emphasis === 'finale'}
                    <div class="finale-layout" aria-label="Closing commitments">
                      <section class="finale-emblem" aria-label="STAR Region commitment">
                        <span>STAR Region Commitment</span>
                        <strong>Every chapter shines</strong>
                        <p>Every CPA proudly belongs.</p>
                      </section>

                      <ol class="finale-commitments">
                        {#each slide.statements.slice(0, -1) as statement, index}
                          <li>
                            <span>{String(index + 1).padStart(2, '0')}</span>
                            <p>{statement}</p>
                          </li>
                        {/each}
                      </ol>

                      <p class="finale-pledge">{slide.statements[slide.statements.length - 1]}</p>
                    </div>
                  {:else}
                    <div class="statement-stack" class:insight-stack={slide.emphasis === 'insight'}>
                      {#each slide.statements as statement, index}
                        {#if slide.emphasis === 'insight' && typeof statement === 'object'}
                          <article class="insight-card">
                            <span class="insight-index">{String(index + 1).padStart(2, '0')}</span>
                            <p>
                              <strong>{statement.lead}</strong>
                              <span class="insight-bridge">{statement.bridge}</span>
                              <em>{statement.focus}</em>
                            </p>
                          </article>
                        {:else}
                          <p>{statement}</p>
                        {/if}
                      {/each}
                    </div>
                  {/if}
                {:else if slide.kind === 'table'}
                  {#if slide.tableStyle === 'motivation-map'}
                    <dl class="motivation-map" aria-label={slide.title}>
                      {#each slide.rows as row, index}
                        <div class="motivation-card">
                          <span class="motivation-index">{String(index + 1).padStart(2, '0')}</span>
                          <dt>{row[0]}</dt>
                          <dd>
                            <span>{slide.columns[1]}</span>
                            {row[1]}
                          </dd>
                        </div>
                      {/each}
                    </dl>
                  {:else}
                    <div class="table-wrap">
                      <table>
                        <thead>
                          <tr>
                            {#each slide.columns as column}
                              <th>{column}</th>
                            {/each}
                          </tr>
                        </thead>
                        <tbody>
                          {#each slide.rows as row}
                            <tr>
                              {#each row as cell}
                                <td>{cell}</td>
                              {/each}
                            </tr>
                          {/each}
                        </tbody>
                      </table>
                    </div>
                  {/if}
                {:else if slide.kind === 'orbit'}
                  <div class="orbit-grid">
                    {#each slide.items as item, index}
                      <div class="orbit-node">
                        <span>{String(index + 1).padStart(2, '0')}</span>
                        <strong>{item}</strong>
                      </div>
                    {/each}
                  </div>
                {:else if slide.kind === 'acronym'}
                  <div class="acronym-grid">
                    {#each slide.items as item}
                      <div class="acronym-card">
                        <span>{item[0]}</span>
                        <div>
                          <h3>{item[1]}</h3>
                          <p>{item[2]}</p>
                        </div>
                      </div>
                    {/each}
                  </div>
                {:else if slide.kind === 'groups'}
                  {#if slide.emphasis === 'packet-trio'}
                    <div class="packet-layout">
                      {#each slide.groups as group}
                        <h3>{group.title}</h3>
                        <ol class="packet-trio" aria-label={group.title}>
                          {#each group.items as item, index}
                            <li class="packet-card">
                              <span>{String(index + 1).padStart(2, '0')}</span>
                              <strong>{item.name}</strong>
                              <p>{item.audience}</p>
                            </li>
                          {/each}
                        </ol>
                      {/each}
                    </div>
                  {:else if slide.emphasis === 'story-board'}
                    <div class="story-layout">
                      <section class="story-feature" aria-label={slide.feature.label}>
                        <span>{slide.feature.label}</span>
                        <strong>{slide.feature.title}</strong>
                        <p>{slide.feature.text}</p>
                      </section>
                      <div class="story-prompts">
                        {#each slide.groups as group}
                          <h3>{group.title}</h3>
                          <ol class="story-prompt-list" aria-label={group.title}>
                            {#each group.items as item, index}
                              <li>
                                <span>{String(index + 1).padStart(2, '0')} / {item.label}</span>
                                <strong>{item.question}</strong>
                              </li>
                            {/each}
                          </ol>
                        {/each}
                      </div>
                    </div>
                  {:else}
                    <div class="group-grid">
                      {#each slide.groups as group}
                        <section class="group-panel">
                          <h3>{group.title}</h3>
                          {#if group.featured}
                            <p class="featured-line">{group.featured}</p>
                          {/if}
                          {#if group.items?.length}
                            <ul class="star-list" class:dense-list={slide.dense}>
                              {#each group.items as item}
                                <li>{item}</li>
                              {/each}
                            </ul>
                          {/if}
                        </section>
                      {/each}
                    </div>
                  {/if}
                {:else if slide.kind === 'ratio'}
                  <div class="ratio-stack">
                    {#each slide.items as item}
                      <div class="ratio-row">
                        <span>{item[0]}</span>
                        <div>
                          <strong>{item[1]}</strong>
                          <p>{item[2]}</p>
                        </div>
                      </div>
                    {/each}
                  </div>
                {:else if slide.kind === 'cloud'}
                  <div class="value-cloud">
                    {#each slide.items as item}
                      <span>{item}</span>
                    {/each}
                  </div>
                  {#if slide.note}
                    <p class="note-line" class:takeaway-note={slide.noteStyle === 'takeaway'}>
                      {#if slide.noteSegments}
                        {#each slide.noteSegments as segment}
                          {#if segment.emphasis}
                            <strong>{segment.text}</strong>
                          {:else}
                            {segment.text}
                          {/if}
                        {/each}
                      {:else}
                        {slide.note}
                      {/if}
                    </p>
                  {/if}
                {:else if slide.kind === 'hero'}
                  {#if slide.emphasis === 'star-id-value'}
                    <div class="star-id-value-layout">
                      <section class="star-id-pass" aria-label="STAR ID value">
                        <span class="star-id-label">Member value card</span>
                        <strong>{slide.feature.title}</strong>
                        <p>{slide.feature.body}</p>
                        <div class="star-id-tags" aria-label="STAR ID qualities">
                          {#each slide.feature.tags as tag}
                            <span>{tag}</span>
                          {/each}
                        </div>
                      </section>
                      <div class="star-id-benefits">
                        {#each slide.items as item, index}
                          <article class="star-id-benefit">
                            <span>{String(index + 1).padStart(2, '0')}</span>
                            <div>
                              <small>{item.label}</small>
                              <h3>{item.title}</h3>
                              <p>{item.text}</p>
                            </div>
                          </article>
                        {/each}
                      </div>
                    </div>
                  {:else if slide.emphasis === 'invitation-campaign'}
                    <div class="invitation-layout">
                      <section class="invitation-equation" aria-label="Invitation target">
                        <div class="equation-pair">
                          <article>
                            <strong>{slide.formula[0].value}</strong>
                            <span>{slide.formula[0].label}</span>
                          </article>
                          <span class="equation-mark">x</span>
                          <article>
                            <strong>{slide.formula[1].value}</strong>
                            <span>{slide.formula[1].label}</span>
                          </article>
                        </div>
                        <div class="equation-result">
                          <span>=</span>
                          <strong>{slide.formula[2].value}</strong>
                          <p>{slide.formula[2].label}</p>
                        </div>
                      </section>
                      <ol class="invitation-steps" aria-label="Officer invitation rhythm">
                        {#each slide.items as item, index}
                          <li>
                            <span>{String(index + 1).padStart(2, '0')}</span>
                            <div>
                              <h3>{item.label}</h3>
                              <p>{item.text}</p>
                            </div>
                          </li>
                        {/each}
                      </ol>
                    </div>
                  {:else}
                    <div class="hero-statement">
                      {#if slide.metric}
                        <strong>{slide.metric}</strong>
                      {/if}
                      {#if slide.body}
                        <p>{slide.body}</p>
                      {/if}
                    </div>
                  {/if}
                {:else}
                  {#if slide.label}
                    <p class="section-label">{slide.label}</p>
                  {/if}
                  <ul
                    class="star-list"
                    class:dense-list={slide.dense}
                    class:two-columns={slide.columns === 2}
                    class:findings-list={slide.emphasis === 'findings'}
                    class:common-reasons-list={slide.emphasis === 'common-reasons'}
                  >
                    {#each slide.items as item}
                      <li>{item}</li>
                    {/each}
                  </ul>
                  {#if slide.note}
                    <p class="note-line" class:takeaway-note={slide.noteStyle === 'takeaway'}>
                      {#if slide.noteSegments}
                        {#each slide.noteSegments as segment}
                          {#if segment.emphasis}
                            <strong>{segment.text}</strong>
                          {:else}
                            {segment.text}
                          {/if}
                        {/each}
                      {:else}
                        {slide.note}
                      {/if}
                    </p>
                  {/if}
                {/if}
              {/if}
            </div>

            <footer class="slide-footer">
              <div>
                <span>Project</span>
                <strong>PICPA STAR Membership Blueprint</strong>
              </div>
              <div>
                <span>Plate title</span>
                <strong>{slide.title}</strong>
              </div>
              <div>
                <span>Section</span>
                <strong>{slide.section}</strong>
              </div>
              <div>
                <span>Sheet</span>
                <strong>{String(slide.number).padStart(2, '0')} / {slides.length}</strong>
              </div>
            </footer>
            </article>
          {/key}
        {/if}
      </div>

      <footer class="progress-area">
        <span>{String(slide.number).padStart(2, '0')}</span>
        <div class="progress-track" aria-hidden="true">
          <div style={`width: ${progress}%`}></div>
        </div>
        <span>{slides.length}</span>
      </footer>
    </section>
  </div>

  <dialog
    class="share-dialog"
    class:share-dialog-closing={shareDialogClosing}
    style={toneStyle}
    bind:this={shareDialog}
    aria-labelledby="share-dialog-title"
    on:click={handleShareDialogClick}
    on:cancel={handleShareDialogCancel}
    on:close={handleShareDialogClosed}
  >
    <div class="share-panel" on:animationend={handleSharePanelAnimationEnd}>
      <header class="share-panel-header">
        <div>
          <span class="toolbar-kicker">Share link</span>
          <h2 id="share-dialog-title">PICPA STAR Membership Blueprint</h2>
        </div>
        <button type="button" on:click={closeShareDialog} aria-label="Close share dialog" title="Close">
          <X size={18} strokeWidth={1.8} />
        </button>
      </header>

      <div class="qr-frame" aria-live="polite">
        {#if shareQrCode}
          <img src={shareQrCode} alt={`QR code for ${shareUrl}`} />
        {:else}
          <span>{shareError || 'Generating QR code'}</span>
        {/if}
      </div>

      <div class="share-link-row">
        <p>{shareUrl}</p>
        <button
          type="button"
          on:click={copyShareLink}
          aria-label={copiedShareUrl ? 'Link copied' : 'Copy link'}
          title={copiedShareUrl ? 'Copied' : 'Copy link'}
        >
          {#if copiedShareUrl}
            <Check size={18} strokeWidth={1.9} />
          {:else}
            <Copy size={18} strokeWidth={1.8} />
          {/if}
        </button>
      </div>
    </div>
  </dialog>
</main>
