<script>
  import { onMount, onDestroy } from 'svelte'
  import { Calendar } from '@fullcalendar/core'
  import interactionPlugin from '@fullcalendar/interaction'
  import timeGridPlugin from '@fullcalendar/timegrid'

  export let events = []
  export let weekStart = new Date()
  export let slotMinTime = '06:00:00'
  export let slotMaxTime = '22:00:00'
  export let onSelectRange = null
  export let onEventClick = null

  let calendarElement
  let calendar

  function notifySelect(selectionInfo) {
    if (onSelectRange) {
      onSelectRange(selectionInfo.start, selectionInfo.end)
    }
  }

  function notifyEventClick(clickInfo) {
    if (onEventClick) {
      onEventClick({
        id: clickInfo.event.id,
        title: clickInfo.event.title,
        start: clickInfo.event.start,
        end: clickInfo.event.end,
        extendedProps: clickInfo.event.extendedProps,
      })
    }
  }

  onMount(() => {
    calendar = new Calendar(calendarElement, {
      plugins: [timeGridPlugin, interactionPlugin],
      initialView: 'timeGridWeek',
      headerToolbar: false,
      allDaySlot: false,
      nowIndicator: true,
      selectable: true,
      selectMirror: true,
      editable: false,
      eventOverlap: true,
      eventDisplay: 'block',
      dayHeaderFormat: { weekday: 'long' },
      slotDuration: '00:30:00',
      slotLabelInterval: '01:00:00',
      slotMinTime,
      slotMaxTime,
      firstDay: 1,
      height: 'auto',
      initialDate: weekStart,
      events,
      select: notifySelect,
      eventClick: notifyEventClick,
    })

    calendar.render()
  })

  $: if (calendar) {
    calendar.setOption('events', events)
  }

  $: if (calendar) {
    calendar.setOption('slotMinTime', slotMinTime)
    calendar.setOption('slotMaxTime', slotMaxTime)
  }

  $: if (calendar && weekStart) {
    calendar.gotoDate(weekStart)
  }

  onDestroy(() => {
    if (calendar) {
      calendar.destroy()
    }
  })
</script>

<div class="weekly-calendar-host" bind:this={calendarElement}></div>
