---
cssclasses:
  - dashboard
  - stretch
  - no-scroll
---
```dataviewjs
await dv.view("Calendar", {
	pages: "",
	view: "month", 
	firstDayOfWeek: "1", 
	options: "style1 filter noCellNameEvent noWeekNr noFilename lineClamp3 noLayer", css: " .grid { height: 700px !important} "
})
```