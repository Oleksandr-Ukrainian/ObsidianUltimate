---
cssclasses:
  - focus
  - no-space
---
```dataviewjs
await dv.view("Task Panel", {
    pages: '"Diary"',
    section: "## 🏷️ ToSort",
    taskFiles: '"Diary/Tasks/Kanban Boards"',
	select: "Diary/Tasks/Kanban Boards/CanbanUltimate.md",
    forward: true,
    options: "noYear noFile noHeader todayFocus noCounters unplannedFilter"
})
```
***

![[CanbanUltimate#🔁 Recurring]]
![[CanbanUltimate#💥 Quick]]
![[CanbanUltimate#🧱 Mid]]
![[CanbanUltimate#🏗️ Long]]
![[CanbanUltimate#🔥 Important<br>⚡ Urgent]]
![[CanbanUltimate#🏆 Important<br>🌱 NonUrgent]]


## [[Dated|Required]]
```tasks
not done
(due before tomorrow) OR (due today)
 hide created date
 hide backlink
 hide recurrence rule
 hide start date
 hide tags
 show tree
 hide toolbar
```