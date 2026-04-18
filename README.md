# Caretaker Scheduler

A Svelte app for collaborative patient care between caretakers and family coordinators.

## Features

- Role-based login (Caretaker or Family Coordinator)
- Caretaker check-in page with:
  - Highlighted outstanding checklist tasks
  - Task completion tracking with who completed each task and when
  - Check-in note capture
- Caretaker check-out page with:
  - Final checklist completion updates
  - End-of-shift patient notes
- Caretaker weekly schedule page:
  - Morning/Afternoon/Evening assignment reservations across the week
- Family coordinator summary view:
  - Completed vs incomplete checklist overview
  - Assigned schedule slot summary
  - Family follow-up notes
- Shared activity and note feed for visibility across roles

## Run locally

```bash
npm install
npm run dev
```

Then open the printed local URL in your browser.

## Build

```bash
npm run build
```