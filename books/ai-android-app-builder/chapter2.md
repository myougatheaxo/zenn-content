---
title: "最初のアプリを作る — ハビットトラッカー完全手順"
---

理論は十分です。実際に作りましょう。

この章では、**ハビットトラッカーアプリ**をゼロから完成まで作ります。「毎日の習慣を記録して、継続日数を確認できる」シンプルなアプリです。

完成したアプリの機能：
- 習慣を追加・削除できる
- 毎日チェックを入れられる
- 連続達成日数（streak）が表示される
- データはスマホに保存される（Roomデータベース）

---

## ステップ1: プロジェクトを作成する

Android Studioを開き、**New Project** → **Empty Activity** を選択。

設定：
- **Name**: HabitTracker
- **Package name**: com.yourname.habittracker
- **Save location**: 好きな場所
- **Language**: Kotlin
- **Minimum SDK**: API 26 (Android 8.0)
- **Build configuration language**: Kotlin DSL

「Finish」をクリックして、Gradleのビルドが終わるまで待ちます（数分かかります）。

---

## ステップ2: 依存関係を追加する

`app/build.gradle.kts` を開いて、`dependencies { }` ブロックを以下に置き換えます。

AIへの指示：
```
app/build.gradle.kts に以下の依存関係を追加して。
- Jetpack Compose（BOM 2024.09.00）
- Room 2.6.1
- Lifecycle ViewModel Compose
- Navigation Compose
- Coroutines
```

AIが生成するコード（そのままコピーして使えます）：

```kotlin
// app/build.gradle.kts

plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.compose)
    id("com.google.devtools.ksp") version "2.0.21-1.0.27"
}

android {
    namespace = "com.yourname.habittracker"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.yourname.habittracker"
        minSdk = 26
        targetSdk = 35
        versionCode = 1
        versionName = "1.0"
    }

    buildFeatures {
        compose = true
    }
}

dependencies {
    val composeBom = platform("androidx.compose:compose-bom:2024.09.00")
    implementation(composeBom)

    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.activity:activity-compose:1.9.2")
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.5")
    implementation("androidx.navigation:navigation-compose:2.8.2")

    // Room
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    ksp("androidx.room:room-compiler:2.6.1")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")

    debugImplementation("androidx.compose.ui:ui-tooling")
}
```

`settings.gradle.kts` にKSPプラグインの記述も必要です：

```kotlin
// settings.gradle.kts の plugins ブロックに追加
id("com.google.devtools.ksp") version "2.0.21-1.0.27" apply false
```

追加後、**Sync Now** をクリックします。

---

## ステップ3: データモデルを作る

AIへの指示：
```
Habitエンティティを作って。
フィールド:
- id: Int (主キー、自動採番)
- name: String (習慣名)
- createdAt: Long (作成日時 epoch ms)
- completedDates: String (完了した日付リスト、カンマ区切り。例: "2025-01-01,2025-01-02")
```

AIが生成するコード：

```kotlin
// app/src/main/java/com/yourname/habittracker/data/Habit.kt

package com.yourname.habittracker.data

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "habits")
data class Habit(
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,
    val name: String,
    val createdAt: Long = System.currentTimeMillis(),
    val completedDates: String = ""  // "2025-01-01,2025-01-02" 形式
) {
    // 完了した日付のリストを取得
    fun getCompletedDateList(): List<String> {
        return if (completedDates.isEmpty()) emptyList()
        else completedDates.split(",")
    }

    // 今日完了しているか
    fun isCompletedToday(): Boolean {
        val today = java.time.LocalDate.now().toString()
        return getCompletedDateList().contains(today)
    }

    // 連続達成日数（streak）を計算
    fun calculateStreak(): Int {
        val dates = getCompletedDateList().sorted().reversed()
        if (dates.isEmpty()) return 0

        var streak = 0
        var checkDate = java.time.LocalDate.now()

        for (dateStr in dates) {
            val date = java.time.LocalDate.parse(dateStr)
            if (date == checkDate || date == checkDate.minusDays(1)) {
                streak++
                checkDate = date
            } else {
                break
            }
        }
        return streak
    }
}
```

---

## ステップ4: DAOとDatabaseを作る

AIへの指示：
```
HabitDao インターフェースを作って。
操作: 全件取得(Flow)、挿入、削除、日付追加（completedDatesを更新）
```

```kotlin
// app/src/main/java/com/yourname/habittracker/data/HabitDao.kt

package com.yourname.habittracker.data

import androidx.room.*
import kotlinx.coroutines.flow.Flow

@Dao
interface HabitDao {
    @Query("SELECT * FROM habits ORDER BY createdAt DESC")
    fun getAllHabits(): Flow<List<Habit>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertHabit(habit: Habit)

    @Delete
    suspend fun deleteHabit(habit: Habit)

    @Query("UPDATE habits SET completedDates = :dates WHERE id = :id")
    suspend fun updateCompletedDates(id: Int, dates: String)
}
```

```kotlin
// app/src/main/java/com/yourname/habittracker/data/HabitDatabase.kt

package com.yourname.habittracker.data

import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase

@Database(entities = [Habit::class], version = 1, exportSchema = false)
abstract class HabitDatabase : RoomDatabase() {
    abstract fun habitDao(): HabitDao

    companion object {
        @Volatile
        private var INSTANCE: HabitDatabase? = null

        fun getDatabase(context: Context): HabitDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    HabitDatabase::class.java,
                    "habit_database"
                ).build()
                INSTANCE = instance
                instance
            }
        }
    }
}
```

---

## ステップ5: Repositoryを作る

```kotlin
// app/src/main/java/com/yourname/habittracker/data/HabitRepository.kt

package com.yourname.habittracker.data

import kotlinx.coroutines.flow.Flow
import java.time.LocalDate

class HabitRepository(private val dao: HabitDao) {

    val allHabits: Flow<List<Habit>> = dao.getAllHabits()

    suspend fun addHabit(name: String) {
        dao.insertHabit(Habit(name = name))
    }

    suspend fun deleteHabit(habit: Habit) {
        dao.deleteHabit(habit)
    }

    suspend fun toggleToday(habit: Habit) {
        val today = LocalDate.now().toString()
        val dates = habit.getCompletedDateList().toMutableList()
        if (dates.contains(today)) {
            dates.remove(today)
        } else {
            dates.add(today)
        }
        dao.updateCompletedDates(habit.id, dates.joinToString(","))
    }
}
```

---

## ステップ6: ViewModelを作る

```kotlin
// app/src/main/java/com/yourname/habittracker/ui/HabitViewModel.kt

package com.yourname.habittracker.ui

import android.app.Application
import androidx.lifecycle.AndroidViewModel
import androidx.lifecycle.viewModelScope
import com.yourname.habittracker.data.Habit
import com.yourname.habittracker.data.HabitDatabase
import com.yourname.habittracker.data.HabitRepository
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch

class HabitViewModel(application: Application) : AndroidViewModel(application) {

    private val repository: HabitRepository

    val habits: StateFlow<List<Habit>>

    init {
        val dao = HabitDatabase.getDatabase(application).habitDao()
        repository = HabitRepository(dao)
        habits = repository.allHabits.stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )
    }

    fun addHabit(name: String) {
        viewModelScope.launch {
            repository.addHabit(name)
        }
    }

    fun deleteHabit(habit: Habit) {
        viewModelScope.launch {
            repository.deleteHabit(habit)
        }
    }

    fun toggleToday(habit: Habit) {
        viewModelScope.launch {
            repository.toggleToday(habit)
        }
    }
}
```

---

## ステップ7: UIを作る（Jetpack Compose）

AIへの指示：
```
Jetpack ComposeでHabitTrackerのメイン画面を作って。
- 習慣のリスト（カード形式）
- 各カードに: 習慣名、streak数、今日のチェックボタン
- 右上にFAB（+ボタン）で習慣追加ダイアログ
- 長押しで削除確認ダイアログ
```

```kotlin
// app/src/main/java/com/yourname/habittracker/ui/HabitScreen.kt

package com.yourname.habittracker.ui

import androidx.compose.foundation.ExperimentalFoundationApi
import androidx.compose.foundation.combinedClickable
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Add
import androidx.compose.material.icons.filled.Check
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import com.yourname.habittracker.data.Habit

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun HabitScreen(viewModel: HabitViewModel = viewModel()) {
    val habits by viewModel.habits.collectAsStateWithLifecycle()
    var showAddDialog by remember { mutableStateOf(false) }
    var habitToDelete by remember { mutableStateOf<Habit?>(null) }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("習慣トラッカー", fontWeight = FontWeight.Bold) },
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer
                )
            )
        },
        floatingActionButton = {
            FloatingActionButton(onClick = { showAddDialog = true }) {
                Icon(Icons.Default.Add, contentDescription = "習慣を追加")
            }
        }
    ) { padding ->
        if (habits.isEmpty()) {
            Box(
                modifier = Modifier.fillMaxSize().padding(padding),
                contentAlignment = Alignment.Center
            ) {
                Column(horizontalAlignment = Alignment.CenterHorizontally) {
                    Text("習慣がまだありません", style = MaterialTheme.typography.titleMedium)
                    Spacer(modifier = Modifier.height(8.dp))
                    Text("+ボタンで追加してみよう", style = MaterialTheme.typography.bodyMedium,
                        color = MaterialTheme.colorScheme.onSurfaceVariant)
                }
            }
        } else {
            LazyColumn(
                modifier = Modifier.padding(padding),
                contentPadding = PaddingValues(16.dp),
                verticalArrangement = Arrangement.spacedBy(12.dp)
            ) {
                items(habits, key = { it.id }) { habit ->
                    HabitCard(
                        habit = habit,
                        onToggle = { viewModel.toggleToday(habit) },
                        onLongPress = { habitToDelete = habit }
                    )
                }
            }
        }
    }

    if (showAddDialog) {
        AddHabitDialog(
            onConfirm = { name ->
                viewModel.addHabit(name)
                showAddDialog = false
            },
            onDismiss = { showAddDialog = false }
        )
    }

    habitToDelete?.let { habit ->
        DeleteConfirmDialog(
            habitName = habit.name,
            onConfirm = {
                viewModel.deleteHabit(habit)
                habitToDelete = null
            },
            onDismiss = { habitToDelete = null }
        )
    }
}

@OptIn(ExperimentalFoundationApi::class)
@Composable
fun HabitCard(habit: Habit, onToggle: () -> Unit, onLongPress: () -> Unit) {
    val isCompleted = habit.isCompletedToday()
    val streak = habit.calculateStreak()

    Card(
        modifier = Modifier
            .fillMaxWidth()
            .combinedClickable(
                onClick = {},
                onLongClick = onLongPress
            ),
        colors = CardDefaults.cardColors(
            containerColor = if (isCompleted)
                MaterialTheme.colorScheme.primaryContainer
            else
                MaterialTheme.colorScheme.surface
        )
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Column(modifier = Modifier.weight(1f)) {
                Text(
                    text = habit.name,
                    style = MaterialTheme.typography.titleMedium,
                    fontWeight = FontWeight.SemiBold
                )
                Spacer(modifier = Modifier.height(4.dp))
                Text(
                    text = if (streak > 0) "🔥 $streak 日連続" else "今日から始めよう",
                    style = MaterialTheme.typography.bodySmall,
                    color = MaterialTheme.colorScheme.onSurfaceVariant
                )
            }
            IconButton(onClick = onToggle) {
                Icon(
                    imageVector = Icons.Default.Check,
                    contentDescription = "今日チェック",
                    tint = if (isCompleted)
                        MaterialTheme.colorScheme.primary
                    else
                        MaterialTheme.colorScheme.outline
                )
            }
        }
    }
}

@Composable
fun AddHabitDialog(onConfirm: (String) -> Unit, onDismiss: () -> Unit) {
    var text by remember { mutableStateOf("") }

    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("習慣を追加") },
        text = {
            OutlinedTextField(
                value = text,
                onValueChange = { text = it },
                label = { Text("習慣名（例: 朝ランニング）") },
                singleLine = true
            )
        },
        confirmButton = {
            TextButton(
                onClick = { if (text.isNotBlank()) onConfirm(text.trim()) },
                enabled = text.isNotBlank()
            ) { Text("追加") }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) { Text("キャンセル") }
        }
    )
}

@Composable
fun DeleteConfirmDialog(habitName: String, onConfirm: () -> Unit, onDismiss: () -> Unit) {
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("習慣を削除") },
        text = { Text("「$habitName」を削除しますか？記録も全て消えます。") },
        confirmButton = {
            TextButton(onClick = onConfirm) {
                Text("削除", color = MaterialTheme.colorScheme.error)
            }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) { Text("キャンセル") }
        }
    )
}
```

---

## ステップ8: MainActivity を修正する

```kotlin
// app/src/main/java/com/yourname/habittracker/MainActivity.kt

package com.yourname.habittracker

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import com.yourname.habittracker.ui.HabitScreen
import com.yourname.habittracker.ui.theme.HabitTrackerTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            HabitTrackerTheme {
                HabitScreen()
            }
        }
    }
}
```

---

## ステップ9: ビルドして実行する

1. Android Studioの緑色の▶ボタン（Run）をクリック
2. エミュレータか実機を選択
3. アプリが起動すれば成功

よくあるエラーと解決策：

| エラー | 原因 | 解決策 |
|--------|------|--------|
| `Unresolved reference: Room` | 依存関係の同期漏れ | Sync Now をクリック |
| `KSP not found` | KSPプラグイン設定漏れ | settings.gradle.kts を確認 |
| `Cannot find symbol` | パッケージ名のミス | import文を確認、AIに聞く |

エラーが出たら、エラーメッセージをそのままAIにコピーして「このエラーを直して」と言えばOKです。

---

## 完成

ハビットトラッカーアプリが動いているはずです。チェックを入れると色が変わり、翌日も続けるとstreakが増えていきます。

次の章では、このアプリのコードが「何をしているのか」を解説します。丸暗記は不要です。「なんとなく分かる」レベルで十分です。
