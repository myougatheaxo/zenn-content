---
title: "ページネーション応用ガイド — Keyset/Cursor/無限スクロール/Paging3カスタム"
emoji: "📄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "paging"]
published: true
---

## この記事で学べること

**ページネーション応用**（Keyset/Cursorベース、カスタムPagingSource、RemoteMediator、エラーリトライ）を解説します。

---

## Cursor/Keysetベースのページネーション

```kotlin
class CursorPagingSource(
    private val api: PostApi
) : PagingSource<String, Post>() {

    override suspend fun load(params: LoadParams<String>): LoadResult<String, Post> {
        return try {
            val cursor = params.key
            val response = api.getPosts(cursor = cursor, limit = params.loadSize)

            LoadResult.Page(
                data = response.posts,
                prevKey = null,  // 前方ページングは不要
                nextKey = response.nextCursor  // APIが返すカーソル
            )
        } catch (e: IOException) {
            LoadResult.Error(e)
        } catch (e: HttpException) {
            LoadResult.Error(e)
        }
    }

    override fun getRefreshKey(state: PagingState<String, Post>): String? = null
}
```

---

## RemoteMediator（オフライン対応）

```kotlin
@OptIn(ExperimentalPagingApi::class)
class PostRemoteMediator(
    private val api: PostApi,
    private val db: AppDatabase
) : RemoteMediator<Int, PostEntity>() {

    override suspend fun load(
        loadType: LoadType,
        state: PagingState<Int, PostEntity>
    ): MediatorResult {
        val page = when (loadType) {
            LoadType.REFRESH -> 1
            LoadType.PREPEND -> return MediatorResult.Success(endOfPaginationReached = true)
            LoadType.APPEND -> {
                val lastItem = state.lastItemOrNull()
                    ?: return MediatorResult.Success(endOfPaginationReached = true)
                lastItem.page + 1
            }
        }

        return try {
            val response = api.getPosts(page = page, limit = state.config.pageSize)
            db.withTransaction {
                if (loadType == LoadType.REFRESH) {
                    db.postDao().clearAll()
                }
                db.postDao().insertAll(response.map { it.toEntity(page) })
            }
            MediatorResult.Success(endOfPaginationReached = response.isEmpty())
        } catch (e: Exception) {
            MediatorResult.Error(e)
        }
    }
}
```

---

## エラーリトライUI

```kotlin
@Composable
fun PagingListScreen(viewModel: PostViewModel = hiltViewModel()) {
    val posts = viewModel.posts.collectAsLazyPagingItems()

    LazyColumn(Modifier.fillMaxSize()) {
        items(posts.itemCount, key = { posts[it]?.id ?: it }) { index ->
            posts[index]?.let { post ->
                PostItem(post)
            }
        }

        // ロード状態表示
        when (posts.loadState.append) {
            is LoadState.Loading -> {
                item {
                    Box(Modifier.fillMaxWidth().padding(16.dp), contentAlignment = Alignment.Center) {
                        CircularProgressIndicator()
                    }
                }
            }
            is LoadState.Error -> {
                item {
                    Column(Modifier.fillMaxWidth().padding(16.dp), horizontalAlignment = Alignment.CenterHorizontally) {
                        Text("読み込みエラー", color = MaterialTheme.colorScheme.error)
                        Button(onClick = { posts.retry() }) { Text("再試行") }
                    }
                }
            }
            else -> {}
        }
    }

    // 全体エラー
    if (posts.loadState.refresh is LoadState.Error) {
        Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
            Column(horizontalAlignment = Alignment.CenterHorizontally) {
                Text("データを取得できません")
                Button(onClick = { posts.refresh() }) { Text("再読み込み") }
            }
        }
    }
}
```

---

## まとめ

| パターン | 用途 |
|---------|------|
| Offset | シンプルなページ番号 |
| Cursor/Keyset | リアルタイムデータ |
| RemoteMediator | オフライン対応 |
| リトライ | エラーからの回復 |

- Cursorベースでデータ整合性を保証
- `RemoteMediator`でAPI→Room→UIのパイプライン
- `posts.retry()`/`posts.refresh()`でエラーリトライ
- `loadState`で各段階のエラー/ローディング表示

---

8種類のAndroidアプリテンプレート（ページネーション対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Paging3基本](https://zenn.dev/myougatheaxo/articles/android-compose-paging3-compose-2026)
- [無限スクロール](https://zenn.dev/myougatheaxo/articles/android-compose-infinite-scroll-2026)
- [Room + Paging](https://zenn.dev/myougatheaxo/articles/android-compose-paging-room-2026)
