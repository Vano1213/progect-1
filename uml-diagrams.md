# UML-диаграммы — «Интернет-банк для физических лиц»

По ТЗ из README репозитория. Диаграммы написаны на Mermaid (rendering в VSCode, GitHub, Mermaid Live Editor и т.п.).

---

## 1. Диаграмма вариантов использования (Use Case)

```mermaid
flowchart TD
    Op[Операционист] --> UC1[Вести карточки клиентов]
    Op --> UC2[Вести справочник счетов]
    Op --> UC3[Регистрировать операции: зачисления/списания/переводы]
    Op --> UC4[Оплата услуг и платежей]
    Op --> UC5[Формировать выписку по счёту]
    Op --> UC6[Формировать отчёты и реестры]

    Kas[Кассир] --> UC3
    Kas --> UC7[Приём наличных платежей]

    Spec[Ведущий специалист по работе с клиентами] --> UC5
    Spec --> UC8[Формировать список операций, требующих контроля]
    Spec --> UC9[Сводная отчётность по офису]

    Adm[Администратор системы] --> UC10[Управление пользователями и правами]
    Adm --> UC11[Резервное копирование и восстановление БД]
    Adm --> UC12[Настройка справочника видов платежей]

    UC5 --> EX1[Экспорт в MS Excel]
    UC6 --> EX1
    UC9 --> EX1

    UC1 --> INCL1[Поиск клиентов: фамилия, ИНН]
    UC2 --> INCL1

    subgraph "Аутентификация"
        AUTH[Вход по паролю]
    end
    Op --> AUTH
    Kas --> AUTH
    Spec --> AUTH
    Adm --> AUTH
```

---

## 2. Диаграмма классов (Class)

```mermaid
classDiagram
    class User {
        +id: int
        +login: string
        +passwordHash: string
        +role: Role
        +fullName: string
    }

    class Role {
        <<enumeration>>
        OPERATONIST
        CASHIER
        SPECIALIST
        ADMIN
    }

    class Client {
        +id: int
        +fullName: string
        +birthDate: date
        +passportSeries: string
        +passportNumber: string
        +inn: string
        +phone: string
        +email: string
        +status: ClientStatus
    }

    class ClientStatus {
        <<enumeration>>
        ACTIVE
        BLOCKED
        CLOSED
    }

    class Account {
        +id: int
        +number: string
        +currency: string
        +type: string
        +balance: decimal
        +usageMode: string
    }

    class Operation {
        +id: int
        +amount: decimal
        +dateTime: datetime
        +type: OperationType
        +commission: decimal
    }

    class OperationType {
        <<enumeration>>
        CREDIT
        DEBIT
        TRANSFER
        PAYMENT
    }

    class PaymentType {
        +id: int
        +name: string
        +code: string
        +commission: decimal
        +conditions: string
    }

    class Report {
        +id: int
        +periodFrom: date
        +periodTo: date
        +createdAt: datetime
        +generate(): void
        +exportToExcel(): void
    }

    Client "1" --> "0..*" Account : has
    Client "1" --> "0..*" Operation : performs
    Account "1" --> "0..*" Operation : source/target
    Operation "1" --> "1" PaymentType : uses
    Report "1" --> "0..*" Operation : aggregates
    User "1" --> "0..*" Report : generates

    class AccountController {
        +openAccount(client: Client): Account
        +closeAccount(account: Account): void
        +blockAccount(account: Account): void
    }

    class OperationService {
        +credit(account: Account, amount: decimal): void
        +debit(account: Account, amount: decimal): void
        +transfer(from: Account, to: Account, amount: decimal): void
        +pay(paymentType: PaymentType, account: Account, amount: decimal): void
        -validateAmount(amount: decimal): bool
        -checkSufficientFunds(account: Account, amount: decimal): bool
    }

    class ReportService {
        +accountStatement(account: Account, from: date, to: date): Report
        +officeDailyReport(date: date): Report
        +controlOperationsReport(): Report
        +paymentRegister(from: date, to: date): Report
    }

    User --> OperationService : uses
    User --> ReportService : uses
    User --> AccountController : uses
```

---

## 3. Диаграмма последовательности — перевод между счетами (Sequence)

```mermaid
sequenceDiagram
    participant Op as Операционист
    participant UI as UI-слой (Операции)
    participant OS as OperationService
    participant DB as База данных
    participant Rep as ReportService

    Op->>UI: ввод реквизитов перевода (счёт-источник, счёт-приёмник, сумма)
    UI->>OS: transfer(from, to, amount)
    OS->>OS: validateAmount(amount)
    alt сумму отрицательная или 0
        OS-->>UI: ошибка «некорректная сумма»
    end
    OS->>DB: SELECT баланс счёта-источника
    DB-->>OS: остаток
    alt остаток < суммы
        OS-->>UI: ошибка «превышение остатка по счёту»
    end
    OS->>DB: UPDATE счёт-источник (списание) — т.е. BEGIN TRANSACTION
    OS->>DB: UPDATE счёт-приёмник (зачисление)
    OS->>DB: INSERT операция (перевод, комиссия) — COMMIT
    DB-->>OS: ok
    OS-->>UI: успех
    UI->>Rep: формирование выписки / квитанции
    Rep-->>Op: результат
```

---

## 4. Диаграмма деятельности — приём платежа (Activity)

```mermaid
flowchart TD
    A([Начало]) --> B[Аутентификация пользователя]
    B --> C{Пароль верен?}
    C -- нет --> B
    C -- да --> D[Выбор вида платежа]
    D --> E[Ввод суммы и реквизитов]
    E --> F{Проверка входных данных}
    F -- ошибка --> E
    F -- ок --> G{Остаток достаточен?}
    G -- нет --> E
    G -- да --> H[Провести операцию в БД]
    H --> I[Пересчитать остаток по счёту]
    I --> J{Нужен отчёт?}
    J -- да --> K[Сформировать отчёт / квитанцию]
    J -- нет --> L
    K --> L[Экспорт в Excel при необходимости]
    L --> M([Конец])
```

---

## 5. Диаграмма компонентов (Component)

```mermaid
flowchart LR
    subgraph UI_SOFT[UI-слой]
        UC[Модуль учёта клиентов и счетов]
        UO[Модуль операций]
        UR[Модуль формирования отчётов]
        UA[Модуль администрирования]
    end

    subgraph APP[Прикладной слой]
        SVC[Сервисы: клиенты, счета, операции]
        REP[Сервис отчётности]
        AUTH[Аутентификация и права]
        EXC[Экспорт в MS Excel]
    end

    subgraph DATA[Слой данных]
        DB[(База данных до 1 млн счетов)]
        BK[Резервное копирование / восстановление]
    end

    UC --> SVC
    UO --> SVC
    UR --> REP
    UA --> AUTH
    REP --> EXC
    SVC --> DB
    DB --> BK
    AUTH --> DB
```

---

## 6. Диаграмма развёртывания (Deployment)

```mermaid
flowchart TD
    subgraph Offices[Банковская сеть]
        PK[Персональный компьютер / рабочие места пользователей]
        PK --> |"Windows 7/10/11, GUI"| APP["Клиентское ПО «Интернет-банк»"]
        APP --> |сетевой доступ| SRV[Сервер БД]
        SRV --> DB[(База данных<br/>до 1 000 000 счетов)]
        SRV --> BK[Система резервного копирования]
        APP --> PRN[Принтер<br/>печать выписок и отчётов]
        APP --> EXL[MS Excel 2010+<br/>экспорт отчётов]
    end
```

---

### Замечания
- Диаграммы построены по тексту ТЗ и описывают **логику из документа**, без привязки к фактическому коду (в README его нет).
- Точные атрибуты сущностей (поля классов) уточняются по образцам входных/выходных документов из Приложений 1–3 ТЗ.