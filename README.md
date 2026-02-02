# Frontend Test – QuestionCard

## Part 1. Component Architecture

- **QuestionCardContainer**
  - **Header** (номер, таймер, режим demo)
  - **QuestionRenderer**
    - Fallback (если TipTap/KaTeX упали)
  - **AnswerOptions**
    - AnswerOptionItem
      - type: `single` | `multiple`
      - control: `radio` | `checkbox`
  - **ActionBar**
    - PrimaryAction
      - CheckAnswerButton
      - NextQuestionButton
    - SecondaryAction
      - RetryButton (при ошибке)
    - StatusLine
      - LoadingIndicator
      - ErrorMessage
  - **ExplanationBlock** (conditional)
    - ExplanationContent
    - DemoOverlay (blur + CTA)

### State

**Глобально хранятся:**
1. данные вопроса (stem TipTap JSON, options, правильный ответ если приходит, explanation, флаги demo)
2. loading/error для запроса вопроса
3. userMode (demo или заплатил)

**Локально внутри QuestionCard:**
1. selectedAnswerIds — выбранные пользователем варианты ответа
2. isChecked — пользователь нажал “Проверить ответ” и получил результат
3. isChecking
4. checkResult
5. isCorrect
6. checkError
7. renderError — показать fallback UI без краша всей карточки

**Где хранится selectedAnswer:** локально в QuestionCard  
**Где хранится isChecked:** локально в QuestionCard

### Что сбрасывается при смене questionId:

- selectedAnswerIds = empty Set
- isChecked = false
- checkResult = null
- isChecking = false
- renderError = null
- checkError = null

### Что будет, если пользователь кликает очень быстро:

**Проблема 1: гонка ответов**  
Ответ от старого questionId прилетел позже и перезаписал state.  
**Решение:** принимать результат только если `token === activeRequestToken` и `questionId` тот же.

**Проблема 2: двойной check**  
Многократные клики по кнопке.  
**Решение:** `isChecking=true` → кнопки disabled.

**Проблема 3: API вернул ошибку**  
**Решение:** показать inline error + “Повторить проверку”, не показывать explanation до успешного check.

## Part 2. Pseudocode Logic

```js
state:
  selectedAnswerIds: Set<string>
  isChecked: boolean
  isChecking: boolean
  checkResult: { isCorrect: boolean, explanation?: TipTapJson } | null
  checkError: string | null

  activeToken: number // токен последнего актуального запроса
  demoMode: boolean
  question: { id, stemJson, options[], ... }

derived state:
    hasSelection = selectedAnswerIds.size > 0

    disableCheckButton = isChecking || !hasSelection

    showExplanation = isChecked && checkResult != null


Выбор ответа:
    onSelectAnswer(answerId):
    if (isChecking):
        return

    if (question.type === "single"):
        selectedAnswerIds = new Set([answerId])

    if (question.type == "multiple"):
        if (selectedAnswerIds.has(answerId)):
        selectedAnswerIds.remove(answerId)
        else:
        selectedAnswerIds.add(answerId)

    // если пользователь меняет ответ после проверки
    if (isChecked):
        isChecked = false
        checkResult = null
        checkError = null


Проверка ответа:
    onCheckAnswer():
        if (disableCheckButton):
        return

    isChecking = true
    checkError = null

    localToken = activeRequestToken

    response = api.checkAnswer({
        questionId,
        answers: Array.from(selectedAnswerIds)
    })

    if (localToken != activeRequestToken):
        return   // устаревший ответ, игнорируем

    if (response.error):
        isChecking = false
        checkError = "Не удалось проверить ответ"
        return

    isChecking = false
    isChecked = true
    checkResult = response


Показ explanation:
    renderExplanation():
    if (!showExplanation):
        return null

    if (demoMode):
        return BlurredExplanationWithCTA

    return ExplanationContent


Смена вопроса:
    onQuestionChange(newQuestionId):
    activeRequestToken++

    selectedAnswerIds = empty Set
    isChecked = false
    isChecking = false
    checkResult = null
    checkError = null


ActionBar:
    if (!isChecked):
    render CheckAnswerButton (disabled = disableCheckButton)

    if (isChecking):
    render LoadingIndicator

    if (checkError):
    render ErrorMessage + RetryButton

    if (isChecked):
    render NextQuestionButton

```

## Part 3. Edge Cases & UX

### 1) Explanation отсутствует
**Что происходит:**  
Пояснение не приходит с бэкенда.

**Как ведёт себя UI:**  
- После **Check Answer** показываем результат как обычно (✅ / ❌)
- Вместо пояснения рендерим placeholder  
  > “Пояснение пока недоступно.”

---

### 2) В stem только формулы
**Что происходит:**  
Вопрос состоит только из математических формул.

**Как ведёт себя UI:**  
- `QuestionStem` рендерит KaTeX как основной контент
- Если формула слишком широкая:
  - контейнер получает `overflow-x: auto`
  - появляется горизонтальный скролл

---

### 3) В stem очень длинный текст
**Что происходит:**  
Вопрос содержит большой объём текста.

**Как ведёт себя UI:**  
- Текст отображается полностью
- UI остаётся управляемым за счёт:
  - ограничения максимальной ширины
  - переноса строк
  - адекватного `line-height`

---

### 4) KaTeX упал с ошибкой
**Что происходит:**  
KaTeX выбрасывает ошибку при рендере формулы.

**Как ведёт себя UI:**  
- Карточка вопроса не падает целиком
- В месте формулы показываем fallback:
  > “Не удалось отобразить формулу.”


### 5) Пользователь меняет ответ после Check Answer
**Что происходит:**  
Пользователь пытается изменить выбранный ответ после проверки.

**UX-варианты (выбираем один и используем везде):**

#### Вариант A — строгий / экзаменационный
- После **Check Answer** варианты ответа становятся `disabled`
- Пользователь не может изменить ответ
- Показываем подсказку:  
  > “Ответ уже проверен. Перейдите к следующему вопросу.”

#### Вариант B — обучающий / learning
- Разрешаем менять ответ после проверки

**При изменении ответа:**
- `isChecked = false`
- скрываем explanation
- убираем результат проверки
- кнопка **Check Answer** снова активна
- показываем подсказку:  
  > “Вы изменили ответ — нажмите Проверить.”

---

### 6) Пользователь в demo-режиме
**Что происходит:**  
Пользователь не оплатил полный доступ.

**После Check Answer:**
- Результат проверки (✅ / ❌) **можно показывать** (если продукт это позволяет)

**Explanation:**
- либо полностью скрыт
- либо отображается заблюренным (blur overlay)
- при этом место под пояснение видно, чтобы было понятно, что контент существует

**Обязательный поясняющий текст:**  
> “Разбор решения доступен в полной версии.  
> В демо-режиме пояснения скрыты.”

**CTA (обязателен):**
- Кнопка: **“Открыть доступ”** или **“Перейти к оплате”**
    CTA на оплату (обязателен):
        Кнопка: "Открыть доступ" или "Перейти к оплате"



