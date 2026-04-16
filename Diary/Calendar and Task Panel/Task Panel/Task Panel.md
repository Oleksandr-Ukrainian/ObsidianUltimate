```dataviewjs
await dv.view("Task Panel", {
    pages: '"Diary"',
    section: "## Sort",
    taskFiles: '"Diary/Tasks/Kanban Boards"',
	select: "Diary/Tasks/Kanban Boards/Life.md",
    forward: true,
    options: "noYear noFile noHeader todayFocus noCounters unplannedFilter"
})
```