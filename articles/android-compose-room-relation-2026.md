---
title: "Room Relation完全ガイド — @Relation/1対多/多対多/ネスト"
emoji: "🔗"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room Relation**（@Relation、1対多、多対多、ネストされたリレーション）を解説します。

---

## 1対多リレーション

```kotlin
@Entity
data class User(
    @PrimaryKey val userId: Long,
    val name: String
)

@Entity(foreignKeys = [ForeignKey(entity = User::class, parentColumns = ["userId"], childColumns = ["authorId"], onDelete = ForeignKey.CASCADE)])
data class Post(
    @PrimaryKey val postId: Long,
    val authorId: Long,
    val title: String,
    val content: String
)

// リレーションデータクラス
data class UserWithPosts(
    @Embedded val user: User,
    @Relation(parentColumn = "userId", entityColumn = "authorId")
    val posts: List<Post>
)

@Dao
interface UserDao {
    @Transaction
    @Query("SELECT * FROM User")
    fun getUsersWithPosts(): Flow<List<UserWithPosts>>

    @Transaction
    @Query("SELECT * FROM User WHERE userId = :id")
    fun getUserWithPosts(id: Long): Flow<UserWithPosts>
}
```

---

## 多対多リレーション

```kotlin
@Entity
data class Playlist(
    @PrimaryKey val playlistId: Long,
    val name: String
)

@Entity
data class Song(
    @PrimaryKey val songId: Long,
    val title: String,
    val artist: String
)

@Entity(primaryKeys = ["playlistId", "songId"])
data class PlaylistSongCrossRef(
    val playlistId: Long,
    val songId: Long
)

data class PlaylistWithSongs(
    @Embedded val playlist: Playlist,
    @Relation(
        parentColumn = "playlistId",
        entityColumn = "songId",
        associateBy = Junction(PlaylistSongCrossRef::class)
    )
    val songs: List<Song>
)

@Dao
interface PlaylistDao {
    @Transaction
    @Query("SELECT * FROM Playlist")
    fun getPlaylistsWithSongs(): Flow<List<PlaylistWithSongs>>

    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun addSongToPlaylist(crossRef: PlaylistSongCrossRef)
}
```

---

## ネストリレーション

```kotlin
data class UserWithPostsAndComments(
    @Embedded val user: User,
    @Relation(
        entity = Post::class,
        parentColumn = "userId",
        entityColumn = "authorId"
    )
    val postsWithComments: List<PostWithComments>
)

data class PostWithComments(
    @Embedded val post: Post,
    @Relation(parentColumn = "postId", entityColumn = "postId")
    val comments: List<Comment>
)
```

---

## まとめ

| リレーション | アノテーション |
|-------------|-------------|
| 1対多 | `@Relation` |
| 多対多 | `@Relation` + `Junction` |
| ネスト | `entity` パラメータ |
| 外部キー | `@ForeignKey` |

- `@Relation`で1対多リレーションを定義
- `Junction`で多対多の中間テーブルを指定
- `@Transaction`でリレーションクエリの一貫性保証
- `@Embedded`で親エンティティを埋め込み

---

8種類のAndroidアプリテンプレート（Room DB対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room/Flow](https://zenn.dev/myougatheaxo/articles/android-compose-room-flow-2026)
- [Room Migration](https://zenn.dev/myougatheaxo/articles/android-compose-room-migration-advanced-2026)
- [TypeConverter](https://zenn.dev/myougatheaxo/articles/android-compose-room-typeconverter-2026)
