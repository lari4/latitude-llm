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

## Пайплайн финансового анализа

### Общее описание

Система для анализа фондового рынка в реальном времени. Координатор оркестрирует два специализированных агента: один собирает новости и анализирует настроения через веб-поиск, другой получает ценовые данные и рассчитывает технические индикаторы. Результаты синтезируются в комплексный финансовый отчет с рекомендациями.

### Схема пайплайна (Stock Market Analysis)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STOCK MARKET ANALYSIS PIPELINE                        │
└─────────────────────────────────────────────────────────────────────────┘

                        ┌─────────────────────┐
                        │   USER REQUEST      │
                        │  Stock symbols: []  │
                        │  Market focus       │
                        │  Analysis depth     │
                        └──────────┬──────────┘
                                   │
                     ┌─────────────▼──────────────┐
                     │  Financial Coordinator     │
                     │  (OpenAI GPT-4.1)          │
                     │  Temperature: 0.2          │
                     │                            │
                     │  Analyzes requirements     │
                     │  Coordinates data          │
                     └─────────────┬──────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
          ┌─────────▼─────────┐  ┌▼──────────────┐
          │ Market Researcher │  │ Price Analyzer│
          │ (OpenAI GPT-4o)   │  │ (GPT-4o-mini) │
          └─────────┬─────────┘  └───┬───────────┘
                    │                │
      [Web Search]  │                │  [Yahoo Finance]
      [Extract]     │                │  [Code Execution]
                    │                │
          ┌─────────▼─────────────────▼───────────┐
          │      NEWS & SENTIMENT     │   PRICES  │
          │                           │           │
          │  • Breaking news          │  • Current│
          │  • Analyst reports        │  • Change │
          │  • Market sentiment       │  • Volume │
          │  • Sector developments    │  • RSI    │
          │  • Regulatory changes     │  • MACD   │
          │                           │  • SMA/EMA│
          └─────────┬─────────────────┴───────────┘
                    │
          ┌─────────▼──────────────┐
          │  SYNTHESIS & ANALYSIS  │
          │  (Financial Coord.)    │
          │                        │
          │  • Combines data       │
          │  • Identifies trends   │
          │  • Generates insights  │
          └─────────┬──────────────┘
                    │
          ┌─────────▼──────────────┐
          │    FINAL REPORT        │
          │                        │
          │  {                     │
          │    market_summary,     │
          │    stock_analysis[],   │
          │    market_trends[],    │
          │    recommendations[]   │
          │  }                     │
          └────────────────────────┘
```

### Потоки данных

#### 1. Входные данные

```javascript
{
  stock_symbols: string[],     // ["AAPL", "GOOGL", "MSFT"]
  market_focus: string,        // "tech sector" | "growth stocks"
  analysis_depth: string       // "quick" | "detailed" | "comprehensive"
}
```

#### 2. Market Researcher (Новости и настроения)

**Инструменты:**
- `latitude/search` - Веб-поиск финансовых новостей
- `latitude/extract` - Извлечение контента из источников

**Процесс:**
1. Поиск последних новостей по акциям/секторам
2. Извлечение контента из Bloomberg, Reuters, MarketWatch
3. Анализ настроений инвесторов
4. Выявление трендов и движущих сил рынка

**Возвращает:**
```javascript
{
  breaking_news: [{
    source: string,
    headline: string,
    summary: string,
    sentiment: "bullish" | "bearish" | "neutral",
    impact_score: number  // 1-10
  }],
  analyst_reports: [{
    analyst: string,
    rating: "buy" | "hold" | "sell",
    target_price: number,
    reasoning: string
  }],
  market_sentiment: {
    overall: "bullish" | "bearish" | "mixed",
    confidence: number,  // 0-1
    key_factors: string[]
  },
  sector_developments: string[],
  regulatory_changes: string[]
}
```

#### 3. Price Analyzer (Цены и технический анализ)

**Инструменты:**
- `latitude/code` - Выполнение Python кода для расчетов
- `latitude/search` - Поиск данных на Yahoo Finance

**Процесс:**
1. Поиск "Yahoo Finance [SYMBOL] stock price"
2. Извлечение текущих цен и исторических данных
3. Выполнение кода для расчета индикаторов
4. Анализ волатильности и объемов

**Возвращает:**
```javascript
{
  price_data: [{
    symbol: string,
    current_price: number,
    change: number,
    change_percent: number,
    volume: number,
    market_cap: number
  }],
  technical_indicators: [{
    symbol: string,
    rsi: number,              // 0-100
    macd: {
      macd_line: number,
      signal_line: number,
      histogram: number
    },
    moving_averages: {
      sma_20: number,
      sma_50: number,
      sma_200: number,
      ema_12: number,
      ema_26: number
    },
    bollinger_bands: {
      upper: number,
      middle: number,
      lower: number
    }
  }],
  signals: [{
    symbol: string,
    signal: "buy" | "sell" | "hold",
    strength: number,         // 0-1
    reasoning: string
  }]
}
```

**Пример кода для RSI:**
```python
import pandas as pd
import numpy as np

def calculate_rsi(prices, period=14):
    delta = prices.diff()
    gain = (delta.where(delta > 0, 0)).rolling(window=period).mean()
    loss = (-delta.where(delta < 0, 0)).rolling(window=period).mean()
    rs = gain / loss
    rsi = 100 - (100 / (1 + rs))
    return rsi
```

#### 4. Synthesis (Financial Coordinator)

**Логика генерации рекомендаций:**

```
FOR each stock:
  // Комбинировать сигналы
  technical_score = calculate_technical_score(rsi, macd, moving_averages)
  sentiment_score = calculate_sentiment_score(news, analyst_reports)

  // Взвешивание
  combined_score = (technical_score * 0.6) + (sentiment_score * 0.4)

  IF combined_score > 7:
    recommendation = "Strong Buy"
  ELSE IF combined_score > 5:
    recommendation = "Buy"
  ELSE IF combined_score > 3:
    recommendation = "Hold"
  ELSE IF combined_score > 1:
    recommendation = "Sell"
  ELSE:
    recommendation = "Strong Sell"
```

**Финальный выход:**
```javascript
{
  market_summary: string,  // Executive summary

  stock_analysis: [{
    symbol: string,
    current_price: number,
    price_change: number,
    sentiment: "bullish" | "bearish" | "neutral",
    key_news: string[],
    technical_signals: string[],
    analyst_consensus: string
  }],

  market_trends: [
    "Tech sector showing strong momentum with AI developments",
    "Rising interest rate concerns affecting growth stocks",
    "Increased volatility in semiconductor stocks"
  ],

  recommendations: [{
    action: "buy" | "sell" | "hold",
    symbol: string,
    reasoning: string,
    confidence: number,    // 0-1
    entry_price: number,
    target_price: number,
    stop_loss: number
  }]
}
```

### Конфигурация моделей

| Компонент | Провайдер | Модель | Temperature | Причина выбора |
|-----------|-----------|--------|-------------|----------------|
| Financial Coordinator | OpenAI | gpt-4.1 | 0.2 | Точный финансовый анализ с низкой вариативностью |
| Market Researcher | OpenAI | gpt-4o | default | Понимание контекста новостей и настроений |
| Price Analyzer | OpenAI | gpt-4o-mini | default | Быстрые количественные расчеты |

### Пример выполнения

```
=== ВХОД ===
stock_symbols: ["AAPL"]
market_focus: "tech sector"
analysis_depth: "detailed"

=== MARKET RESEARCHER ===
{
  breaking_news: [{
    source: "Bloomberg",
    headline: "Apple announces new AI features in iOS",
    sentiment: "bullish",
    impact_score: 8
  }],
  analyst_reports: [{
    analyst: "Morgan Stanley",
    rating: "buy",
    target_price: 220,
    reasoning: "Strong iPhone 16 pre-orders"
  }],
  market_sentiment: {
    overall: "bullish",
    confidence: 0.85,
    key_factors: ["AI innovation", "Strong demand", "Services growth"]
  }
}

=== PRICE ANALYZER ===
{
  price_data: {
    symbol: "AAPL",
    current_price: 185.50,
    change: +2.30,
    change_percent: +1.26%,
    volume: 58M
  },
  technical_indicators: {
    rsi: 62,  // Slightly overbought but not extreme
    macd: {
      macd_line: 1.2,
      signal_line: 0.8,
      histogram: 0.4  // Positive = bullish
    },
    moving_averages: {
      sma_20: 183.0,  // Price above = bullish
      sma_50: 178.5,
      ema_12: 184.2
    }
  },
  signals: [{
    signal: "buy",
    strength: 0.75,
    reasoning: "Price above all major MAs, positive MACD crossover"
  }]
}

=== SYNTHESIS ===
{
  market_summary: "AAPL shows strong bullish momentum driven by AI announcements
                   and positive technical signals. Analyst consensus is Buy with
                   average target $220.",

  stock_analysis: [{
    symbol: "AAPL",
    current_price: 185.50,
    price_change: +2.30,
    sentiment: "bullish",
    key_news: [
      "New AI features announced",
      "Strong pre-order numbers",
      "Analyst upgrades"
    ]
  }],

  market_trends: [
    "AI integration driving tech sector momentum",
    "Apple ecosystem strength continues",
    "Premium smartphone demand resilient"
  ],

  recommendations: [{
    action: "buy",
    symbol: "AAPL",
    reasoning: "Strong technical and fundamental alignment. Positive momentum
                with upcoming product cycle. RSI not yet overbought.",
    confidence: 0.85,
    entry_price: 185.50,
    target_price: 220.00,
    stop_loss: 175.00
  }]
}
```

---

## Пайплайн оценки (LLM-as-a-Judge)

### Общее описание

Система автоматической оценки качества AI-ответов с использованием LLM в качестве судьи. Оценочные промпты получают ответ от другой модели вместе с контекстом и критериями, затем выносят вердикт с объяснением.

### Схема пайплайна

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      LLM-AS-A-JUDGE EVALUATION PIPELINE                  │
└─────────────────────────────────────────────────────────────────────────┘

                        ┌─────────────────────┐
                        │  EVALUATION INPUT   │
                        │                     │
                        │  • actualOutput     │
                        │  • conversation     │
                        │  • criteria         │
                        │  • expectedOutput   │
                        │  • metadata         │
                        └──────────┬──────────┘
                                   │
                     ┌─────────────▼──────────────┐
                     │   EVALUATION TYPE?         │
                     └─┬────────┬────────┬────────┘
                       │        │        │
         ┌─────────────▼──┐  ┌─▼─────┐ │
         │ Binary         │  │Rating │ │
         │ (Pass/Fail)    │  │(Scale)│ │
         └─────────────┬──┘  └─┬─────┘ │
                       │       │        │
                       │       │    ┌───▼────────┐
                       │       │    │Comparison  │
                       │       │    │(0-100)     │
                       │       │    └───┬────────┘
                       │       │        │
                       └───┬───┴────────┴─────┐
                           │                  │
                  ┌────────▼──────────┐      │
                  │  EVALUATOR LLM    │      │
                  │                   │      │
                  │  Analyzes:        │      │
                  │  • Response       │      │
                  │  • Context        │      │
                  │  • Tool calls     │      │
                  │  • Metadata       │      │
                  │                   │      │
                  │  Applies:         │      │
                  │  • Criteria       │      │
                  │  • Logic          │      │
                  │  • Judgment       │      │
                  └────────┬──────────┘      │
                           │                  │
                  ┌────────▼──────────────────▼───┐
                  │   STRUCTURED OUTPUT           │
                  │                               │
                  │  Binary: {passed, reason}     │
                  │  Rating: {rating, reason}     │
                  │  Comparison: {score, reason}  │
                  └────────┬──────────────────────┘
                           │
                  ┌────────▼──────────┐
                  │  NORMALIZATION    │
                  │  & SCORING        │
                  │                   │
                  │  • Apply reverse  │
                  │  • Check thresh.  │
                  │  • Calculate final│
                  └────────┬──────────┘
                           │
                  ┌────────▼──────────┐
                  │  EVALUATION       │
                  │  RESULT           │
                  │                   │
                  │  {                │
                  │    score,         │
                  │    normalizedScore│
                  │    hasPassed,     │
                  │    reason,        │
                  │    metadata       │
                  │  }                │
                  └───────────────────┘
```

### Типы оценки

#### 1. Binary Evaluation (Бинарная)

**Вход:**
```javascript
{
  actualOutput: string,      // Ответ для оценки
  conversation: string,      // Контекст разговора
  criteria: string,          // "Response must be polite and helpful"
  passDescription: string,   // "Response meets all criteria"
  failDescription: string,   // "Response violates criteria"
  toolCalls: array,          // Вызовы инструментов (если есть)
  metadata: {
    cost: number,
    tokens: number,
    duration: number
  }
}
```

**Промпт для LLM:**
```
You're an expert LLM-as-a-judge evaluator. Judge whether the response meets:
${criteria}

Verdict is `true` if meets criteria, `false` otherwise:
- `true` = "${passDescription}"
- `false` = "${failDescription}"

Response to evaluate: ${actualOutput}
Context: ${conversation}
Tool calls: ${toolCalls}
Metadata: Cost ${cost}, Tokens ${tokens}, Duration ${duration}

Output JSON: {passed: boolean, reason: string}
```

**Выход:**
```javascript
{
  score: 0 | 1,            // 0 = fail, 1 = pass
  normalizedScore: 0 | 1,  // Может быть reversed если reverseScale=true
  hasPassed: boolean,
  reason: string,
  metadata: { ... }
}
```

#### 2. Rating Evaluation (По шкале)

**Вход:**
```javascript
{
  actualOutput: string,
  conversation: string,
  criteria: string,          // "Rate response helpfulness"
  minRating: number,         // 1
  minRatingDescription: string,  // "Not helpful at all"
  maxRating: number,         // 5
  maxRatingDescription: string,  // "Extremely helpful"
  minThreshold: number,      // 3 (optional)
  maxThreshold: number       // 5 (optional)
}
```

**Промпт для LLM:**
```
You're an expert LLM-as-a-judge evaluator. Judge whether the response meets:
${criteria}

Verdict is integer between `${minRating}` and `${maxRating}`:
- `${minRating}` = "${minRatingDescription}"
- `${maxRating}` = "${maxRatingDescription}"

Response to evaluate: ${actualOutput}
Context: ${conversation}

Output JSON: {rating: number, reason: string}
```

**Выход:**
```javascript
{
  score: number,           // Raw rating (e.g., 4)
  normalizedScore: number, // 0-1 scale (e.g., 0.75)
  hasPassed: boolean,      // Based on thresholds
  reason: string,
  metadata: { ... }
}
```

#### 3. Comparison Evaluation (Сравнительная)

**Вход:**
```javascript
{
  actualOutput: string,
  expectedOutput: string,   // Эталонный ответ для сравнения
  conversation: string,
  criteria: string,         // "Compare similarity to expected output"
  passDescription: string,  // "Perfect match"
  failDescription: string,  // "No similarity"
  minThreshold: number,     // 70 (optional)
  maxThreshold: number      // 100 (optional)
}
```

**Промпт для LLM:**
```
You're an expert LLM-as-a-judge evaluator. Judge how well the response compares
to expected output, following:
${criteria}

Expected output to compare against:
${expectedOutput}

Verdict is integer between `0` (poor match) and `100` (perfect match):
- `0` = "${failDescription}"
- `100` = "${passDescription}"

Response to evaluate: ${actualOutput}
Context: ${conversation}

Output JSON: {score: number, reason: string}
```

**Выход:**
```javascript
{
  score: number,           // 0-100
  normalizedScore: number, // 0-1 scale
  hasPassed: boolean,      // Based on thresholds
  reason: string,
  metadata: { ... }
}
```

### Процесс оценки

```
1. BUILD PROMPT
   ├─ Select evaluation type (binary/rating/comparison)
   ├─ Format criteria and descriptions
   ├─ Include actualOutput, conversation, toolCalls
   └─ Add metadata (cost, tokens, duration)

2. EXECUTE EVALUATION
   ├─ Parse prompt with promptl-ai
   ├─ Validate against schema
   ├─ Run through AI provider
   └─ Extract structured output

3. PROCESS RESULT
   ├─ Validate output format
   ├─ Apply reverseScale (if configured)
   ├─ Normalize score to 0-1 range
   ├─ Check against thresholds
   └─ Determine hasPassed status

4. RETURN EVALUATION
   └─ {score, normalizedScore, hasPassed, reason, metadata}
```

### Доступные параметры для кастомной оценки

```javascript
{
  actualOutput: string,      // Фактический ответ
  expectedOutput: string,    // Ожидаемый ответ
  conversation: string,      // История разговора
  messages: array,           // Массив сообщений
  toolCalls: array,          // Вызовы инструментов
  prompt: string,            // Оригинальный промпт
  config: object,            // Конфигурация
  parameters: object,        // Параметры запуска
  context: object,           // Дополнительный контекст
  response: object,          // Полный ответ
  cost: number,              // Стоимость (центы)
  tokens: number,            // Количество токенов
  duration: number           // Продолжительность (сек)
}
```

### Примеры использования

#### Пример 1: Binary - Проверка вежливости

```
Вход:
actualOutput: "Here's the answer to your stupid question..."
criteria: "Response must be professional and respectful"
passDescription: "Response is polite and professional"
failDescription: "Response is rude or unprofessional"

Оценка LLM:
{
  passed: false,
  reason: "Response contains disrespectful language ('stupid question')
           which violates professional communication standards"
}

Результат:
{
  score: 0,
  normalizedScore: 0,
  hasPassed: false,
  reason: "..."
}
```

#### Пример 2: Rating - Оценка полезности

```
Вход:
actualOutput: "To reset your password, go to Settings > Account >
               Security and click 'Reset Password'. You'll receive an
               email with instructions."
criteria: "Rate how helpful and clear the response is"
minRating: 1, maxRating: 5
minThreshold: 3

Оценка LLM:
{
  rating: 5,
  reason: "Response is exceptionally clear with step-by-step instructions
           and mentions the email confirmation, covering all aspects"
}

Результат:
{
  score: 5,
  normalizedScore: 1.0,
  hasPassed: true,
  reason: "..."
}
```

#### Пример 3: Comparison - Сравнение с эталоном

```
Вход:
actualOutput: "The capital of France is Paris."
expectedOutput: "Paris is the capital and largest city of France."
criteria: "Compare factual accuracy and completeness"

Оценка LLM:
{
  score: 85,
  reason: "Factually accurate but less complete than expected output.
           Missing 'largest city' information but core fact is correct."
}

Результат:
{
  score: 85,
  normalizedScore: 0.85,
  hasPassed: true,  // Если minThreshold < 85
  reason: "..."
}
```

### Особенности пайплайна

1. **Структурированный выход:** JSON schema validation для консистентности
2. **Контекстная осведомленность:** Учет всего разговора, не только ответа
3. **Метаданные:** Включение cost, tokens, duration в оценку
4. **Гибкость:** 4 типа оценки + кастомные промпты
5. **Explainability:** Каждая оценка с подробным объяснением

---

## Базовая инфраструктура агентов

### Общие концепции

#### 1. Agent Type Prompt
```promptl
type: agent
agents:
  - agent_name_1
  - agent_name_2
```

#### 2. Tool Integration
```javascript
tools:
  - tool_name:
      description: "What the tool does"
      parameters:
        type: object
        properties: { ... }
```

#### 3. Schema Validation
```javascript
schema:
  type: object
  properties: { ... }
  required: [...]
```

#### 4. Step-based Execution
```promptl
<step agents={{ ["agent_name"] }}>
  Instructions for this step
</step>
```

### Сервисы оркестрации

**Файл:** `packages/core/src/services/chains/run.ts`

Основной раннер цепочек промптов:
- Парсинг PromptL документов
- Валидация конфигурации
- Выполнение через провайдеров
- Кеширование результатов

**Файл:** `packages/core/src/services/agents/agentsAsTools.ts`

Конвертация агентов в инструменты:
- Агенты могут вызывать другие агенты как tools
- Автоматическое построение tool schemas
- Передача контекста между агентами

### Поток выполнения агента

```
1. USER REQUEST
   └─> Parse PromptL document

2. VALIDATE CONFIGURATION
   ├─> Check provider/model
   ├─> Validate schema
   └─> Verify agents/tools

3. BUILD AGENT CHAIN
   ├─> Convert agents to tools
   ├─> Setup tool handlers
   └─> Prepare context

4. EXECUTE STEPS
   ├─> Step 1: agent_1
   │   ├─> Call tool_a
   │   ├─> Call tool_b
   │   └─> Return result
   │
   ├─> Step 2: agent_2
   │   ├─> Receive agent_1 result
   │   ├─> Process data
   │   └─> Return result
   │
   └─> Step N: final synthesis

5. RETURN RESULT
   └─> Structured output with schema
```

### Конфигурация провайдеров

```typescript
// Автоматический выбор температуры
temperature: model.startsWith('gpt-5') ? 1 : 0.7

// Поддерживаемые провайдеры
providers: [
  'openai',
  'anthropic',
  'google',
  'perplexity',
  'groq',
  'deepseek',
  'mistral',
  'xai',
  'amazonBedrock',
  'vertexGoogle',
  'vertexAnthropic',
  'custom'
]
```

---

## Заключение

Документация охватывает 4 основных агентных пайплайна:

1. **Модерация контента** - 3 параллельных агента с синтезом
2. **Поддержка клиентов** - Последовательный пайплайн исследования и создания
3. **Финансовый анализ** - Комбинирование новостей и технических данных
4. **Оценка (LLM-as-a-Judge)** - 3 типа автоматической оценки

Каждый пайплайн демонстрирует:
- Четкое разделение ответственности между агентами
- Специализацию агентов для конкретных задач
- Структурированные потоки данных
- Синтез результатов в полезные выходы
- Конфигурацию моделей с обоснованием выбора
