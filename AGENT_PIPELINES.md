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
