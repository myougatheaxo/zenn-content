---
title: "KSP(Kotlin Symbol Processing)完全ガイド — コード生成/アノテーション処理"
emoji: "⚙️"
type: "tech"
topics: ["android", "kotlin", "ksp", "jetpackcompose"]
published: true
---

## この記事で学べること

**KSP**（Kotlin Symbol Processing、カスタムアノテーション、コード生成、KAPT比較、実践パターン）を解説します。

---

## KSPとは

KAPTの代替。Kotlinコンパイラプラグインとしてシンボル情報にアクセスし、コードを生成します。KAPTより最大2倍高速。

```kotlin
// build.gradle.kts
plugins {
    id("com.google.devtools.ksp") version "2.1.0-1.0.29"
}

dependencies {
    ksp("com.example:my-processor:1.0.0")
    implementation("com.example:my-annotations:1.0.0")
}
```

---

## アノテーション定義

```kotlin
// annotations モジュール
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class AutoFactory

@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class AutoMapper(val target: KClass<*>)
```

---

## SymbolProcessor

```kotlin
class AutoFactoryProcessor(
    private val codeGenerator: CodeGenerator,
    private val logger: KSPLogger
) : SymbolProcessor {

    override fun process(resolver: Resolver): List<KSAnnotated> {
        val symbols = resolver.getSymbolsWithAnnotation(AutoFactory::class.qualifiedName!!)
        val unprocessed = mutableListOf<KSAnnotated>()

        symbols.forEach { symbol ->
            if (symbol !is KSClassDeclaration) {
                logger.error("@AutoFactory is only applicable to classes", symbol)
                return@forEach
            }
            if (!symbol.validate()) {
                unprocessed.add(symbol)
                return@forEach
            }
            generateFactory(symbol)
        }

        return unprocessed
    }

    private fun generateFactory(classDeclaration: KSClassDeclaration) {
        val packageName = classDeclaration.packageName.asString()
        val className = classDeclaration.simpleName.asString()
        val constructor = classDeclaration.primaryConstructor ?: return

        val file = codeGenerator.createNewFile(
            Dependencies(true, classDeclaration.containingFile!!),
            packageName,
            "${className}Factory"
        )

        file.writer().use { writer ->
            writer.write("package $packageName\n\n")
            writer.write("object ${className}Factory {\n")
            writer.write("    fun create(\n")

            constructor.parameters.forEachIndexed { index, param ->
                val name = param.name?.asString() ?: return@forEachIndexed
                val type = param.type.resolve().declaration.qualifiedName?.asString() ?: "Any"
                writer.write("        $name: $type")
                if (index < constructor.parameters.size - 1) writer.write(",")
                writer.write("\n")
            }

            writer.write("    ): $className = $className(\n")
            constructor.parameters.forEachIndexed { index, param ->
                val name = param.name?.asString() ?: return@forEachIndexed
                writer.write("        $name = $name")
                if (index < constructor.parameters.size - 1) writer.write(",")
                writer.write("\n")
            }
            writer.write("    )\n}\n")
        }
    }
}
```

---

## ProcessorProvider

```kotlin
class AutoFactoryProcessorProvider : SymbolProcessorProvider {
    override fun create(environment: SymbolProcessorEnvironment): SymbolProcessor {
        return AutoFactoryProcessor(
            codeGenerator = environment.codeGenerator,
            logger = environment.logger
        )
    }
}

// resources/META-INF/services/com.google.devtools.ksp.processing.SymbolProcessorProvider
// com.example.processor.AutoFactoryProcessorProvider
```

---

## 使用例

```kotlin
@AutoFactory
data class User(
    val id: String,
    val name: String,
    val email: String
)

// 生成されるコード:
// object UserFactory {
//     fun create(id: String, name: String, email: String): User =
//         User(id = id, name = name, email = email)
// }
```

---

## まとめ

| 要素 | 役割 |
|------|------|
| `SymbolProcessor` | シンボル処理ロジック |
| `CodeGenerator` | ファイル生成 |
| `Resolver` | シンボル解決 |
| `SymbolProcessorProvider` | エントリーポイント |

- KSPはKAPTより最大2倍高速
- Room、Hilt、Moshiなど主要ライブラリがKSP対応
- カスタムアノテーションでボイラープレート削減
- `validate()`で未解決シンボルを遅延処理

---

8種類のAndroidアプリテンプレート（KSP対応ビルド設定）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [カスタムLint](https://zenn.dev/myougatheaxo/articles/android-compose-custom-lint-rule-2026)
- [Convention Plugin](https://zenn.dev/myougatheaxo/articles/android-compose-gradle-convention-plugin-2026)
- [Hilt DI](https://zenn.dev/myougatheaxo/articles/android-compose-dependency-injection-hilt-2026)
