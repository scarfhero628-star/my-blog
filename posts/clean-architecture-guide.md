---
title: 클린 아키텍처 완전 가이드 — 의존성 역전으로 유지보수하기 좋은 소프트웨어 만들기
date: 2026-11-12
tags: [클린 아키텍처, 소프트웨어 설계, 아키텍처, 의존성 역전, 백엔드]
description: 로버트 마틴의 클린 아키텍처 원칙을 실전 예제와 함께 익혀 변경에 강하고 테스트하기 쉬운 소프트웨어를 설계하는 방법을 소개합니다.
---

## 클린 아키텍처란 무엇인가

소프트웨어 프로젝트가 커질수록 특정 고민이 반복됩니다. "이 기능 하나 바꾸려면 왜 이렇게 많은 파일을 건드려야 할까?", "테스트 코드를 짜려면 데이터베이스를 반드시 켜야 하나?", "프레임워크를 바꾸면 처음부터 다시 짜야 할까?" 이런 질문들이 생기는 순간, 아키텍처를 다시 생각해볼 때가 된 겁니다.

**클린 아키텍처(Clean Architecture)**는 로버트 C. 마틴(Robert C. Martin, 일명 Uncle Bob)이 제안한 소프트웨어 설계 원칙입니다. 핵심 목표는 단순합니다. **비즈니스 로직(핵심 규칙)을 프레임워크, 데이터베이스, UI 등 외부 세계로부터 완전히 독립시키는 것**입니다.

이를 통해 얻을 수 있는 것들이 있습니다.

- 프레임워크에 종속되지 않는 코드
- 테스트하기 쉬운 비즈니스 로직
- 데이터베이스 교체가 쉬운 구조
- UI 변경이 비즈니스 로직에 영향을 주지 않는 설계

---

## 동심원 구조: 계층을 나누는 방식

클린 아키텍처는 흔히 **동심원(Concentric Circles)** 으로 표현됩니다. 안쪽 원일수록 고수준 정책(비즈니스 규칙)이고, 바깥쪽 원일수록 세부 구현(DB, 프레임워크, UI)입니다.

```
[ Frameworks & Drivers ]     ← 가장 바깥: Express, React, MySQL 등
       ↓
[ Interface Adapters ]        ← 컨트롤러, 프레젠터, 게이트웨이
       ↓
[ Application Business Rules ]← 유스케이스(Use Cases)
       ↓
[ Enterprise Business Rules ] ← 엔티티(Entities)
```

### 핵심 규칙: 의존성은 항상 안쪽을 향한다

가장 중요한 규칙은 **의존성 규칙(Dependency Rule)**입니다. 바깥 원의 코드는 안쪽 원을 알 수 있지만, 안쪽 원의 코드는 바깥 원을 절대 알아서는 안 됩니다.

즉, 엔티티는 유스케이스를 모르고, 유스케이스는 컨트롤러를 모르고, 컨트롤러는 Express를 모르는 방향으로 설계합니다.

---

## 각 계층의 역할

### 1. 엔티티 (Entities)

가장 안쪽에 위치한 순수한 비즈니스 객체입니다. 어떤 프레임워크나 라이브러리에도 의존하지 않습니다.

```typescript
// entities/User.ts
export class User {
  constructor(
    public readonly id: string,
    public readonly email: string,
    private password: string,
    public readonly createdAt: Date
  ) {}

  isValidEmail(): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(this.email);
  }

  changePassword(newPassword: string): void {
    if (newPassword.length < 8) {
      throw new Error('비밀번호는 8자 이상이어야 합니다.');
    }
    this.password = newPassword;
  }
}
```

이 클래스는 Node.js 없이도, Express 없이도, 어떤 환경에서도 테스트할 수 있습니다.

### 2. 유스케이스 (Use Cases)

애플리케이션 고유의 비즈니스 로직을 담습니다. "사용자가 회원 가입한다", "주문을 생성한다" 같은 구체적인 시나리오를 표현합니다.

```typescript
// use-cases/RegisterUser.ts
export interface UserRepository {
  findByEmail(email: string): Promise<User | null>;
  save(user: User): Promise<void>;
}

export class RegisterUser {
  constructor(private userRepository: UserRepository) {}

  async execute(email: string, password: string): Promise<User> {
    const existing = await this.userRepository.findByEmail(email);
    if (existing) {
      throw new Error('이미 등록된 이메일입니다.');
    }

    const user = new User(
      generateId(),
      email,
      password,
      new Date()
    );

    if (!user.isValidEmail()) {
      throw new Error('유효하지 않은 이메일 형식입니다.');
    }

    await this.userRepository.save(user);
    return user;
  }
}
```

여기서 `UserRepository`는 인터페이스입니다. 유스케이스는 실제 데이터베이스가 MySQL인지, MongoDB인지, 메모리 저장소인지 전혀 알 필요가 없습니다.

### 3. 인터페이스 어댑터 (Interface Adapters)

외부 세계(HTTP 요청, 데이터베이스)와 내부 세계(유스케이스, 엔티티) 사이의 변환을 담당합니다.

```typescript
// adapters/UserController.ts
export class UserController {
  constructor(private registerUser: RegisterUser) {}

  async handleRegister(req: Request, res: Response): Promise<void> {
    try {
      const { email, password } = req.body;
      const user = await this.registerUser.execute(email, password);
      res.status(201).json({ id: user.id, email: user.email });
    } catch (error) {
      res.status(400).json({ message: error.message });
    }
  }
}
```

```typescript
// adapters/UserRepositoryImpl.ts
export class UserRepositoryImpl implements UserRepository {
  constructor(private db: Database) {}

  async findByEmail(email: string): Promise<User | null> {
    const row = await this.db.query(
      'SELECT * FROM users WHERE email = ?',
      [email]
    );
    if (!row) return null;
    return new User(row.id, row.email, row.password, row.created_at);
  }

  async save(user: User): Promise<void> {
    await this.db.query(
      'INSERT INTO users (id, email, password, created_at) VALUES (?, ?, ?, ?)',
      [user.id, user.email, user.password, user.createdAt]
    );
  }
}
```

### 4. 프레임워크와 드라이버 (Frameworks & Drivers)

가장 바깥쪽에 Express, React, MySQL 같은 실제 프레임워크와 라이브러리가 위치합니다.

```typescript
// main.ts (조립 지점)
const db = new Database(process.env.DATABASE_URL);
const userRepository = new UserRepositoryImpl(db);
const registerUser = new RegisterUser(userRepository);
const userController = new UserController(registerUser);

const app = express();
app.post('/register', (req, res) => userController.handleRegister(req, res));
```

이처럼 `main.ts`(또는 진입점)가 모든 의존성을 조립합니다. 이를 **의존성 주입(Dependency Injection)**이라 합니다.

---

## 테스트가 쉬워지는 이유

클린 아키텍처의 가장 큰 실용적 이점은 **테스트 용이성**입니다. 유스케이스는 인터페이스에만 의존하므로, 테스트 시 실제 데이터베이스 대신 가짜(mock) 구현체를 주입할 수 있습니다.

```typescript
// tests/RegisterUser.test.ts
class InMemoryUserRepository implements UserRepository {
  private users: User[] = [];

  async findByEmail(email: string): Promise<User | null> {
    return this.users.find(u => u.email === email) ?? null;
  }

  async save(user: User): Promise<void> {
    this.users.push(user);
  }
}

describe('RegisterUser', () => {
  it('이미 존재하는 이메일로 가입하면 에러가 발생한다', async () => {
    const repository = new InMemoryUserRepository();
    const useCase = new RegisterUser(repository);

    await useCase.execute('test@example.com', 'password123');

    await expect(
      useCase.execute('test@example.com', 'password456')
    ).rejects.toThrow('이미 등록된 이메일입니다.');
  });
});
```

데이터베이스 없이 순수 메모리에서 테스트가 실행됩니다. 빠르고, 안정적이고, CI 환경에서도 별도 인프라가 필요 없습니다.

---

## 언제 클린 아키텍처를 도입해야 할까

클린 아키텍처가 항상 정답은 아닙니다. 트레이드오프가 있습니다.

| 도입이 적합한 경우 | 도입이 과도한 경우 |
|---|---|
| 장기간 유지보수할 복잡한 시스템 | 프로토타입 또는 MVP 단계 |
| 다양한 외부 시스템과 연동 | 단순한 CRUD 애플리케이션 |
| 여러 팀이 협업하는 대규모 프로젝트 | 1~2인 소규모 단기 프로젝트 |
| 도메인 로직이 복잡한 비즈니스 서비스 | 비즈니스 로직이 거의 없는 도구 |

초기에는 계층 구조로 인한 파일 수 증가와 보일러플레이트가 부담스럽게 느껴질 수 있습니다. 하지만 프로젝트가 커질수록 그 가치가 드러납니다.

---

## 실무 적용 팁

### 모든 것을 한 번에 바꾸려 하지 마세요

기존 프로젝트에 클린 아키텍처를 도입할 때는 한 번에 전체를 리팩토링하지 말고, 새로운 기능을 추가할 때 그 부분만 클린 아키텍처 방식으로 작성하는 **점진적 도입 전략**을 권장합니다.

### 인터페이스 위치에 주의하세요

인터페이스는 그것을 **사용하는** 쪽(유스케이스)에 위치해야 합니다. `UserRepository` 인터페이스는 `adapters` 폴더가 아니라 `use-cases` 폴더에 두는 게 맞습니다.

### 도메인 언어를 코드에 담으세요

엔티티와 유스케이스의 이름은 비즈니스 담당자가 사용하는 언어와 일치해야 합니다. 기술 용어보다 도메인 언어를 우선합니다.

---

## 마치며

클린 아키텍처는 처음에는 오버엔지니어링처럼 보일 수 있습니다. 하지만 1년 뒤, 3년 뒤 그 코드베이스를 다시 열었을 때의 차이를 경험하면 생각이 바뀝니다. 비즈니스 로직이 외부 세계로부터 보호받는 코드는 변경에 강하고, 테스트하기 쉽고, 협업하기 편합니다.

완벽한 아키텍처는 없습니다. 하지만 의존성이 항상 안쪽을 향한다는 단 하나의 원칙만 지켜도, 여러분의 코드는 훨씬 더 오래, 더 쉽게 살아남을 수 있습니다.
