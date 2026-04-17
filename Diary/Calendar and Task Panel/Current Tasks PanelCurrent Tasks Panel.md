---
cssclasses:
  - focus
  - no-space
---
```dataviewjs
await dv.view("Task Panel", {     pages: '"Diary"',    section: "## 🏷️ ToSort",    taskFiles: '"Diary/Tasks/Kanban Boards"', 	select: "Diary/Tasks/Kanban Boards/KanbanUltimate.md",    forward: true,    options: "noYear noFile noHeader todayFocus noCounters unplannedFilter" })
```
***

![[KanbanUltimate#🔁 Recurring]]  
![[KanbanUltimate#💥 Quick]]  
![[KanbanUltimate#🧱 Mid]]  
![[KanbanUltimate#🏗️ Long]]  
![[KanbanUltimate#🔥 Important<br>⚡ Urgent]]
![[KanbanUltimate#🏆 Important<br>🌱 NonUrgent]]

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