---
title: "Compose Room Relation完全ガイド — @Relation/1対多/多対多/Embedded/ネストクエリ"
emoji: "🔗"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Compose Room Relation**（@Relation、1対多、多対多、@Embedded、トランザクション）を解説します。

---

## 1対多リレーション

```kotlin
@Entity(tableName = "users")
data class User(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val name: String
)

@Entity(tableName = "posts", foreignKeys = [
    ForeignKey(entity = User::class, parentColumns = ["id"],
        childColumns = ["userId"], onDelete = ForeignKey.CASCADE)
])
data class Post(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val userId: Long,
    val title: String,
    val content: String
)

data class UserWithPosts(
    @Embedded val user: User,
    @Relation(parentColumn = "id", entityColumn = "userId")
    val posts: List<Post>
)

@Dao
interface UserDao {
    @Transaction
    @Query("SELECT * FROM users")
    fun getUsersWithPosts(): Flow<List<UserWithPosts>>

    @Transaction
    @Query("SELECT * FROM users WHERE id = :userId")
    fun getUserWithPosts(userId: Long): Flow<UserWithPosts>
}
```

---

## 多対多リレーション

```kotlin
@Entity(tableName = "playlists")
data class Playlist(
    @PrimaryKey(autoGenerate = true) val playlistId: Long = 0,
    val name: String
)

@Entity(tableName = "songs")
data class Song(
    @PrimaryKey(autoGenerate = true) val songId: Long = 0,
    val title: String, val artist: String
)

@Entity(primaryKeys = ["playlistId", "songId"])
data class PlaylistSongCrossRef(
    val playlistId: Long, val songId: Long
)

data class PlaylistWithSongs(
    @Embedded val playlist: Playlist,
    @Relation(
        parentColumn = "playlistId", entityColumn = "songId",
        associateBy = Junction(PlaylistSongCrossRef::class)
    )
    val songs: List<Song>
)

@Dao
interface PlaylistDao {
    @Transaction
    @Query("SELECT * FROM playlists")
    fun getPlaylistsWithSongs(): Flow<List<PlaylistWithSongs>>

    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun addSongToPlaylist(crossRef: PlaylistSongCrossRef)
}
```

---

## Compose UI

```kotlin
@Composable
fun UserPostsScreen(viewModel: UserViewModel = hiltViewModel()) {
    val usersWithPosts by viewModel.usersWithPosts.collectAsStateWithLifecycle(emptyList())

    LazyColumn(Modifier.padding(16.dp)) {
        items(usersWithPosts) { userWithPosts ->
            Card(Modifier.fillMaxWidth().padding(vertical = 4.dp)) {
                Column(Modifier.padding(16.dp)) {
                    Text(userWithPosts.user.name, style = MaterialTheme.typography.titleMedium)
                    Text("投稿数: ${userWithPosts.posts.size}",
                        color = MaterialTheme.colorScheme.onSurfaceVariant)
                    userWithPosts.posts.forEach { post ->
                        Text("• ${post.title}", Modifier.padding(start = 8.dp, top = 4.dp))
                    }
                }
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `@Relation` | リレーション定義 |
| `@Embedded` | エンティティ埋め込み |
| `@Junction` | 多対多の中間テーブル |
| `@Transaction` | 一貫性のある読み取り |

- `@Relation`で1対多/多対多を宣言的に定義
- `@Junction`で中間テーブルを指定して多対多を実現
- `@Transaction`でリレーション取得の一貫性を保証
- `ForeignKey`のonDeleteでカスケード削除

---

8種類のAndroidアプリテンプレート（Room対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Room](https://zenn.dev/myougatheaxo/articles/android-compose-compose-room-2026)
- [Compose Room Migration](https://zenn.dev/myougatheaxo/articles/android-compose-compose-room-migration-2026)
- [Compose Paging3](https://zenn.dev/myougatheaxo/articles/android-compose-compose-paging3-2026)
