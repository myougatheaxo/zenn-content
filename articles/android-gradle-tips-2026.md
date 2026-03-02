---
title: "Android Gradleビルド高速化 — 遅いビルドを3倍速にする設定"
emoji: "🏎️"
type: "tech"
topics: ["android", "gradle", "kotlin", "build"]
published: true
---

## この記事で学べること

Gradleビルドが遅い原因と、**即座に効果が出る高速化設定**を解説します。ビルド時間を最大3倍速にできます。

---

## gradle.properties の最適化

```properties
# gradle.properties

# 並列ビルド
org.gradle.parallel=true

# Gradleデーモン
org.gradle.daemon=true

# JVMメモリ設定
org.gradle.jvmargs=-Xmx4g -XX:MaxMetaspaceSize=512m -XX:+UseParallelGC

# 設定キャッシュ
org.gradle.configuration-cache=true

# ビルドキャッシュ
org.gradle.caching=true
```

---

## 各設定の効果

| 設定 | 効果 | 目安 |
|------|------|------|
| `parallel=true` | モジュール並列ビルド | 20-30%短縮 |
| `daemon=true` | JVM再起動を回避 | 初回以降高速 |
| `jvmargs -Xmx4g` | メモリ不足を防止 | GC停止回避 |
| `configuration-cache` | 設定フェーズをキャッシュ | 20-50%短縮 |
| `caching=true` | ビルド成果物をキャッシュ | 変更なし部分スキップ |

---

## KSP vs kapt

```kotlin
// ❌ kapt（遅い）
kapt("androidx.room:room-compiler:2.6.1")

// ✅ KSP（速い）
ksp("androidx.room:room-compiler:2.6.1")
```

```kotlin
// build.gradle.kts
plugins {
    id("com.google.devtools.ksp") version "2.0.0-1.0.22"
}
```

KSPはkaptの**2倍以上高速**。Room、Hiltなど対応ライブラリはKSPに移行すべき。

---

## 不要なビルド機能を無効化

```kotlin
android {
    buildFeatures {
        buildConfig = true
        viewBinding = false    // Compose使用時は不要
        dataBinding = false    // Compose使用時は不要
        aidl = false
        renderScript = false
    }
}
```

---

## ビルド時間の計測

```bash
# ビルド時間の詳細レポート
./gradlew assembleDebug --profile

# Build Scan（詳細分析）
./gradlew assembleDebug --scan
```

`--profile`でHTML形式のレポートが生成されます。どのタスクが遅いか一目瞭然。

---

## モジュール分割

```
# 1モジュール: 全体を毎回ビルド
ビルド時間: 90秒

# マルチモジュール: 変更部分だけビルド
ビルド時間: 30秒（変更したモジュールのみ）
```

---

## CI/CDでのキャッシュ

```yaml
# GitHub Actions
- name: Setup Gradle
  uses: gradle/actions/setup-gradle@v3
  with:
    cache-read-only: ${{ github.ref != 'refs/heads/main' }}
```

---

## やってはいけないこと

| NG | 理由 |
|----|------|
| `clean`を毎回実行 | キャッシュが全削除される |
| JVMメモリを1GB以下 | GCが頻繁に発生 |
| kaptのまま放置 | KSPの2倍以上遅い |
| 全モジュールにCompose | 不要なモジュールまで遅くなる |

---

## まとめ

1. `gradle.properties`に5つの設定を追加
2. kapt → KSPに移行
3. 不要なbuildFeaturesを無効化
4. `--profile`でボトルネックを特定
5. モジュール分割で増分ビルドを活用

---

8種類のAndroidアプリテンプレート（Gradle最適化設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Version Catalog入門](https://zenn.dev/myougatheaxo/articles/android-version-catalog-2026)
- [CI/CD入門（GitHub Actions）](https://zenn.dev/myougatheaxo/articles/android-cicd-github-actions-2026)
- [マルチモジュール入門](https://zenn.dev/myougatheaxo/articles/android-multi-module-2026)
