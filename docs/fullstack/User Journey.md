## MVP Flow

```text
Create Workspace
      ↓
Define Research Question
      ↓
Add / Find Sources
      ↓
Run Research
      ↓
Review Evidence
      ↓
Validate Claims
      ↓
Create Decision
      ↓
Generate Decision Brief
      ↓
Review / Accept Decision
      ↓
Ask Workspace / Follow-up Question
```

## Steps

### 1. Create Workspace
Пользователь создаёт Workspace и задаёт название и описание исследовательской области.

### 2. Define Research Question
Пользователь создаёт Research Task и формулирует конкретный исследовательский вопрос.

### 3. Add / Find Sources
Пользователь добавляет или находит источники: URL, PDF, Doc, GitHub, видео или manual note.

### 4. Run Research
Пользователь запускает контролируемый Research Run. Agent помогает найти и структурировать информацию.

### 5. Review Evidence
Пользователь проверяет Evidence Items — конкретные фрагменты источников, на которых основаны результаты исследования.

### 6. Validate Claims
Пользователь проверяет Claims и определяет, какие утверждения подтверждены, слабы, конфликтуют или устарели.

### 7. Create Decision
На основе проверенных Claims пользователь формулирует Decision: что решили и почему.

### 8. Generate Decision Brief
Система формирует короткий итог исследования:
- Decision
- Rationale
- Key Claims
- Supporting Evidence
- Confidence
- Open Questions

### 9. Review / Accept Decision
Пользователь финально проверяет Decision и принимает его либо отправляет на повторный review.

### 10. Ask Workspace (Follow-up Question)
Пользователь может задать вопрос по накопленной памяти Workspace (Sources / Evidence / Claims / Decisions).
Если памяти недостаточно — система предлагает создать Follow-up Research Task, который переиспользует flow из шагов 2–3.

## Product Principle
> Agent помогает найти и структурировать информацию. Только проверенные человеком evidence, claims и decisions становятся знанием команды.