<script>
  import WeeklyCalendar from './lib/WeeklyCalendar.svelte'
  import * as ICAL from 'ical.js'

  const weekDays = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday']
  const dayIndexByName = Object.fromEntries(weekDays.map((day, index) => [day, index]))
  const baseCaretakers = ['Avery', 'Jordan', 'Priya', 'Sam']
  const scheduleStartHour = 6
  const scheduleEndHour = 22
  const scheduleIntervalMinutes = 30
  const totalScheduleSlots = ((scheduleEndHour - scheduleStartHour) * 60) / scheduleIntervalMinutes
  const millisecondsPerDay = 24 * 60 * 60 * 1000
  const scheduleSlotMilliseconds = scheduleIntervalMinutes * 60 * 1000
  const scheduleMinTime = `${String(scheduleStartHour).padStart(2, '0')}:00:00`
  const scheduleMaxTime = `${String(scheduleEndHour).padStart(2, '0')}:00:00`
  const caretakerSwatches = [
    { background: 'oklch(0.78 0.08 155)', foreground: 'oklch(0.23 0.03 155)' },
    { background: 'oklch(0.82 0.11 50)', foreground: 'oklch(0.26 0.05 40)' },
    { background: 'oklch(0.8 0.05 240)', foreground: 'oklch(0.22 0.03 240)' },
    { background: 'oklch(0.83 0.08 90)', foreground: 'oklch(0.28 0.04 80)' },
    { background: 'oklch(0.78 0.12 15)', foreground: 'oklch(0.95 0.02 80)' },
  ]

  let loginName = ''
  let selectedRole = 'caretaker'
  let currentUser = null
  let activeView = 'login'
  let quickNote = ''
  let familyNote = ''
  let checkOutNote = ''
  let selectedScheduleDay = weekDays[0]
  let selectedScheduleCaretaker = baseCaretakers[0]
  let selectedStartSlot = hourToSlot(8)
  let selectedEndSlot = hourToSlot(12)
  let scheduleWeekStart = getStartOfWeek(new Date())
  let scheduleIcs = ''
  let calendarEvents = []

  let tasks = [
    {
      id: 'medication',
      title: 'Administer morning medication',
      dueTime: '8:00 AM',
      completed: false,
      completedBy: null,
      completedAt: null,
    },
    {
      id: 'hydration',
      title: 'Hydration check and refill water bottle',
      dueTime: '10:00 AM',
      completed: false,
      completedBy: null,
      completedAt: null,
    },
    {
      id: 'mobility',
      title: 'Assisted mobility exercise',
      dueTime: '1:00 PM',
      completed: false,
      completedBy: null,
      completedAt: null,
    },
    {
      id: 'vitals',
      title: 'Record afternoon vitals',
      dueTime: '4:30 PM',
      completed: false,
      completedBy: null,
      completedAt: null,
    },
    {
      id: 'dinner',
      title: 'Assist with dinner and evening medication',
      dueTime: '7:00 PM',
      completed: false,
      completedBy: null,
      completedAt: null,
    },
  ]

  let notes = [
    {
      id: makeId(),
      author: 'System',
      context: 'handoff',
      text: 'Night shift reported stable sleep and no overnight incidents.',
      createdAt: new Date(Date.now() - 2 * 60 * 60 * 1000).toISOString(),
    },
  ]

  let completionHistory = []
  let activityFeed = [
    {
      id: makeId(),
      message: 'Checklist initialized for today.',
      time: new Date(Date.now() - 3 * 60 * 60 * 1000).toISOString(),
    },
  ]

  let schedule = {}
  initializeSchedule()

  $: completedTasks = tasks.filter((task) => task.completed)
  $: incompleteTasks = tasks.filter((task) => !task.completed)
  $: completionRate = Math.round((completedTasks.length / tasks.length) * 100)
  $: assignedHalfHourSlots = Object.values(schedule).filter(Boolean).length
  $: availableCaretakers = getCaretakerRoster()
  $: startTimeOptions = Array.from({ length: totalScheduleSlots }, (_, index) => index)
  $: endTimeOptions = Array.from({ length: totalScheduleSlots }, (_, index) => index + 1)
  $: scheduleBlocks = weekDays.flatMap((day) => getScheduleBlocksForDay(day))
  $: scheduledBlocks = scheduleBlocks.length
  $: reservedSlots = assignedHalfHourSlots
  $: scheduleIcs = buildIcsFromBlocks(scheduleBlocks)
  $: calendarEvents = mapIcsToCalendarEvents(scheduleIcs)
  $: if (!availableCaretakers.includes(selectedScheduleCaretaker)) {
    selectedScheduleCaretaker = availableCaretakers[0] ?? ''
  }
  $: if (selectedEndSlot <= selectedStartSlot) {
    selectedEndSlot = Math.min(totalScheduleSlots, Number(selectedStartSlot) + 1)
  }

  function makeId() {
    if (typeof crypto !== 'undefined' && crypto.randomUUID) {
      return crypto.randomUUID()
    }
    return `${Date.now()}-${Math.random().toString(16).slice(2)}`
  }

  function hourToSlot(hourValue) {
    return ((hourValue - scheduleStartHour) * 60) / scheduleIntervalMinutes
  }

  function getStartOfWeek(inputDate) {
    const date = new Date(inputDate)
    const mondayOffset = (date.getDay() + 6) % 7
    date.setDate(date.getDate() - mondayOffset)
    date.setHours(0, 0, 0, 0)
    return date
  }

  function startOfDay(date) {
    const dayDate = new Date(date)
    dayDate.setHours(0, 0, 0, 0)
    return dayDate
  }

  function createEmptySchedule() {
    const next = {}
    for (const day of weekDays) {
      for (let slotIndex = 0; slotIndex < totalScheduleSlots; slotIndex += 1) {
        next[scheduleKey(day, slotIndex)] = ''
      }
    }
    return next
  }

  function initializeSchedule() {
    const seed = createEmptySchedule()
    setScheduleRange(seed, 'Monday', hourToSlot(8), hourToSlot(12), 'Avery')
    setScheduleRange(seed, 'Monday', hourToSlot(17), hourToSlot(21), 'Sam')
    setScheduleRange(seed, 'Tuesday', hourToSlot(12), hourToSlot(16), 'Jordan')
    setScheduleRange(seed, 'Friday', hourToSlot(8), hourToSlot(12), 'Priya')
    schedule = seed
  }

  function setScheduleRange(targetSchedule, day, startSlot, endSlot, assignee) {
    for (let slotIndex = startSlot; slotIndex < endSlot; slotIndex += 1) {
      targetSchedule[scheduleKey(day, slotIndex)] = assignee
    }
  }

  function scheduleKey(day, slotIndex) {
    return `${day}-${slotIndex}`
  }

  function getCaretakerRoster() {
    const roster = [...baseCaretakers]
    if (
      currentUser &&
      currentUser.role === 'caretaker' &&
      currentUser.name &&
      !roster.includes(currentUser.name)
    ) {
      roster.push(currentUser.name)
    }
    return roster
  }

  function getScheduleBlocksForDay(day) {
    const blocks = []
    let activeCaretaker = ''
    let activeStartSlot = 0

    for (let slotIndex = 0; slotIndex <= totalScheduleSlots; slotIndex += 1) {
      const assignee = slotIndex < totalScheduleSlots ? schedule[scheduleKey(day, slotIndex)] || '' : ''
      if (assignee !== activeCaretaker) {
        if (activeCaretaker) {
          blocks.push({
            id: `${day}-${activeStartSlot}-${slotIndex}-${activeCaretaker}`,
            day,
            caretaker: activeCaretaker,
            startSlot: activeStartSlot,
            endSlot: slotIndex,
          })
        }
        activeCaretaker = assignee
        activeStartSlot = slotIndex
      }
    }

    return blocks
  }

  function slotToDate(day, slotIndex) {
    const dayIndex = dayIndexByName[day]
    const date = new Date(scheduleWeekStart)
    date.setDate(date.getDate() + dayIndex)

    const totalMinutes = scheduleStartHour * 60 + slotIndex * scheduleIntervalMinutes
    const hour = Math.floor(totalMinutes / 60)
    const minute = totalMinutes % 60
    date.setHours(hour, minute, 0, 0)
    return date
  }

  function getSchedulePointFromDate(date, roundMode = 'floor') {
    const dayOffset = Math.floor((startOfDay(date).getTime() - scheduleWeekStart.getTime()) / millisecondsPerDay)
    if (dayOffset < 0 || dayOffset >= weekDays.length) {
      return null
    }

    const totalMinutes = date.getHours() * 60 + date.getMinutes()
    const minutesFromStart = totalMinutes - scheduleStartHour * 60
    const slotFloat = minutesFromStart / scheduleIntervalMinutes
    const roundedSlot = roundMode === 'ceil' ? Math.ceil(slotFloat) : Math.floor(slotFloat)
    const clampedSlot = Math.max(0, Math.min(totalScheduleSlots, roundedSlot))
    return {
      day: weekDays[dayOffset],
      slotIndex: clampedSlot,
    }
  }

  function getCaretakerSwatch(caretaker) {
    const roster = getCaretakerRoster()
    const caretIndex = roster.indexOf(caretaker)
    return caretakerSwatches[(caretIndex < 0 ? 0 : caretIndex) % caretakerSwatches.length]
  }

  function formatScheduleTime(slotIndex) {
    const normalizedSlot = Number(slotIndex)
    const totalMinutes = scheduleStartHour * 60 + normalizedSlot * scheduleIntervalMinutes
    const hour = Math.floor(totalMinutes / 60)
    const minute = totalMinutes % 60
    const time = new Date(2020, 0, 1, hour, minute)
    return time.toLocaleTimeString([], { hour: 'numeric', minute: '2-digit' })
  }

  function formatScheduleRange(startSlot, endSlot) {
    return `${formatScheduleTime(startSlot)} - ${formatScheduleTime(endSlot)}`
  }

  function formatCalendarDateRange(start, end) {
    return `${start.toLocaleDateString([], { weekday: 'long' })} ${start.toLocaleTimeString([], {
      hour: 'numeric',
      minute: '2-digit',
    })} - ${end.toLocaleTimeString([], { hour: 'numeric', minute: '2-digit' })}`
  }

  function buildIcsUid(block) {
    const caretakerSlug = block.caretaker.trim().toLowerCase().replace(/\s+/g, '-')
    return `schedule-${block.day}-${block.startSlot}-${block.endSlot}-${caretakerSlug}@caretaker-scheduler.local`
  }

  function buildIcsFromBlocks(blocks) {
    const calendar = new ICAL.Component(['vcalendar', [], []])
    calendar.addPropertyWithValue('prodid', '-//Caretaker Scheduler//EN')
    calendar.addPropertyWithValue('version', '2.0')
    calendar.addPropertyWithValue('calscale', 'GREGORIAN')

    for (const block of blocks) {
      const eventComponent = new ICAL.Component('vevent')
      eventComponent.addPropertyWithValue('uid', buildIcsUid(block))
      eventComponent.addPropertyWithValue('summary', block.caretaker)
      eventComponent.addPropertyWithValue('description', `Caretaker assignment for ${block.day}`)
      eventComponent.addPropertyWithValue('dtstamp', ICAL.Time.fromJSDate(new Date(), true))
      eventComponent.addPropertyWithValue('dtstart', ICAL.Time.fromJSDate(slotToDate(block.day, block.startSlot), false))
      eventComponent.addPropertyWithValue('dtend', ICAL.Time.fromJSDate(slotToDate(block.day, block.endSlot), false))
      calendar.addSubcomponent(eventComponent)
    }

    return calendar.toString()
  }

  function parseIcsEvents(icsText) {
    if (!icsText.trim()) return []

    try {
      const parsed = ICAL.parse(icsText)
      const calendar = new ICAL.Component(parsed)
      const eventComponents = calendar.getAllSubcomponents('vevent')
      return eventComponents
        .map((eventComponent) => {
          const event = new ICAL.Event(eventComponent)
          const start = event.startDate?.toJSDate()
          const end = event.endDate?.toJSDate()
          if (!start || !end || end <= start) return null
          return {
            id: event.uid || makeId(),
            caretaker: (event.summary || 'Unassigned').trim() || 'Unassigned',
            start,
            end,
          }
        })
        .filter(Boolean)
    } catch {
      return []
    }
  }

  function mapIcsToCalendarEvents(icsText) {
    return parseIcsEvents(icsText).map((eventEntry) => {
      const swatch = getCaretakerSwatch(eventEntry.caretaker)
      return {
        id: eventEntry.id,
        title: eventEntry.caretaker,
        start: eventEntry.start,
        end: eventEntry.end,
        backgroundColor: swatch.background,
        borderColor: swatch.background,
        textColor: swatch.foreground,
        extendedProps: {
          caretaker: eventEntry.caretaker,
        },
      }
    })
  }

  function hydrateScheduleFromIcs(icsText, sourceLabel) {
    const parsedEvents = parseIcsEvents(icsText)
    if (!parsedEvents.length) return false

    const importedWeekStart = getStartOfWeek(parsedEvents[0].start)
    const nextSchedule = createEmptySchedule()

    for (const eventEntry of parsedEvents) {
      for (
        let cursor = new Date(eventEntry.start);
        cursor < eventEntry.end;
        cursor = new Date(cursor.getTime() + scheduleSlotMilliseconds)
      ) {
        const point = getSchedulePointFromDate(
          new Date(cursor.getTime() - (importedWeekStart.getTime() - scheduleWeekStart.getTime()))
        )
        if (!point || point.slotIndex >= totalScheduleSlots) continue
        nextSchedule[scheduleKey(point.day, point.slotIndex)] = eventEntry.caretaker
      }
    }

    scheduleWeekStart = importedWeekStart
    schedule = nextSchedule

    if (currentUser) {
      addActivity(
        `${currentUser.name} imported ${parsedEvents.length} schedule event${parsedEvents.length === 1 ? '' : 's'} from ${sourceLabel}.`
      )
    }

    return true
  }

  function downloadScheduleIcs() {
    const blob = new Blob([scheduleIcs], { type: 'text/calendar;charset=utf-8' })
    const objectUrl = URL.createObjectURL(blob)
    const link = document.createElement('a')
    const mondayLabel = scheduleWeekStart.toISOString().slice(0, 10)
    link.href = objectUrl
    link.download = `caretaker-schedule-${mondayLabel}.ics`
    link.click()
    URL.revokeObjectURL(objectUrl)
  }

  async function importScheduleFile(event) {
    const file = event.currentTarget.files?.[0]
    if (!file) return
    const fileContent = await file.text()
    const didImport = hydrateScheduleFromIcs(fileContent, '.ics upload')
    if (!didImport && currentUser) {
      addActivity(`${currentUser.name} attempted to import an invalid .ics schedule file.`)
    }
    event.currentTarget.value = ''
  }

  function handleCalendarRangeSelect({ start, end }) {
    if (!(start instanceof Date) || !(end instanceof Date)) return
    prefillScheduleRangeFromCalendar(start, end)
  }

  function handleCalendarEventClick(calendarEvent) {
    clearScheduleEvent(calendarEvent)
  }

  function prefillScheduleRangeFromCalendar(startDate, endDate) {
    const startPoint = getSchedulePointFromDate(startDate, 'floor')
    const endPoint = getSchedulePointFromDate(endDate, 'ceil')
    if (!startPoint || !endPoint) return

    selectedScheduleDay = startPoint.day
    selectedStartSlot = Math.min(totalScheduleSlots - 1, startPoint.slotIndex)
    if (startPoint.day === endPoint.day) {
      selectedEndSlot = Math.max(selectedStartSlot + 1, endPoint.slotIndex)
    } else {
      selectedEndSlot = totalScheduleSlots
    }
  }

  function clearScheduleEvent(calendarEvent) {
    if (!currentUser || !calendarEvent.start || !calendarEvent.end) return

    const nextSchedule = { ...schedule }
    let hasChanges = false

    for (
      let cursor = new Date(calendarEvent.start);
      cursor < calendarEvent.end;
      cursor = new Date(cursor.getTime() + scheduleSlotMilliseconds)
    ) {
      const point = getSchedulePointFromDate(cursor, 'floor')
      if (!point || point.slotIndex >= totalScheduleSlots) continue
      const key = scheduleKey(point.day, point.slotIndex)
      if (!nextSchedule[key]) continue
      nextSchedule[key] = ''
      hasChanges = true
    }

    if (!hasChanges) return
    schedule = nextSchedule

    const caretaker = calendarEvent.extendedProps?.caretaker || calendarEvent.title || 'a caretaker'
    addActivity(`${currentUser.name} removed ${caretaker}'s block on ${formatCalendarDateRange(calendarEvent.start, calendarEvent.end)}.`)
  }

  function formatDateTime(value) {
    return new Date(value).toLocaleString([], {
      month: 'short',
      day: 'numeric',
      hour: 'numeric',
      minute: '2-digit',
    })
  }

  function addActivity(message) {
    activityFeed = [{ id: makeId(), message, time: new Date().toISOString() }, ...activityFeed].slice(0, 15)
  }

  function addNote(rawText, context) {
    const text = rawText.trim()
    if (!text || !currentUser) return false

    notes = [
      {
        id: makeId(),
        author: currentUser.name,
        context,
        text,
        createdAt: new Date().toISOString(),
      },
      ...notes,
    ]

    addActivity(`${currentUser.name} added a ${context} note.`)
    return true
  }

  function login() {
    const cleanName = loginName.trim()
    if (!cleanName) return

    currentUser = {
      name: cleanName,
      role: selectedRole,
      signedInAt: new Date().toISOString(),
    }

    activeView = selectedRole === 'caretaker' ? 'checkin' : 'summary'
    addActivity(`${cleanName} signed in as ${selectedRole === 'caretaker' ? 'Caretaker' : 'Family Coordinator'}.`)
  }

  function logout() {
    if (currentUser) {
      addActivity(`${currentUser.name} signed out.`)
    }
    currentUser = null
    loginName = ''
    selectedRole = 'caretaker'
    activeView = 'login'
    quickNote = ''
    familyNote = ''
    checkOutNote = ''
  }

  function toggleTask(taskId, source) {
    if (!currentUser) return

    let updatedTask = null
    tasks = tasks.map((task) => {
      if (task.id !== taskId) return task

      const willBeComplete = !task.completed
      updatedTask = {
        ...task,
        completed: willBeComplete,
        completedBy: willBeComplete ? currentUser.name : null,
        completedAt: willBeComplete ? new Date().toISOString() : null,
      }
      return updatedTask
    })

    if (!updatedTask) return

    if (updatedTask.completed) {
      completionHistory = [
        {
          id: makeId(),
          taskTitle: updatedTask.title,
          by: updatedTask.completedBy,
          at: updatedTask.completedAt,
          source,
        },
        ...completionHistory,
      ].slice(0, 12)
    }

    addActivity(
      `${currentUser.name} marked "${updatedTask.title}" ${updatedTask.completed ? 'complete' : 'incomplete'} in ${source}.`
    )
  }

  function saveQuickNote() {
    if (addNote(quickNote, 'check-in')) {
      quickNote = ''
    }
  }

  function saveFamilyNote() {
    if (addNote(familyNote, 'family')) {
      familyNote = ''
    }
  }

  function completeCheckout() {
    const saved = addNote(checkOutNote, 'check-out')
    checkOutNote = ''
    addActivity(`${currentUser.name} completed check-out${saved ? ' with a patient note.' : '.'}`)
  }

  function updateScheduleRange() {
    const startSlot = Number(selectedStartSlot)
    const endSlot = Number(selectedEndSlot)
    if (!currentUser || endSlot <= startSlot) return

    const nextSchedule = { ...schedule }
    let hasChanges = false

    for (let slotIndex = startSlot; slotIndex < endSlot; slotIndex += 1) {
      const key = scheduleKey(selectedScheduleDay, slotIndex)
      if (nextSchedule[key] !== selectedScheduleCaretaker) {
        nextSchedule[key] = selectedScheduleCaretaker
        hasChanges = true
      }
    }

    if (!hasChanges) return

    schedule = nextSchedule

    if (selectedScheduleCaretaker) {
      addActivity(
        `${currentUser.name} updated schedule: ${selectedScheduleCaretaker} on ${selectedScheduleDay} ${formatScheduleRange(startSlot, endSlot)}.`
      )
    } else {
      addActivity(
        `${currentUser.name} cleared schedule on ${selectedScheduleDay} ${formatScheduleRange(startSlot, endSlot)}.`
      )
    }
  }
</script>

{#if !currentUser}
  <main class="login-screen">
    <section class="panel login-panel">
      <h1>Patient Care Collaborative Hub</h1>
      <p class="lead">
        Sign in to manage checklist completion, schedule assignments, and care notes for the family and caregiving team.
      </p>

      <label for="name">Your name</label>
      <input id="name" bind:value={loginName} placeholder="Enter your full name" />

      <fieldset>
        <legend>Role</legend>
        <label class="radio">
          <input type="radio" bind:group={selectedRole} value="caretaker" />
          Caretaker
        </label>
        <label class="radio">
          <input type="radio" bind:group={selectedRole} value="family" />
          Family Coordinator
        </label>
      </fieldset>

      <button type="button" on:click={login}>Continue to check-in</button>
    </section>
  </main>
{:else}
  <main class="app-shell">
    <header class="topbar">
      <div>
        <h1>Patient Checklist & Scheduler</h1>
        <p>
          Signed in as <strong>{currentUser.name}</strong> ({currentUser.role === 'caretaker'
            ? 'Caretaker'
            : 'Family Coordinator'})
        </p>
      </div>

      <div class="actions">
        {#if currentUser.role === 'caretaker'}
          <button
            type="button"
            class:active={activeView === 'checkin'}
            on:click={() => {
              activeView = 'checkin'
            }}
          >
            Check-in
          </button>
          <button
            type="button"
            class:active={activeView === 'schedule'}
            on:click={() => {
              activeView = 'schedule'
            }}
          >
            Schedule
          </button>
          <button
            type="button"
            class:active={activeView === 'checkout'}
            on:click={() => {
              activeView = 'checkout'
            }}
          >
            Check-out
          </button>
        {:else}
          <button
            type="button"
            class:active={activeView === 'summary'}
            on:click={() => {
              activeView = 'summary'
            }}
          >
            Family Summary
          </button>
        {/if}
        <button type="button" class="ghost" on:click={logout}>Sign out</button>
      </div>
    </header>

    <section class="layout">
      <article class="panel main-panel">
        {#if activeView === 'checkin' && currentUser.role === 'caretaker'}
          <h2>Caretaker Check-in</h2>
          <p class="lead">
            Focus on outstanding tasks first, then leave a handoff note for the team and family.
          </p>

          <div class="summary-grid">
            <article class="stat">
              <h3>Outstanding</h3>
              <p>{incompleteTasks.length}</p>
            </article>
            <article class="stat">
              <h3>Completed</h3>
              <p>{completedTasks.length}</p>
            </article>
            <article class="stat">
              <h3>Completion Rate</h3>
              <p>{completionRate}%</p>
            </article>
          </div>

          <section>
            <h3>Outstanding checklist tasks</h3>
            {#if incompleteTasks.length === 0}
              <p class="good">All checklist tasks are complete.</p>
            {:else}
              <ul class="task-list">
                {#each incompleteTasks as task (task.id)}
                  <li>
                    <div>
                      <p class="task-title">{task.title}</p>
                      <p class="meta">Due {task.dueTime}</p>
                    </div>
                    <button type="button" on:click={() => toggleTask(task.id, 'check-in')}>Mark complete</button>
                  </li>
                {/each}
              </ul>
            {/if}
          </section>

          <section>
            <h3>All checklist items</h3>
            <ul class="task-list condensed">
              {#each tasks as task (task.id)}
                <li>
                  <label>
                    <input
                      type="checkbox"
                      checked={task.completed}
                      on:change={() => toggleTask(task.id, 'check-in')}
                    />
                    <span>{task.title}</span>
                  </label>
                  <p class="meta">
                    {#if task.completed}
                      Completed by {task.completedBy} at {formatDateTime(task.completedAt)}
                    {:else}
                      Due {task.dueTime}
                    {/if}
                  </p>
                </li>
              {/each}
            </ul>
          </section>

          <section>
            <h3>Check-in note</h3>
            <textarea bind:value={quickNote} rows="3" placeholder="Add updates for the care team and family."></textarea>
            <button type="button" on:click={saveQuickNote}>Save note</button>
          </section>
        {:else if activeView === 'schedule' && currentUser.role === 'caretaker'}
          <h2>Caretaker Weekly Schedule</h2>
          <p class="lead">
            The scheduler is stored as iCalendar (.ics) data and visualized in a weekly time-grid calendar. Select on
            the calendar to prefill a block, then apply changes below.
          </p>
          <p class="meta">
            {assignedHalfHourSlots} of {totalScheduleSlots * weekDays.length} half-hour slots assigned across
            {scheduledBlocks} schedule block{scheduledBlocks === 1 ? '' : 's'}.
          </p>

          <section class="schedule-form">
            <h3>Update schedule block</h3>
            <div class="schedule-controls">
              <label>
                Day
                <select bind:value={selectedScheduleDay}>
                  {#each weekDays as day}
                    <option value={day}>{day}</option>
                  {/each}
                </select>
              </label>
              <label>
                Caretaker
                <select bind:value={selectedScheduleCaretaker}>
                  <option value="">Clear assignment</option>
                  {#each availableCaretakers as caretaker}
                    <option value={caretaker}>{caretaker}</option>
                  {/each}
                </select>
              </label>
              <label>
                Start
                <select bind:value={selectedStartSlot}>
                  {#each startTimeOptions as slot}
                    <option value={slot}>{formatScheduleTime(slot)}</option>
                  {/each}
                </select>
              </label>
              <label>
                End
                <select bind:value={selectedEndSlot}>
                  {#each endTimeOptions as slot}
                    {#if slot > selectedStartSlot}
                      <option value={slot}>{formatScheduleTime(slot)}</option>
                    {/if}
                  {/each}
                </select>
              </label>
            </div>
            <div class="schedule-actions">
              <button type="button" on:click={updateScheduleRange}>Apply block</button>
              <button type="button" class="ghost" on:click={downloadScheduleIcs}>Export .ics</button>
              <label class="file-input">
                <span>Import .ics</span>
                <input type="file" accept=".ics,text/calendar" on:change={importScheduleFile} />
              </label>
              <p class="meta">
                Selected: {selectedScheduleDay} {formatScheduleRange(selectedStartSlot, selectedEndSlot)}
              </p>
            </div>
          </section>

          <section class="schedule-legend">
            {#each availableCaretakers as caretaker}
              {@const swatch = getCaretakerSwatch(caretaker)}
              <span
                class="legend-item"
                style={`--legend-bg:${swatch.background}; --legend-fg:${swatch.foreground};`}
              >
                {caretaker}
              </span>
            {/each}
          </section>

          <div class="schedule-wrap">
            <WeeklyCalendar
              events={calendarEvents}
              weekStart={scheduleWeekStart}
              slotMinTime={scheduleMinTime}
              slotMaxTime={scheduleMaxTime}
              onSelectRange={prefillScheduleRangeFromCalendar}
              onEventClick={clearScheduleEvent}
            />
          </div>
        {:else if activeView === 'checkout' && currentUser.role === 'caretaker'}
          <h2>Caretaker Check-out</h2>
          <p class="lead">
            Confirm all final checklist updates, then submit your end-of-shift note for continuity.
          </p>

          <section>
            <h3>Final checklist confirmation</h3>
            <ul class="task-list condensed">
              {#each tasks as task (task.id)}
                <li>
                  <label>
                    <input
                      type="checkbox"
                      checked={task.completed}
                      on:change={() => toggleTask(task.id, 'check-out')}
                    />
                    <span>{task.title}</span>
                  </label>
                  <p class="meta">
                    {#if task.completed}
                      Completed by {task.completedBy} at {formatDateTime(task.completedAt)}
                    {:else}
                      Still pending
                    {/if}
                  </p>
                </li>
              {/each}
            </ul>
          </section>

          <section>
            <h3>End-of-shift patient note</h3>
            <textarea
              bind:value={checkOutNote}
              rows="4"
              placeholder="Document symptoms, appetite, mood, follow-up needs, and handoff instructions."
            ></textarea>
            <button type="button" on:click={completeCheckout}>Submit check-out</button>
          </section>
        {:else}
          <h2>Family Coordinator Summary</h2>
          <p class="lead">Review completed vs incomplete tasks, recent activity, and notes from the caregiver team.</p>

          <div class="summary-grid">
            <article class="stat">
              <h3>Completed Tasks</h3>
              <p>{completedTasks.length}</p>
            </article>
            <article class="stat">
              <h3>Incomplete Tasks</h3>
              <p>{incompleteTasks.length}</p>
            </article>
            <article class="stat">
              <h3>Assigned Slots</h3>
              <p>{reservedSlots}</p>
            </article>
          </div>

          <section class="split">
            <div>
              <h3>Complete</h3>
              {#if completedTasks.length === 0}
                <p class="meta">No items completed yet.</p>
              {:else}
                <ul class="task-list condensed">
                  {#each completedTasks as task (task.id)}
                    <li>
                      <p class="task-title">{task.title}</p>
                      <p class="meta">Completed by {task.completedBy} at {formatDateTime(task.completedAt)}</p>
                    </li>
                  {/each}
                </ul>
              {/if}
            </div>
            <div>
              <h3>Incomplete</h3>
              {#if incompleteTasks.length === 0}
                <p class="good">All tasks complete.</p>
              {:else}
                <ul class="task-list condensed">
                  {#each incompleteTasks as task (task.id)}
                    <li>
                      <p class="task-title">{task.title}</p>
                      <p class="meta">Due {task.dueTime}</p>
                    </li>
                  {/each}
                </ul>
              {/if}
            </div>
          </section>

          <section>
            <h3>Family note</h3>
            <textarea
              bind:value={familyNote}
              rows="3"
              placeholder="Add follow-up requests or clarification for upcoming shifts."
            ></textarea>
            <button type="button" on:click={saveFamilyNote}>Save note</button>
          </section>
        {/if}
      </article>

      <aside class="panel side-panel">
        <section>
          <h3>Recent care notes</h3>
          <ul class="activity">
            {#if notes.length === 0}
              <li class="meta">No notes yet.</li>
            {:else}
              {#each notes.slice(0, 8) as note (note.id)}
                <li>
                  <p>{note.text}</p>
                  <p class="meta">
                    {note.author} ({note.context}) &middot; {formatDateTime(note.createdAt)}
                  </p>
                </li>
              {/each}
            {/if}
          </ul>
        </section>

        <section>
          <h3>Recent completion events</h3>
          <ul class="activity">
            {#if completionHistory.length === 0}
              <li class="meta">No completion events logged yet.</li>
            {:else}
              {#each completionHistory.slice(0, 6) as event (event.id)}
                <li>
                  <p>{event.taskTitle}</p>
                  <p class="meta">
                    Completed by {event.by} in {event.source} &middot; {formatDateTime(event.at)}
                  </p>
                </li>
              {/each}
            {/if}
          </ul>
        </section>

        <section>
          <h3>Activity feed</h3>
          <ul class="activity">
            {#each activityFeed as item (item.id)}
              <li>
                <p>{item.message}</p>
                <p class="meta">{formatDateTime(item.time)}</p>
              </li>
            {/each}
          </ul>
        </section>
      </aside>
    </section>
  </main>
{/if}
