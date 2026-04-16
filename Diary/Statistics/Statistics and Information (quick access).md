---
cssclasses:
  - focus
---
```dataviewjs
const pluginIds = ["tasks", "obsidian-tasks-plugin", "obsidian-tasks"];
let TasksPlugin = null;
for (const id of pluginIds) {
    if (app.plugins.plugins[id]) {
        TasksPlugin = app.plugins.plugins[id];
        break;
    }
}
if (!TasksPlugin) {
    dv.span("❌ Tasks plugin is not active");
} else {
    const now = new Date();
    const today = new Date(now.getFullYear(), now.getMonth(), now.getDate());
    
    // === MAIN BUG FIX ===
    // Correct calculation of the date 'one month ago' (same day of the previous month,
    // with correct handling of short months — e.g., March 31 → February 28).
    // Old logic broke due to date overflow when using setMonth().
    const daysInPrevMonth = new Date(today.getFullYear(), today.getMonth(), 0).getDate();
    const targetDay = Math.min(today.getDate(), daysInPrevMonth);
    const lastMonth = new Date(today.getFullYear(), today.getMonth() - 1, targetDay);
    
    const getDateKey = date => `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}-${String(date.getDate()).padStart(2, '0')}`;
    const todayKey = getDateKey(today);
    const lastMonthKey = getDateKey(lastMonth);
    
    let allTasks = [];
    if (typeof TasksPlugin.getTasks === 'function') {
        allTasks = TasksPlugin.getTasks();
    } else if (TasksPlugin?.index?.tasks) {
        allTasks = Object.values(TasksPlugin.index.tasks);
    } else if (TasksPlugin?.cache) {
        allTasks = TasksPlugin.cache;
    }
    
    const counts = {};
    const completedDates = new Set();
    let maxTasksDayCount = 0;
    const tagCounts = new Map();
    const weekStartDate = new Date(today);
    weekStartDate.setDate(today.getDate() - 6);
    const weekStartKey = getDateKey(weekStartDate);
    
    for (const task of allTasks) {
        const doneDate = task.doneDate || task.completionDate || (task.completed ? task.completion : null);
        const status = task.status?.symbol || task.status;
        if ((status === 'x' || status === 'completed') && doneDate) {
            const d = doneDate instanceof window.moment ? doneDate.toDate() : new Date(doneDate);
            if (isNaN(d.getTime())) continue;
            const dateKey = getDateKey(d);
            counts[dateKey] = (counts[dateKey] || 0) + 1;
            if (counts[dateKey] > maxTasksDayCount) maxTasksDayCount = counts[dateKey];
            completedDates.add(dateKey);
            if (dateKey >= weekStartKey) {
                const tags = task.tags || [];
                if (Array.isArray(tags)) {
                    for (const tag of tags) {
                        tagCounts.set(tag, (tagCounts.get(tag) || 0) + 1);
                    }
                }
            }
        }
    }
    
    const sortedDates = Array.from(completedDates).sort();
    
    let mostPopularTag = "";
    let mostPopularTagCount = 0;
    if (tagCounts.size > 0) {
        let maxCount = 0;
        for (const [tag, count] of tagCounts) {
            if (count > maxCount) {
                maxCount = count;
                mostPopularTag = tag;
                mostPopularTagCount = count;
            }
        }
    }
    
    let currentStreak = 0;
    let bestStreak = 0;
    if (sortedDates.length > 0) {
        // Current streak (from today backwards)
        let checkDate = new Date(today);
        for (let i = 0; i < 365; i++) {
            const checkKey = getDateKey(checkDate);
            if (counts[checkKey] && counts[checkKey] > 0) {
                currentStreak++;
                checkDate.setDate(checkDate.getDate() - 1);
            } else {
                break;
            }
        }
        
        // Best streak of all time
        let tempStreak = 1;
        bestStreak = 1;
        for (let i = 1; i < sortedDates.length; i++) {
            const prevDate = new Date(sortedDates[i-1]);
            const currDate = new Date(sortedDates[i]);
            const diffTime = currDate - prevDate;
            const diffDays = Math.round(diffTime / (1000 * 60 * 60 * 24));
            if (diffDays === 1) {
                tempStreak++;
                bestStreak = Math.max(bestStreak, tempStreak);
            } else {
                tempStreak = 1;
            }
        }
    }
    
    const completedToday = counts[todayKey] || 0;
    const completedLastMonth = counts[lastMonthKey] || 0;
    const totalCompleted = Object.values(counts).reduce((a, b) => a + b, 0);
    const daysWithTasks = sortedDates.length;
    const totalDays = daysWithTasks > 0 ? 
        Math.ceil((today - new Date(sortedDates[0])) / (1000 * 60 * 60 * 24)) + 1 : 0;
    const avgPerDay = daysWithTasks > 0 ? (totalCompleted / daysWithTasks).toFixed(1) : "0.0";
    
    const weekData = [];
    for (let i = 0; i < 7; i++) {
        const d = new Date(today);
        d.setDate(today.getDate() - i);
        const key = getDateKey(d);
        weekData.unshift(counts[key] || 0);
    }
    const weekAvg = weekData.reduce((a, b) => a + b, 0) / 7;
    const currentWeekTotal = weekData.reduce((a, b) => a + b, 0);
    
    let trendSymbol = "—";
    let trendColor = "#999";
    let diffText = "0";
    let diffColor = "#999";
    let diffValue = 0;
    if (completedLastMonth > 0) {
        const diff = completedToday - completedLastMonth;
        diffValue = diff;
        const pct = Math.abs(Math.round((diff / completedLastMonth) * 100));
        if (diff > 0) {
            trendSymbol = `↑+${pct}%`;
            trendColor = "#4CAF50";
            diffText = `${diff} more`;
            diffColor = "#4CAF50";
        } else if (diff < 0) {
            trendSymbol = `↓${pct}%`;
            trendColor = "#FF9800";
            diffText = `${Math.abs(diff)} less`;
            diffColor = "#FF9800";
        } else {
            trendSymbol = "→0%";
            trendColor = "#4FC3F7";
            diffText = "0";
            diffColor = "#4FC3F7";
        }
    } else if (completedToday > 0) {
        trendSymbol = "🆕";
        trendColor = "#4CAF50";
        diffText = `${completedToday} more`;
        diffColor = "#4CAF50";
        diffValue = completedToday;
    }
    
    const panelStyle = `
        background: var(--background-primary);
        border: 1px solid var(--background-modifier-border);
        border-radius: 8px;
        padding: 10px;
        margin: 0.8em auto;
        max-width: 650px;
        box-shadow: 0 1px 4px rgba(0,0,0,0.03);
        font-family: var(--font-ui);
        position: relative;
        overflow: hidden;
    `;
    const createCardStyle = (color) => `
        width: 32px;
        height: 32px;
        background: ${color}10;
        border-radius: 6px;
        display: flex;
        align-items: center;
        justify-content: center;
        margin: 0 auto 4px;
        color: ${color};
        font-size: 16px;
        font-weight: 600;
    `;
    const createStatCard = (icon, value, label, color = "var(--text-normal)", subLabel = null, subColor = "var(--text-faint)") => {
        const card = createDiv({ cls: "stat-card" });
        card.style.cssText = `
            background: var(--background-secondary);
            border-radius: 8px;
            padding: 10px 8px;
            text-align: center;
            border: 1px solid var(--background-modifier-border);
            transition: all 0.2s ease;
            box-shadow: 0 1px 3px rgba(0,0,0,0.02);
            min-height: 90px;
        `;
        const iconContainer = createDiv();
        iconContainer.style.cssText = createCardStyle(color);
        iconContainer.setText(icon);
        const lab = createDiv({ text: label });
        lab.style.cssText = `
            font-size: 12px;
            font-weight: 500;
            color: var(--text-muted);
            margin-bottom: 2px;
        `;
        const val = createDiv({ text: value });
        val.style.cssText = `
            font-size: 20px;
            font-weight: 800;
            color: ${color};
            margin-bottom: ${subLabel ? '2px' : '0'};
            line-height: 1.1;
        `;
        card.append(iconContainer, lab, val);
        if (subLabel) {
            const subLab = createDiv({ text: subLabel });
            subLab.style.cssText = `
                font-size: 10px;
                color: ${subColor};
                font-style: italic;
            `;
            card.appendChild(subLab);
        }
        return card;
    };
    
    const panel = createDiv({ cls: "tasks-stats-panel" });
    panel.style.cssText = panelStyle;
    
    const createSection = (title, content, subtitle = null) => {
        const section = createDiv({ cls: "stats-section" });
        section.style.cssText = "margin-bottom: 12px;";
        const header = createDiv({ text: title });
        header.style.cssText = `
            font-size: 13px;
            font-weight: 700;
            color: var(--text-normal);
            text-align: center;
            margin-bottom: 6px;
            padding-bottom: 4px;
            border-bottom: 1px solid var(--background-modifier-border);
        `;
        section.appendChild(header);
        if (subtitle) {
            const sub = createDiv({ text: subtitle });
            sub.style.cssText = `
                font-size: 11px;
                color: var(--text-muted);
                text-align: center;
                margin-bottom: 8px;
            `;
            section.appendChild(sub);
        }
        section.appendChild(content);
        return section;
    };
    
    const comparisonContent = createDiv();
    comparisonContent.style.cssText = `
        display: flex;
        justify-content: center;
        align-items: center;
        gap: 8px;
        font-family: var(--font-monospace);
        flex-wrap: wrap;
        min-height: 55px;
    `;
    const createComparisonBox = (value, color, bgColor, borderColor) => {
        const box = createDiv();
        box.style.cssText = `
            text-align: center;
            background: ${bgColor};
            border-radius: 8px;
            padding: 8px 12px;
            min-width: 60px;
            flex: 0 0 auto;
            box-shadow: 0 1px 3px ${bgColor};
            border: 1px solid ${borderColor};
        `;
        const num = createDiv({ text: String(value) });
        num.style.cssText = `
            font-size: 24px;
            font-weight: 800;
            color: ${color};
            line-height: 1;
        `;
        box.appendChild(num);
        return box;
    };
    
    comparisonContent.append(
        createComparisonBox(completedToday, completedToday > 0 ? "#2E7D32" : "#999", "rgba(46, 125, 50, 0.08)", "rgba(46, 125, 50, 0.15)"),
        (() => {
            const container = createDiv();
            container.style.cssText = `
                display: flex;
                flex-direction: column;
                align-items: center;
                justify-content: center;
                min-width: 70px;
                flex: 0 0 auto;
                padding: 2px 0;
                background: transparent;
            `;
            const trend1 = createDiv({ text: trendSymbol });
            trend1.style.cssText = `
                font-size: 16px;
                font-weight: 800;
                color: ${trendColor};
                line-height: 1;
            `;
            const trend2 = createDiv({ text: diffText });
            trend2.style.cssText = `
                font-size: 11px;
                color: ${diffColor};
                font-weight: 600;
                margin-top: 1px;
                background: ${diffValue !== 0 ? `${diffColor}15` : "transparent"};
                padding: 1px 4px;
                border-radius: 4px;
                min-height: 16px;
            `;
            container.append(trend1, trend2);
            return container;
        })(),
        createComparisonBox(completedLastMonth, completedLastMonth > 0 ? "#E65100" : "#999", "rgba(255, 152, 0, 0.08)", "rgba(255, 152, 0, 0.15)")
    );
    
    panel.appendChild(createSection("Activity Comparison", comparisonContent, "Today vs Last Month"));
    
    const recordsContent = createDiv();
    recordsContent.style.cssText = "display: grid; grid-template-columns: repeat(3, 1fr); gap: 8px;";
    recordsContent.append(
        createStatCard("🔥", String(currentStreak), "Current Streak", currentStreak > 0 ? "#FF5722" : "#999", currentStreak > 0 ? "days in a row" : "broken"),
        createStatCard("👑", String(bestStreak), "Best Streak", bestStreak > 0 ? "#FFC107" : "#999", bestStreak > 0 ? "days in a row" : null),
        createStatCard("⚡", String(maxTasksDayCount), "Daily Maximum", maxTasksDayCount > 0 ? "#FF9800" : "#999", maxTasksDayCount > 0 ? new Date(Object.keys(counts).find(k => counts[k] === maxTasksDayCount)).toLocaleDateString('ru-RU', {day: 'numeric', month: 'short'}) : null)
    );
    panel.appendChild(createSection("Records", recordsContent));
    
    const globalStatsContent = createDiv();
    globalStatsContent.style.cssText = "display: grid; grid-template-columns: repeat(3, 1fr); gap: 8px;";
    const daysCard = createDiv();
    daysCard.style.cssText = `
        background: var(--background-secondary);
        border-radius: 8px;
        padding: 10px 8px;
        text-align: center;
        border: 1px solid var(--background-modifier-border);
        box-shadow: 0 1px 3px rgba(0,0,0,0.02);
        min-height: 90px;
    `;
    const calIcon = createDiv();
    calIcon.style.cssText = createCardStyle("#9C27B0");
    calIcon.setText("📅");
    const daysLab = createDiv({ text: "Days with tasks" });
    daysLab.style.cssText = `
        font-size: 12px;
        font-weight: 500;
        color: var(--text-muted);
        margin-bottom: 2px;
    `;
    const daysVal = createDiv({ text: String(daysWithTasks) });
    daysVal.style.cssText = `
        font-size: 20px;
        font-weight: 800;
        color: ${daysWithTasks > 0 ? "#9C27B0" : "#999"};
        margin-bottom: 2px;
        line-height: 1.1;
    `;
    const totalLab = createDiv({ text: `out of ${totalDays}` });
    totalLab.style.cssText = `
        font-size: 10px;
        color: var(--text-faint);
        font-style: italic;
    `;
    daysCard.append(calIcon, daysLab, daysVal, totalLab);
    
    globalStatsContent.append(
        createStatCard("✓", String(totalCompleted), "Completed tasks", "#4CAF50", "all time"),
        createStatCard("🔁", avgPerDay, "Average", "#2196F3", daysWithTasks > 0 ? "tasks per day" : null),
        daysCard
    );
    panel.appendChild(createSection("Global Statistics", globalStatsContent));
    
    const weekContent = createDiv();
    weekContent.style.cssText = "margin-top: 8px;";
    const weekHeader = createDiv({ text: "Activity for 7 days" });
    weekHeader.style.cssText = `
        font-size: 13px;
        font-weight: 700;
        color: var(--text-normal);
        text-align: center;
        margin-bottom: 10px;
        padding-bottom: 4px;
        border-bottom: 1px solid var(--background-modifier-border);
    `;
    weekContent.appendChild(weekHeader);
    
    const weekContainer = createDiv();
    weekContainer.style.cssText = `
        display: flex;
        justify-content: space-between;
        align-items: flex-end;
        height: 130px;
        padding: 10px 5px 10px 5px;
        background: var(--background-secondary);
        border-radius: 8px;
        border: 1px solid var(--background-modifier-border);
        position: relative;
        overflow: hidden;
        margin-bottom: 12px;
    `;
    const maxVal = Math.max(...weekData, 1);
    for (let i = 0; i < weekData.length; i++) {
        const val = weekData[i];
        const heightPercent = maxVal > 0 ? Math.max((val / maxVal) * 100, 3) : 3;
        const isToday = i === weekData.length - 1;
        const hasTasks = val > 0 || isToday;
        let dayLabel = i === weekData.length - 1 ? 'Tod.' : (i === weekData.length - 2 ? 'Yes.' : `${weekData.length - 1 - i}`);
        
        const barContainer = createDiv();
        barContainer.style.cssText = `
            display: flex;
            flex-direction: column;
            align-items: center;
            height: 100%;
            justify-content: flex-end;
            flex: 1;
            max-width: 22px;
            position: relative;
            z-index: 2;
            margin: 0 3px;
        `;
        
        if (isToday || val > 0) {
            const valueLabel = createDiv({ text: isToday ? String(completedToday) : String(val) });
            valueLabel.style.cssText = `
                font-size: 10px;
                font-weight: 600;
                color: ${isToday ? "#2E7D32" : "var(--text-normal)"};
                margin-bottom: 3px;
                background: ${isToday ? "rgba(46, 125, 50, 0.85)" : "rgba(0,0,0,0.6)"};
                color: white;
                padding: 1px 2px;
                border-radius: 2px;
                white-space: nowrap;
                min-width: 16px;
                text-align: center;
                line-height: 1.1;
            `;
            barContainer.appendChild(valueLabel);
        }
        
        const bar = createDiv();
        bar.style.cssText = `
            width: ${hasTasks ? '14px' : '8px'};
            height: ${heightPercent}%;
            background: ${isToday ? "linear-gradient(to top, #2E7D32, #4CAF50)" : (hasTasks ? "linear-gradient(to top, #1976D2, #2196F3)" : "var(--background-modifier-border)")};
            border-radius: 3px 3px 0 0;
            transition: all 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
            box-shadow: 0 1px 2px rgba(0,0,0,0.1);
        `;
        
        const dateLabel = createDiv({ text: dayLabel });
        dateLabel.style.cssText = `
            font-size: 9px;
            color: ${isToday ? "#2E7D32" : "var(--text-muted)"};
            font-weight: ${isToday ? 600 : 500};
            margin-top: 4px;
            text-align: center;
            line-height: 1;
            white-space: nowrap;
        `;
        barContainer.append(bar, dateLabel);
        weekContainer.appendChild(barContainer);
    }
    weekContent.appendChild(weekContainer);
    
    const weekStats = createDiv();
    weekStats.style.cssText = "display: grid; grid-template-columns: repeat(3, 1fr); gap: 8px; margin-top: 6px;";
    weekStats.append(
        createStatCard("📈", String(currentWeekTotal), "Completed", "#3F51B5", "tasks this week"),
        createStatCard("📊", weekAvg.toFixed(1), "Average", "#009688", "tasks per week")
    );
    if (mostPopularTag) {
        weekStats.appendChild(createStatCard("🏷️", String(mostPopularTagCount), mostPopularTag, "#9C27B0", "popular tag"));
    } else {
        weekStats.appendChild(createStatCard("🏷️", "—", "No data", "#999"));
    }
    weekContent.appendChild(weekStats);
    panel.appendChild(weekContent);
    
    const footer = createDiv();
    footer.style.cssText = `
        font-size: 9px;
        color: var(--text-muted);
        text-align: center;
        padding-top: 8px;
        border-top: 1px solid var(--background-modifier-border);
        margin-top: 6px;
    `;
    const footerText = createDiv({ text: `Updated: ${now.toLocaleTimeString([], {hour: '2-digit', minute:'2-digit'})}` });
    footerText.style.cssText = `
        color: var(--text-faint);
        font-style: italic;
    `;
    footer.appendChild(footerText);
    panel.appendChild(footer);
    
    dv.paragraph(panel);
}
```
## Statistics and Progress
- [[Achievements Over the Last 30 Days.canvas|Achievements Over the Last 30 Days]]
- [[Statistics and Focus for 2026.canvas|Statistics and Focus for 2026]]
Formatting is supported
## About Life
Here you can add hyperlinks to dreams, goals, and desires you want to pursue. In short, anything you wish.