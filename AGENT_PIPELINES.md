# Agent Pipelines Documentation

Документация всех агентных пайплайнов и схем работы в приложении Latitude LLM.

## Оглавление

1. [Пайплайн модерации контента](#пайплайн-модерации-контента)
2. [Пайплайн поддержки клиентов](#пайплайн-поддержки-клиентов)
3. [Пайплайн финансового анализа](#пайплайн-финансового-анализа)
4. [Пайплайн оценки (LLM-as-a-Judge)](#пайплайн-оценки-llm-as-a-judge)
5. [Базовая инфраструктура агентов](#базовая-инфраструктура-агентов)

---

## Пайплайн модерации контента

### Общее описание

Многоагентная система для интеллектуальной модерации пользовательского контента. Главный координатор оркестрирует работу трех специализированных агентов, каждый из которых анализирует контент с разных точек зрения, затем синтезирует их результаты в финальное решение о модерации.

### Схема пайплайна

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        CONTENT MODERATION PIPELINE                       │
└─────────────────────────────────────────────────────────────────────────┘

                              ┌──────────────┐
                              │   USER       │
                              │   INPUT      │
                              └──────┬───────┘
                                     │
                         ┌───────────▼────────────┐
                         │   Main Coordinator     │
                         │ (Google Gemini Flash)  │
                         │  Temperature: 0.2      │
                         └───────────┬────────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
          ┌─────────▼─────────┐  ┌──▼────────┐  ┌───▼──────────┐
          │  Rule Checker     │  │ Toxicity  │  │   Safety     │
          │ (OpenAI GPT-4o-m) │  │ Evaluator │  │   Scorer     │
          │  Temperature: 0.1 │  │ (Claude)  │  │   (Claude)   │
          └─────────┬─────────┘  └──┬────────┘  └───┬──────────┘
                    │               │               │
                    │ [Tools]       │ [Analysis]    │ [Results]
                    │               │               │
          ┌─────────▼─────────────────────────────────────────┐
          │  check_profanity_filter                           │
          │  validate_content_length                          │
          │  scan_for_patterns                                │
          └─────────┬─────────────────────────────────────────┘
                    │
                    │
          ┌─────────▼──────────────┐
          │    DATA AGGREGATION    │
          │                        │
          │  Rule violations       │
          │  Toxicity scores       │
          │  Safety metrics        │
          └─────────┬──────────────┘
                    │
          ┌─────────▼──────────────┐
          │  Main Coordinator      │
          │  SYNTHESIS & DECISION  │
          │                        │
          │  • Analyzes all data   │
          │  • Applies logic       │
          │  • Makes decision      │
          └─────────┬──────────────┘
                    │
          ┌─────────▼──────────────┐
          │    FINAL OUTPUT        │
          │                        │
          │  {                     │
          │    decision: "approve/flag/reject",
          │    confidence: 0.95,   │
          │    reasoning: "...",   │
          │    violations: [...],  │
          │    recommended_action  │
          │  }                     │
          └────────────────────────┘
```

### Потоки данных

#### 1. Входные данные

```javascript
{
  content: string,           // Контент для модерации
  content_type: string,      // Тип контента (text, comment, post)
  platform_context: string   // Контекст платформы
}
```

#### 2. Rule Checker → Main Coordinator

**Промпт Rule Checker получает:**
```javascript
{
  content: string,
  content_type: string
}
```

**Возвращает:**
```javascript
{
  rule_violations: string[],      // ["profanity", "spam"]
  severity: "low" | "medium" | "high",
  details: string,                 // Детали находок
  passed_basic_filters: boolean    // true/false
}
```

**Используемые инструменты:**
- `check_profanity_filter(content, content_type)` → результаты проверки на нецензурную лексику
- `validate_content_length(content, content_type)` → валидация длины
- `scan_for_patterns(content, pattern_types)` → обнаружение спам-паттернов

#### 3. Toxicity Evaluator → Main Coordinator

**Промпт Toxicity Evaluator получает:**
```javascript
{
  content: string,
  platform_context: string,
  user_history: string
}
```

**Возвращает:**
```javascript
{
  toxicity_detected: boolean,
  toxicity_type: "harassment" | "hate_speech" | "threat" | "other" | "none",
  severity_score: number,      // 1-10
  confidence: number,          // 0-1
  reasoning: string,
  context_factors: string[]    // Факторы, влияющие на оценку
}
```

**Анализ включает:**
- Контекстуальную токсичность (сарказм, скрытый вред)
- Культурную чувствительность
- Классификацию намерений
- Оценку серьезности по градуированной шкале

#### 4. Safety Scorer → Main Coordinator

**Промпт Safety Scorer получает:**
```javascript
{
  content: string,
  rule_results: object,        // Результаты от Rule Checker
  toxicity_results: object     // Результаты от Toxicity Evaluator
}
```

**Возвращает:**
```javascript
{
  safety_scores: {
    immediate_harm_risk: number,         // 0-100
    community_impact: number,            // 0-100
    policy_violation_severity: number,   // 0-100
    escalation_potential: number,        // 0-100
    context_sensitivity: number          // 0-100
  },
  overall_risk_score: number,            // 0-100 (weighted average)
  confidence_interval: [number, number], // [lower, upper]
  requires_human_review: boolean,
  monitoring_level: "none" | "light" | "heavy",
  risk_factors: string[]
}
```

**Система оценки:** Негативная (высокие значения = больше риска)

#### 5. Синтез решения (Main Coordinator)

**Логика принятия решения:**

```
IF overall_risk_score > 80 OR toxicity_detected AND severity_score >= 8:
  decision = "reject"

ELSE IF overall_risk_score > 50 OR requires_human_review:
  decision = "flag"

ELSE IF passed_basic_filters AND overall_risk_score < 30:
  decision = "approve"

ELSE:
  decision = "flag"  // По умолчанию для граничных случаев
```

**Финальный выход:**
```javascript
{
  decision: "approve" | "flag" | "reject",
  confidence: number,              // 0-1
  reasoning: string,               // Краткое объяснение
  violations: string[],            // Список нарушений
  recommended_action: string       // Конкретное действие
}
```

### Особенности пайплайна

1. **Параллельное выполнение:** Три агента могут работать параллельно
2. **Специализация:** Каждый агент фокусируется на своей области экспертизы
3. **Контекстная осведомленность:** Агенты учитывают контекст платформы и историю пользователя
4. **Градуированная оценка:** Не просто "да/нет", а детальные метрики
5. **Explainability:** Каждое решение объясняется с указанием причин

### Конфигурация моделей

| Агент | Провайдер | Модель | Temperature | Причина выбора |
|-------|-----------|--------|-------------|----------------|
| Main Coordinator | Google | gemini-1.5-flash | 0.2 | Быстрая оркестрация, низкая креативность |
| Rule Checker | OpenAI | gpt-4o-mini | 0.1 | Детерминированные проверки, максимальная консистентность |
| Toxicity Evaluator | Anthropic | claude-3-5-sonnet | 0.3 | Понимание нюансов и контекста |
| Safety Scorer | Anthropic | claude-3-5-sonnet | 0.1 | Точные количественные оценки |

### Примеры выполнения

#### Пример 1: Безобидный контент

```
Вход: "Great product! Really helped me solve my problem."

Rule Checker: {
  rule_violations: [],
  severity: "low",
  passed_basic_filters: true
}

Toxicity Evaluator: {
  toxicity_detected: false,
  toxicity_type: "none",
  severity_score: 1,
  confidence: 0.98
}

Safety Scorer: {
  overall_risk_score: 5,
  requires_human_review: false,
  monitoring_level: "none"
}

Решение: {
  decision: "approve",
  confidence: 0.99,
  reasoning: "Content is positive feedback with no policy violations",
  violations: []
}
```

#### Пример 2: Токсичный контент

```
Вход: "[hateful content directed at protected group]"

Rule Checker: {
  rule_violations: ["hate_speech"],
  severity: "high",
  passed_basic_filters: false
}

Toxicity Evaluator: {
  toxicity_detected: true,
  toxicity_type: "hate_speech",
  severity_score: 9,
  confidence: 0.95
}

Safety Scorer: {
  overall_risk_score: 92,
  requires_human_review: false,
  monitoring_level: "heavy"
}

Решение: {
  decision: "reject",
  confidence: 0.96,
  reasoning: "Content contains hate speech targeting protected group",
  violations: ["hate_speech", "community_guidelines"],
  recommended_action: "Permanent removal, possible account action"
}
```

---

## Пайплайн поддержки клиентов

### Общее описание

Интеллектуальная система для автоматизации обработки запросов клиентов. Координатор анализирует входящие запросы, определяет необходимость дополнительной информации, делегирует исследование информации о клиенте специализированному агенту, затем передает данные агенту-составителю email для создания персонализированного ответа.

### Схема пайплайна

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CUSTOMER SUPPORT EMAIL PIPELINE                       │
└─────────────────────────────────────────────────────────────────────────┘

                        ┌─────────────────────┐
                        │  CUSTOMER QUERY     │
                        │  + Email            │
                        │  + Priority Level   │
                        └──────────┬──────────┘
                                   │
                     ┌─────────────▼──────────────┐
                     │  Main Coordinator          │
                     │  (Google Gemini Flash)     │
                     │  Temperature: 0.3          │
                     │                            │
                     │  STEP 1: Analyze Query     │
                     └─────────────┬──────────────┘
                                   │
                ┌──────────────────┼──────────────────┐
                │                  │                  │
    ┌───────────▼────────┐  ┌──────▼──────┐  ┌──────▼──────────┐
    │ get_customer_      │  │ get_order_  │  │ check_known_    │
    │ details()          │  │ history()   │  │ issues()        │
    └───────────┬────────┘  └──────┬──────┘  └──────┬──────────┘
                │                  │                 │
                └──────────────────┼─────────────────┘
                                   │
                     ┌─────────────▼──────────────┐
                     │                            │
                     │  NEEDS CLARIFICATION?      │
                     │                            │
                     └───┬─────────────────┬──────┘
                         │ YES             │ NO
                         │                 │
            ┌────────────▼────┐           │
            │  RETURN         │           │
            │  {               │           │
            │  needs_clarif... │           │
            │  questions: [...]}          │
            └─────────────────┘           │
                                          │
                             ┌────────────▼──────────────┐
                             │  STEP 2:                  │
                             │  Customer Researcher      │
                             │  (OpenAI GPT-4.1)         │
                             │                           │
                             │  6-STEP RESEARCH:         │
                             │  1. Extract identifiers   │
                             │  2. Gather details        │
                             │  3. Check issues          │
                             │  4. Previous interactions │
                             │  5. Subscription level    │
                             │  6. Special circumstances │
                             └────────────┬──────────────┘
                                          │
                             ┌────────────▼──────────────┐
                             │  RESEARCH REPORT          │
                             │                           │
                             │  • Customer profile       │
                             │  • Account status         │
                             │  • Order/subscription     │
                             │  • Known issues           │
                             │  • Red flags              │
                             │  • Recommendations        │
                             └────────────┬──────────────┘
                                          │
                             ┌────────────▼──────────────┐
                             │  STEP 3:                  │
                             │  Email Composer           │
                             │  (Anthropic Claude)       │
                             │  Temperature: 0.4         │
                             │                           │
                             │  PROCESS:                 │
                             │  1. Analyze emotion       │
                             │  2. Personalize           │
                             │  3. Address issue         │
                             │  4. Set expectations      │
                             │  5. Add next steps        │
                             └────────────┬──────────────┘
                                          │
                             ┌────────────▼──────────────┐
                             │  FINAL EMAIL RESPONSE     │
                             │                           │
                             │  {                        │
                             │    subject: "...",        │
                             │    body: "...",           │
                             │    tone_analysis: "...",  │
                             │    personalization: [...] │
                             │  }                        │
                             └───────────────────────────┘
```

### Потоки данных

#### 1. Входные данные

```javascript
{
  customer_email: string,      // Email клиента
  customer_query: string,      // Запрос клиента
  priority_level: string       // Уровень приоритета (low, medium, high, urgent)
}
```

#### 2. Main Coordinator → Tools

**Вызовы инструментов:**

```javascript
// Получение деталей клиента
get_customer_details({ email: customer_email })
→ {
  customer_id: string,
  name: string,
  account_type: "free" | "premium" | "enterprise",
  created_at: date,
  status: "active" | "suspended" | "trial"
}

// Получение истории заказов
get_order_history({ customer_id: string })
→ {
  orders: [{
    order_id: string,
    date: date,
    amount: number,
    status: "completed" | "pending" | "refunded",
    items: [...]
  }]
}

// Проверка известных проблем
check_known_issues({ issue_keywords: string[] })
→ {
  known_issues: [{
    issue_id: string,
    description: string,
    status: "open" | "resolved" | "investigating",
    workaround: string
  }]
}
```

#### 3. Main Coordinator → Customer Researcher

**Промпт Customer Researcher получает:**
```javascript
{
  research_request: string  // Включает контекст и конкретные вопросы для исследования
}
```

**Возвращает детальный отчет:**
```javascript
{
  customer_profile: {
    name: string,
    email: string,
    account_type: string,
    tenure: string,           // Как долго клиент с нами
    lifetime_value: number
  },
  account_status: {
    current_status: string,
    recent_activity: string,
    payment_status: string
  },
  order_subscription_history: [{
    type: "order" | "subscription",
    date: date,
    details: string,
    issues: string[]
  }],
  known_related_issues: [{
    issue: string,
    relevance: string,
    status: string,
    solution: string
  }],
  previous_interactions: [{
    date: date,
    type: "email" | "chat" | "phone",
    summary: string,
    resolution: string
  }],
  special_circumstances: {
    is_vip: boolean,
    recent_issues_count: number,
    escalation_history: string[],
    notes: string
  },
  recommended_approach: string,  // Как лучше обработать этот запрос
  red_flags: string[]            // Потенциальные проблемы
}
```

**6-шаговый процесс исследования:**
1. Извлечение email клиента и идентификаторов
2. Сбор деталей аккаунта и истории
3. Проверка известных проблем
4. Поиск предыдущих взаимодействий
5. Определение уровня подписки
6. Отметка особых обстоятельств (VIP, недавние проблемы)

#### 4. Customer Researcher → Email Composer

**Промпт Email Composer получает:**
```javascript
{
  customer_info: string,     // Отформатированная информация о клиенте
  issue_details: string,     // Детали проблемы из исследования
  tone_required: string      // "standard" | "urgent" | "empathetic" | "technical"
}
```

**Возвращает:**
```javascript
{
  subject: string,                // Профессиональная тема письма
  body: string,                   // Полное тело письма с форматированием
  tone_analysis: string,          // Анализ используемого тона
  personalization_elements: [     // Элементы персонализации
    "Used customer name",
    "Referenced order #12345",
    "Acknowledged account anniversary"
  ]
}
```

**Процесс создания email:**
1. Анализ эмоционального состояния клиента (frustrated/confused/satisfied)
2. Использование информации о клиенте для персонализации
3. Решение конкретной проблемы с четкими действиями
4. Включение релевантных деталей аккаунта
5. Установка ожиданий по срокам решения
6. Завершение следующими шагами и контактной информацией

#### 5. Логика определения тона

```
IF priority_level == "urgent" OR customer_history.recent_issues_count > 2:
  tone = "empathetic"

ELSE IF issue_type == "technical" OR customer.account_type == "enterprise":
  tone = "technical"

ELSE IF priority_level == "high":
  tone = "urgent"

ELSE:
  tone = "standard"
```

### Варианты обработки по типу запроса

#### A. Разгневанный клиент

```
Input: "This is the third time I'm contacting support! Nothing works!"

Flow:
1. Coordinator detects frustration in sentiment analysis
2. Customer Researcher finds 3 previous tickets
3. Tone set to "empathetic"
4. Email Composer creates response with:
   - Sincere apology
   - Acknowledgment of frustration
   - Immediate action plan
   - Direct contact to support manager
   - Compensation offer (if applicable)
```

#### B. Технический запрос

```
Input: "Getting 500 error when calling /api/users endpoint with OAuth token"

Flow:
1. Coordinator identifies technical nature
2. Customer Researcher checks:
   - API usage history
   - Known API issues
   - Account API limits
3. Tone set to "technical"
4. Email Composer creates response with:
   - Technical analysis of issue
   - Step-by-step debugging guide
   - Code examples
   - API documentation links
   - Direct line to technical support
```

#### C. Вопрос по оплате

```
Input: "Why was I charged twice for my subscription?"

Flow:
1. Coordinator triggers billing inquiry path
2. Customer Researcher verifies:
   - Payment history with get_order_history()
   - Subscription details
   - Recent charge patterns
3. Tone set to "standard" with extra precision
4. Email Composer creates response with:
   - Exact charge breakdown
   - Clear explanation of charges
   - Refund process (if duplicate found)
   - Updated payment schedule
   - Billing contact information
```

### Конфигурация моделей

| Компонент | Провайдер | Модель | Temperature | Причина выбора |
|-----------|-----------|--------|-------------|----------------|
| Main Coordinator | Google | gemini-1.5-flash | 0.3 | Быстрый анализ и маршрутизация запросов |
| Customer Researcher | OpenAI | gpt-4.1 | default | Глубокое исследование и синтез информации |
| Email Composer | Anthropic | claude-3-sonnet | 0.4 | Эмпатичное и естественное написание текста |

### Пример полного выполнения

```
=== ВХОД ===
customer_email: "john@example.com"
customer_query: "I ordered Premium plan 3 days ago but still don't have access"
priority_level: "medium"

=== ШАГ 1: КООРДИНАТОР АНАЛИЗИРУЕТ ===
- Sentiment: Frustrated but polite
- Issue type: Access/provisioning
- Urgency: Medium-high (3 days waiting)

=== ШАГ 2: ИНСТРУМЕНТЫ ===
get_customer_details("john@example.com")
→ { customer_id: "c_123", name: "John Doe", account_type: "free" }

get_order_history("c_123")
→ { orders: [{ order_id: "ord_456", date: "3 days ago",
              amount: 49.99, status: "completed", item: "Premium Plan" }] }

check_known_issues(["premium", "access", "provisioning"])
→ { known_issues: [{ issue_id: "iss_789",
                     description: "Delayed Premium activation for some users",
                     status: "investigating",
                     workaround: "Manual activation available" }] }

=== ШАГ 3: CUSTOMER RESEARCHER ===
Research Report:
{
  customer_profile: {
    name: "John Doe",
    tenure: "2 years",
    account_type: "free → premium (pending)",
    lifetime_value: 49.99
  },
  issue: "Premium upgrade not activated after successful payment",
  known_related_issue: "System experiencing delays in automatic upgrades",
  recommended_approach: "Immediate manual activation + apology + compensation",
  red_flags: ["3-day delay unacceptable", "Risk of churn"]
}

=== ШАГ 4: EMAIL COMPOSER (Empathetic tone) ===
{
  subject: "Your Premium Plan Activation - Immediate Resolution",

  body: "
    Dear John,

    Thank you for reaching out, and I sincerely apologize for the delay in
    activating your Premium plan. I can see that your payment was processed
    successfully 3 days ago (Order #ord_456), and you should have had immediate
    access.

    I've personally escalated this to our technical team, and your Premium
    features have now been manually activated. You should see full Premium
    access within the next 15 minutes.

    As an apology for this inconvenience, I've also added an additional month
    to your Premium subscription at no charge.

    To verify everything is working:
    1. Log out of your account
    2. Log back in
    3. Check Settings > Subscription
    4. You should see 'Premium - Active'

    If you experience any further issues, please reply directly to this email
    or call our Priority Support line at 1-800-XXX-XXXX (mention ticket #TKT123
    for immediate assistance).

    Thank you for your patience and for choosing our service.

    Best regards,
    Sarah Chen
    Senior Customer Success Manager
  ",

  tone_analysis: "Empathetic with immediate action",
  personalization_elements: [
    "Used customer name",
    "Referenced specific order #ord_456",
    "Mentioned exact timeline (3 days)",
    "Provided compensation",
    "Gave direct priority support access"
  ]
}

=== ВЫХОД ===
{
  needs_clarification: false,
  clarification_questions: [],
  email_response: { ... (см. выше) }
}
```

### Особенности пайплайна

1. **Контекстная осведомленность:** Учет истории клиента и предыдущих взаимодействий
2. **Адаптивный тон:** Автоматический выбор тона на основе ситуации
3. **Персонализация:** Использование конкретных деталей аккаунта
4. **Проактивность:** Предвидение потребностей на основе исследования
5. **Эскалация:** Автоматическое определение необходимости эскалации

---
