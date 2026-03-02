---
title: "カレンダーUI実装ガイド — Compose月表示/日付選択"
emoji: "📅"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "calendar"]
published: true
---

## この記事で学べること

Composeでの**カレンダーUI**（月表示、日付選択、イベント表示）の実装を解説します。

---

## カレンダーグリッド

```kotlin
@Composable
fun CalendarView(
    selectedDate: LocalDate,
    onDateSelected: (LocalDate) -> Unit,
    events: Map<LocalDate, List<String>> = emptyMap()
) {
    var currentMonth by remember { mutableStateOf(YearMonth.now()) }
    val daysInMonth = currentMonth.lengthOfMonth()
    val firstDayOfWeek = currentMonth.atDay(1).dayOfWeek.value % 7 // 日曜=0

    Column(Modifier.padding(8.dp)) {
        // 月ナビゲーション
        Row(
            Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment = Alignment.CenterVertically
        ) {
            IconButton(onClick = { currentMonth = currentMonth.minusMonths(1) }) {
                Icon(Icons.Default.ChevronLeft, "前月")
            }
            Text(
                "${currentMonth.year}年${currentMonth.monthValue}月",
                style = MaterialTheme.typography.titleMedium
            )
            IconButton(onClick = { currentMonth = currentMonth.plusMonths(1) }) {
                Icon(Icons.Default.ChevronRight, "次月")
            }
        }

        // 曜日ヘッダー
        Row(Modifier.fillMaxWidth()) {
            listOf("日", "月", "火", "水", "木", "金", "土").forEach { day ->
                Text(
                    day,
                    Modifier.weight(1f),
                    textAlign = TextAlign.Center,
                    style = MaterialTheme.typography.labelSmall,
                    color = MaterialTheme.colorScheme.outline
                )
            }
        }

        // 日付グリッド
        val totalCells = firstDayOfWeek + daysInMonth
        val rows = (totalCells + 6) / 7

        for (row in 0 until rows) {
            Row(Modifier.fillMaxWidth()) {
                for (col in 0..6) {
                    val dayIndex = row * 7 + col - firstDayOfWeek + 1
                    if (dayIndex in 1..daysInMonth) {
                        val date = currentMonth.atDay(dayIndex)
                        val isSelected = date == selectedDate
                        val hasEvent = events.containsKey(date)

                        DayCell(
                            day = dayIndex,
                            isSelected = isSelected,
                            isToday = date == LocalDate.now(),
                            hasEvent = hasEvent,
                            onClick = { onDateSelected(date) },
                            modifier = Modifier.weight(1f)
                        )
                    } else {
                        Spacer(Modifier.weight(1f))
                    }
                }
            }
        }
    }
}
```

---

## 日付セル

```kotlin
@Composable
fun DayCell(
    day: Int,
    isSelected: Boolean,
    isToday: Boolean,
    hasEvent: Boolean,
    onClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    Box(
        modifier = modifier
            .aspectRatio(1f)
            .padding(2.dp)
            .clip(CircleShape)
            .background(
                when {
                    isSelected -> MaterialTheme.colorScheme.primary
                    isToday -> MaterialTheme.colorScheme.primaryContainer
                    else -> Color.Transparent
                }
            )
            .clickable(onClick = onClick),
        contentAlignment = Alignment.Center
    ) {
        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            Text(
                "$day",
                color = when {
                    isSelected -> MaterialTheme.colorScheme.onPrimary
                    isToday -> MaterialTheme.colorScheme.onPrimaryContainer
                    else -> MaterialTheme.colorScheme.onSurface
                },
                style = MaterialTheme.typography.bodyMedium
            )
            if (hasEvent) {
                Box(
                    Modifier
                        .size(4.dp)
                        .clip(CircleShape)
                        .background(
                            if (isSelected) MaterialTheme.colorScheme.onPrimary
                            else MaterialTheme.colorScheme.primary
                        )
                )
            }
        }
    }
}
```

---

## イベント表示

```kotlin
@Composable
fun CalendarWithEvents() {
    var selectedDate by remember { mutableStateOf(LocalDate.now()) }
    val events = remember {
        mapOf(
            LocalDate.now() to listOf("会議 10:00", "ランチ 12:00"),
            LocalDate.now().plusDays(2) to listOf("プレゼン 14:00"),
            LocalDate.now().plusDays(5) to listOf("締切日")
        )
    }

    Column {
        CalendarView(
            selectedDate = selectedDate,
            onDateSelected = { selectedDate = it },
            events = events
        )

        HorizontalDivider()

        // 選択日のイベント
        val dayEvents = events[selectedDate] ?: emptyList()
        if (dayEvents.isEmpty()) {
            Box(Modifier.fillMaxWidth().padding(32.dp), contentAlignment = Alignment.Center) {
                Text("予定なし", color = MaterialTheme.colorScheme.outline)
            }
        } else {
            LazyColumn {
                items(dayEvents) { event ->
                    ListItem(
                        headlineContent = { Text(event) },
                        leadingContent = {
                            Icon(Icons.Default.Event, null, tint = MaterialTheme.colorScheme.primary)
                        }
                    )
                }
            }
        }
    }
}
```

---

## まとめ

- `YearMonth`/`LocalDate`で月と日付を管理
- `LazyVerticalGrid`ではなくRow+Weightで7列グリッド
- `DayCell`でカスタム日付セル（選択/今日/イベント表示）
- 月ナビゲーションで前月/次月切り替え
- イベントドットで予定あり表示
- 選択日のイベントリストをカレンダー下に表示

---

8種類のAndroidアプリテンプレート（カレンダー機能対応可能）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [日付/時刻ピッカーガイド](https://zenn.dev/myougatheaxo/articles/android-compose-date-time-picker-2026)
- [カスタムレイアウトガイド](https://zenn.dev/myougatheaxo/articles/android-compose-custom-layout-2026)
- [Canvas描画ガイド](https://zenn.dev/myougatheaxo/articles/compose-canvas-drawing-2026)
