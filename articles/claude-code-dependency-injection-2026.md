---
title: "Claude Codeで依存性注入を設計する：テスト可能なコードの作り方"
emoji: "💉"
type: "tech"
topics: ["claudecode", "typescript", "nodejs", "設計パターン", "テスト"]
published: true
---

## はじめに

依存性注入（DI）を適切に設計するとモジュール間の結合が弱まり、テストが容易になる。Claude CodeにDIコンテナなしの軽量DIとtsinjectを使ったDIの両方を設計させる。

---

## CLAUDE.mdにDI設計ルールを書く

```markdown
## 依存性注入（DI）ルール

### 基本方針
- コンストラクタインジェクション（constructor injection）を基本とする
- DIコンテナは必要に応じて使う（小規模プロジェクトは不要）
- サービスクラスは具体的な依存を直接インスタンス化しない

### インターフェース定義
- 全ての外部依存はインターフェースで抽象化
  例: IEmailService, IUserRepository
- テスト時はモック実装に差し替えられること
- インターフェースはsrc/interfaces/ に配置

### 命名規則
- インターフェース: I{Name}（例: IUserRepository）
- 実装クラス: {Name}（例: UserRepository）
- モック: Mock{Name}（例: MockUserRepository）

### テスト設計
- 単体テストは実際のDBを使わない（Repositoryをモックする）
- DIを使っているコードはvitest + mockで十分テストできる

### 禁止
- new KeywordによるService内部での依存の直接生成
- Serviceクラス内でのprismaの直接呼び出し（Repository層を挟む）
```

---

## インターフェースとリポジトリパターンの生成

```
UserServiceのDI設計を生成してください（DIコンテナなし、軽量DI）。

要件：
- IUserRepositoryインターフェースを定義
- UserRepositoryの実装（Prismaを使う）
- MockUserRepositoryの実装（テスト用）
- UserServiceはIUserRepositoryに依存（具体クラスに依存しない）
- EmailServiceもインターフェース経由

生成するファイル:
- src/interfaces/IUserRepository.ts
- src/repositories/UserRepository.ts
- src/repositories/MockUserRepository.ts
- src/services/UserService.ts
```

---

## 生成されるDI設計

```typescript
// src/interfaces/IUserRepository.ts
export interface IUserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  create(data: CreateUserData): Promise<User>;
  update(id: string, data: Partial<UpdateUserData>): Promise<User>;
  delete(id: string): Promise<void>;
}

// src/interfaces/IEmailService.ts
export interface IEmailService {
  sendWelcomeEmail(user: User): Promise<void>;
  sendPasswordResetEmail(user: User, token: string): Promise<void>;
}
```

```typescript
// src/repositories/UserRepository.ts（Prisma実装）
import { IUserRepository } from '../interfaces/IUserRepository';
import { prisma } from '../lib/prisma';

export class UserRepository implements IUserRepository {
  async findById(id: string): Promise<User | null> {
    return prisma.user.findUnique({ where: { id } });
  }

  async findByEmail(email: string): Promise<User | null> {
    return prisma.user.findUnique({ where: { email } });
  }

  async create(data: CreateUserData): Promise<User> {
    return prisma.user.create({ data });
  }

  async update(id: string, data: Partial<UpdateUserData>): Promise<User> {
    return prisma.user.update({ where: { id }, data });
  }

  async delete(id: string): Promise<void> {
    await prisma.user.delete({ where: { id } });
  }
}
```

```typescript
// src/services/UserService.ts（具体クラスに依存しない）
import { IUserRepository } from '../interfaces/IUserRepository';
import { IEmailService } from '../interfaces/IEmailService';

export class UserService {
  constructor(
    private readonly userRepository: IUserRepository,
    private readonly emailService: IEmailService
  ) {}

  async createUser(data: CreateUserData): Promise<User> {
    const existing = await this.userRepository.findByEmail(data.email);
    if (existing) throw new Error('Email already in use');

    const user = await this.userRepository.create(data);
    await this.emailService.sendWelcomeEmail(user);
    return user;
  }

  async getUserById(id: string): Promise<User> {
    const user = await this.userRepository.findById(id);
    if (!user) throw new NotFoundError(`User not found: ${id}`);
    return user;
  }
}
```

---

## テストでのDIの活用

```
UserServiceの単体テストを生成してください。

要件：
- IUserRepositoryをモックに差し替え（DBを使わない）
- IEmailServiceをモックに差し替え（実際にメールを送らない）
- vitestのvi.fn()でモック関数を作成
- テストケース: ユーザー作成成功、メール重複エラー

保存先: src/services/__tests__/UserService.test.ts
```

```typescript
// src/services/__tests__/UserService.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { UserService } from '../UserService';
import { IUserRepository } from '../../interfaces/IUserRepository';
import { IEmailService } from '../../interfaces/IEmailService';

describe('UserService', () => {
  let userService: UserService;
  let mockUserRepository: IUserRepository;
  let mockEmailService: IEmailService;

  beforeEach(() => {
    // モック実装
    mockUserRepository = {
      findById: vi.fn(),
      findByEmail: vi.fn(),
      create: vi.fn(),
      update: vi.fn(),
      delete: vi.fn(),
    };

    mockEmailService = {
      sendWelcomeEmail: vi.fn(),
      sendPasswordResetEmail: vi.fn(),
    };

    // DIでサービスを構築（実際のDBなし）
    userService = new UserService(mockUserRepository, mockEmailService);
  });

  it('ユーザーを作成してウェルカムメールを送る', async () => {
    const data = { email: 'new@example.com', name: 'Test User' };
    const createdUser = { id: '123', ...data, createdAt: new Date() };

    vi.mocked(mockUserRepository.findByEmail).mockResolvedValue(null);
    vi.mocked(mockUserRepository.create).mockResolvedValue(createdUser);
    vi.mocked(mockEmailService.sendWelcomeEmail).mockResolvedValue(undefined);

    const result = await userService.createUser(data);

    expect(result).toEqual(createdUser);
    expect(mockEmailService.sendWelcomeEmail).toHaveBeenCalledWith(createdUser);
  });

  it('メールが重複している場合はエラーを投げる', async () => {
    vi.mocked(mockUserRepository.findByEmail).mockResolvedValue({
      id: 'existing', email: 'existing@example.com', name: 'Existing',
    });

    await expect(
      userService.createUser({ email: 'existing@example.com', name: 'New' })
    ).rejects.toThrow('Email already in use');

    expect(mockUserRepository.create).not.toHaveBeenCalled();
  });
});
```

---

## まとめ

Claude Codeで依存性注入を設計する：

1. **CLAUDE.md** にインターフェース優先・コンストラクタインジェクションを明記
2. **インターフェースを先に定義** してから実装を生成
3. **サービスはインターフェースに依存** （具体クラスに依存しない）
4. **テストはモックで差し替え** （DBなしで高速に実行）

---

*DIとテスト設計の問題を自動検出するスキルは **Code Review Pack（¥980）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
