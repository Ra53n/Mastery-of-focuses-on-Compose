![[image_preview_2.png]]

Привет! Меня зовут Костя, я Android-разработчик в онлайн-кинотеатре PREMIER. В процессе работы над нашим проектом PREMIER для AndroidTV я столкнулся с тем, что в Jetpack Compose механизм фокусов — достаточно сложная и неочевидная тема. А информации об этом в интернете очень мало, особенно для специфичных сценариев вроде TV-приложений или устройств без сенсорного ввода.

Поэтому я решил разобрать тему фокусов в Compose максимально подробно, чтобы помочь разработчикам лучше понять этот механизм и избежать типичных ошибок.

Фокусировка в Jetpack Compose — это не просто перемещение курсора между элементами. За этим процессом стоит сложная система нод, модификаторов и алгоритмов, которые определяют, какой элемент должен получить фокус в каждый момент времени. 

В первой части статьи мы разобрали базовую структуру фокусировки в Compose и то, как элементы обрабатывают запросы фокуса. Теперь пришло время углубиться в технические детали: что именно происходит, когда вызывается `requestFocus()`, как Compose выбирает элемент для фокусировки и какие изменения были внесены в Compose 1.8, чтобы улучшить этот процесс. 

Если вы работаете с приложениями под AndroidTV, кастомными компонентами или просто хотите лучше понимать, как работает система фокусов, эта статья для вас!

Серия статей:

1. Мастерство фокусов на Compose
2. Мастерство фокусов на Compose 2: Как работает запрос фокуса (текущая)

Содержание

- Как работает запрос фокуса в Compose?
	- Взаимодействие с нодами и модификаторами
	- Запрос фокуса через FocusRequester
	- Что происходит под капотом?
	- Почему это важно?
- Разбор механизма запроса фокуса на Compose
	- Поиск фокусируемого элемента
	- Установка фокуса на элементе
	- Состояния фокуса в Compose
	- Обновление работы фокуса на Compose версии 1.8
- Заключение

## Как работает запрос фокуса в Compose?

Когда мы вызываем `requestFocus()`, кажется, что фокус просто переходит к нужному элементу, но на самом деле за этим стоит сложный механизм. Compose должен определить, может ли элемент принять фокус, как это повлияет на другие элементы, и какой путь для передачи фокуса выбрать. Давай разберёмся, как это устроено и какие компоненты участвуют в этом процессе.

#### Взаимодействие с нодами и модификаторами

В **Jetpack Compose** каждый элемент интерфейса представлен в виде **ноды** (**Node**). Эти ноды образуют **UI-дерево** и отвечают за различные аспекты элемента:

- **Размещение и измерение** — `LayoutNode`
- **Обработку ввода и событий** — `ModifierNode`
- **Фокус и его состояние** — `FocusTargetNode`
- **Отрисовку элемента** — `DrawNode`

Когда мы добавляем модификатор `Modifier.focusTarget`, Compose создаёт специальную фокус-ноду, которая может принимать и обрабатывать фокус. А модификатор `Modifier.focusRequester` связывает ноду с объектом `FocusRequester`, который и управляет запросами фокуса.

Если хочется глубже погрузиться в архитектуру нод, рекомендую [документацию Android по фазам Compose](https://developer.android.com/develop/ui/compose/phases) — там детально разобраны этапы построения UI-дерева.

#### Запрос фокуса через FocusRequester

Теперь давайте разберёмся, как программно установить фокус на элементе. Для этого используется класс `FocusRequester`. Он работает как «посредник» между фокус-нодой элемента и системой фокусировки в Compose.

Типичный сценарий выглядит так:

```kotlin
val focusRequester = FocusRequester()

TextField(
    value = text,
    onValueChange = { text = it },
    modifier = Modifier
        .focusRequester(focusRequester)
        .focusTarget()
)

Button(onClick = { focusRequester.requestFocus() }) {
    Text("Получить фокус")
}

```

- Мы создаём экземпляр `FocusRequester`.
- Привязываем его к элементу с помощью `Modifier.focusRequester`.
- Добавляем `Modifier.focusTarget`, чтобы элемент мог принимать фокус.
- При нажатии на кнопку вызываем `focusRequester.requestFocus()`, и фокус переходит на `TextField`.

> ⚠️ **Важно:** Если `FocusRequester` не привязан к ноде через `focusRequester`, вызов `requestFocus()` выбросит исключение.

#### Что происходит под капотом?

Когда мы вызываем `requestFocus()`, Compose делает следующее:

1. **Проверка дерева нод:** Находится ближайшая фокус-нода, связанная с `FocusRequester`.
2. **Проверка состояния фокуса:** Если элемент может принимать фокус, вызывается метод `onFocusChanged`, который обновляет состояние элемента.
3. **Обновление фокусного элемента:** Текущий фокусный элемент теряет фокус (если он был активен), а новый элемент получает его.
4. **Перерисовка UI:** Compose триггерит перерисовку, чтобы обновить визуальное состояние элемента (например, подсветку рамки).

В результате фокус плавно переключается на нужный элемент — либо по нажатию кнопки, либо программно, в ответ на какое-то событие.

#### Почему это важно?

Понимание устройства фокуса и работы `FocusRequester` критично для сложных сценариев:

- **Кастомная клавиатурная навигация:** Например, на TV или планшетах.
- **Формы и валидация:** Автоматический переход к первому полю с ошибкой.
- **Доступность:** Обеспечение правильной последовательности фокусировки для screen reader'ов.

В следующем разделе мы разберём, как механизм фокуса эволюционировал в версиях Compose — посмотрим, как всё работало до 1.7, и какие улучшения появились в 1.8. Это поможет понять, почему фокус иногда может вести себя неожиданно и как с этим справляться.

## Разбор механизма запроса фокуса на Compose

#### Как работает requestFocus()

На первый взгляд, вызов `requestFocus()` выглядит достаточно просто:

```kotlin
fun requestFocus() {  
    focus()  
}
```

Но за этим вызовом скрыта сложная цепочка функций, которая ищет подходящую ноду для фокусировки и обрабатывает вложенные структуры интерфейса.
#### Вызов focus()

Метод `requestFocus()` делегирует свою работу внутренней функции `focus()`. Вот как она выглядит:
```kotlin
internal fun focus(): Boolean = findFocusTargetNode { it.requestFocus() }
```

🔸 **Что делает функция focus():**
- Ищет ноду, которая может принять фокус.
- Если нода найдена, вызывает у неё `requestFocus`.
- Возвращает `true`, если фокус был успешно установлен.

Теперь давайте разберёмся, как именно Compose ищет эту ноду.

#### FocusableNode и FocusTargetModifierNode

Ключевая часть механизма фокуса — это **FocusableNode**. Этот класс создаёт специальную ноду, которая может принимать фокус:
```kotlin
internal class FocusableNode(  
    interactionSource: MutableInteractionSource?  
) : DelegatingNode(), FocusEventModifierNode, SemanticsModifierNode,  
    GlobalPositionAwareModifierNode, FocusRequesterModifierNode {  

    ...
  
    init {  
        delegate(FocusTargetModifierNode())  
    }
    
    ...
    
```

🔸 **Как это работает:**
- При инициализации нода делегирует управление фокусом через `FocusTargetModifierNode`.
- Этот модификатор делает элемент доступным для поиска фокуса (например, при использовании `Modifier.focusable()` или `Modifier.focusTarget()`).
- Нода становится частью системы фокусировки Compose и может быть найдена в процессе обхода иерархии элементов.

Теперь разберёмся, как происходит сам поиск фокусируемой ноды.

### Поиск фокусируемого элемента

Функция `findFocusTargetNode` отвечает за поиск подходящего элемента для фокусировки:

```kotlin
internal fun findFocusTargetNode(onFound: (FocusTargetNode) -> Boolean): Boolean {  
    @OptIn(ExperimentalComposeUiApi::class)  
    return findFocusTarget { focusTarget ->  
        if (focusTarget.fetchFocusProperties().canFocus) {  
            onFound(focusTarget)  
        } else {  
            focusTarget.findChildCorrespondingToFocusEnter(Enter, onFound)  
        }  
    }  
}
```
🔸 **Как работает функция:**
1. **Поиск фокусируемого элемента:** Перебираются все ноды, связанные с `FocusRequester`, в поисках более подходящей для фокуса. Подробнее об этом будет дальше.
2. **Проверка свойства canFocus:** Если у ноды `canFocus = true`, она сразу получает фокус.
3. **Поиск среди дочерних элементов:** Если нода не может принять фокус, поиск продолжается среди её потомков с помощью `findChildCorrespondingToFocusEnter`.

Теперь посмотрим, как именно осуществляется этот обход.
#### Поиск фокусируемой ноды: findFocusTarget

Функция `findFocusTarget` проходит по всем нодам, связанным с `FocusRequester`, и ищет подходящую для фокуса:

```kotlin
private inline fun findFocusTarget(onFound: (FocusTargetNode) -> Boolean): Boolean {  
    check(this !== Default) { InvalidFocusRequesterInvocation }  
    check(this !== Cancel) { InvalidFocusRequesterInvocation }  
    check(focusRequesterNodes.isNotEmpty()) { FocusRequesterNotInitialized }  
    var success = false  
    focusRequesterNodes.forEach { node ->  
        node.visitChildren(Nodes.FocusTarget) {  
            if (onFound(it)) {  
                success = true  
                return@forEach  
            }  
        }  
    }    return success  
}
```

🔸 **Основные шаги:**
1. **Проверка инициализации:** Убеждаемся, что у `FocusRequester` есть ноды, к которым он привязан.
2. **Обход дочерних нод:** С помощью `visitChildren` проверяются все дочерние элементы.
3. **Остановка при успехе:** Если хотя бы одна нода принимает фокус, обход завершается.

Но что, если текущая нода не подходит? Тогда в ход идёт более сложный алгоритм поиска, который лежит в вызове функции `findChildCorrespondingToFocusEnter`.

#### Поиск дочернего кандидата: findChildCorrespondingToFocusEnter

Если текущая нода не может принять фокус, Compose начинает искать подходящего кандидата среди дочерних элементов:

```kotlin
internal fun FocusTargetNode.findChildCorrespondingToFocusEnter(  
    direction: FocusDirection,  
    onFound: (FocusTargetNode) -> Boolean  
): Boolean {  
  
    val focusableChildren = MutableVector<FocusTargetNode>()  
    collectAccessibleChildren(focusableChildren)  
  
    if (focusableChildren.size <= 1) {  
        return focusableChildren.firstOrNull()?.let { onFound.invoke(it) } ?: false  
    }  
	val requestedDirection = when (direction) {
        @OptIn(ExperimentalComposeUiApi::class)  
        Enter -> Right  
        else -> direction  
    }  
   
    val initialFocusRect = when (requestedDirection) {  
        Right, Down -> focusRect().topLeft()  
        Left, Up -> focusRect().bottomRight()  
        else -> error(InvalidFocusDirection)  
    }  
    val nextCandidate = focusableChildren.findBestCandidate(initialFocusRect, requestedDirection)  
    return nextCandidate?.let { onFound.invoke(it) } ?: false  
}
```

#### Алгоритм поиска кандидата:

```kotlin
internal fun FocusTargetNode.findChildCorrespondingToFocusEnter(  
    direction: FocusDirection,  
    onFound: (FocusTargetNode) -> Boolean  
): Boolean {  
    val focusableChildren = MutableVector<FocusTargetNode>()  
    collectAccessibleChildren(focusableChildren)

    if (focusableChildren.size <= 1) {  
        return focusableChildren.firstOrNull()?.let { onFound.invoke(it) } ?: false  
    }
```
1. **Сбор фокусируемых потомков:**
    Функция `collectAccessibleChildren` собирает всех дочерних нод, которые потенциально могут принять фокус (с проверкой на `canFocus`).
    
2. **Оптимизация для одного кандидата:**
    Если в контейнере всего один фокусируемый элемент, он сразу получает фокус.
    
3. **Определение направления фокуса:**  
	Для входа с клавиатуры по нажатию клавиши Enter или с пульта от AndroidTV по нажатию на кнопку OK будет отправлено направление `FocusDirection.Enter`, которое по умолчанию заменяется на направление вправо `FocusDirection.Right`.
	
4. **Выбор начальной точки для поиска кандидата на фокус:**
   В зависимости от направления Compose определяет начальную точку поиска:
	```kotlin
	val initialFocusRect = when (requestedDirection) {  
	    Right, Down -> focusRect().topLeft()  
	    Left, Up -> focusRect().bottomRight()  
	    else -> error(InvalidFocusDirection)  
	}
	```
	- **Для направления вправо или вниз**: стартовая точка — верхний левый угол.
	- **Для направления влево или вверх**: стартовая точка — нижний правый угол.
    
5. **Поиск лучшего кандидата:** 
   Метод `findBestCandidate` ищет оптимальный элемент для фокусировки.
    ```kotlin
    val nextCandidate = focusableChildren.findBestCandidate(initialFocusRect, requestedDirection)
	return nextCandidate?.let { onFound.invoke(it) } ?: false
    ```
#### Как работает `findBestCandidate`:

```kotlin
internal fun MutableVector<FocusTargetNode>.findBestCandidate(
    focusRect: Rect,
    direction: FocusDirection
): FocusTargetNode? {  
    var searchResult: FocusTargetNode? = null  

    var bestCandidate = when (direction) {  
        Left -> focusRect.translate(focusRect.width + 1, 0f)  
        Right -> focusRect.translate(-(focusRect.width + 1), 0f)  
        Up -> focusRect.translate(0f, focusRect.height + 1)  
        Down -> focusRect.translate(0f, -(focusRect.height + 1))  
        else -> error(InvalidFocusDirection)  
    }
	
    forEach { candidateNode ->  
        if (candidateNode.isEligibleForFocusSearch) {  
            val candidateRect = candidateNode.focusRect()  
            if (isBetterCandidate(candidateRect, bestCandidate, focusRect, direction)) {  
                bestCandidate = candidateRect  
                searchResult = candidateNode  
            }  
        }  
    }  
  
    return searchResult  
}
```
#### Этапы поиска лучшего кандидата:

1. **Инициализация стартового прямоугольника:**  
    В зависимости от направления создаётся прямоугольник, гарантированно выходящий за текущий фокус, чтобы можно было найти ближайшего кандидата.
    
2. **Проход по дочерним элементам:**  
    Для каждого фокусируемого потомка проверяется его eligibility (может ли он принять фокус).
    
3. **Сравнение кандидатов:**  
    Функция `isBetterCandidate` определяет, является ли текущий элемент более подходящим для фокуса, чем предыдущий лучший кандидат.

#### Как `isBetterCandidate` выбирает лучшего кандидата:

```kotlin
private fun isBetterCandidate(  
    proposedCandidate: Rect,  
    currentCandidate: Rect,  
    focusedRect: Rect,  
    direction: FocusDirection  
): Boolean {  
  
    fun Rect.isCandidate() = when (direction) {  
        Left -> (focusedRect.right > right || focusedRect.left >= right) && focusedRect.left > left  
        Right -> (focusedRect.left < left || focusedRect.right <= left) && focusedRect.right < right  
        Up -> (focusedRect.bottom > bottom || focusedRect.top >= bottom) && focusedRect.top > top  
        Down -> (focusedRect.top < top || focusedRect.bottom <= top) && focusedRect.bottom < bottom  
        else -> error(InvalidFocusDirection)  
    }  
  
    fun Rect.majorAxisDistance(): Float {  
        val majorAxisDistance = when (direction) {  
            Left -> focusedRect.left - right  
            Right -> left - focusedRect.right  
            Up -> focusedRect.top - bottom  
            Down -> top - focusedRect.bottom  
            else -> error(InvalidFocusDirection)  
        }  
        return max(0.0f, majorAxisDistance)  
    }  
  
    fun Rect.minorAxisDistance() = when (direction) {    
        Left, Right -> (focusedRect.top + focusedRect.height / 2) - (top + height / 2)    
        Up, Down -> (focusedRect.left + focusedRect.width / 2) - (left + width / 2)  
        else -> error(InvalidFocusDirection)  
    }  
  
    fun weightedDistance(candidate: Rect): Long {  
        val majorAxisDistance = candidate.majorAxisDistance().absoluteValue.toLong()  
        val minorAxisDistance = candidate.minorAxisDistance().absoluteValue.toLong()  
        return 13 * majorAxisDistance * majorAxisDistance + minorAxisDistance * minorAxisDistance  
    }  
  
    return when {    
        !proposedCandidate.isCandidate() -> false
           
        !currentCandidate.isCandidate() -> true    
        
        beamBeats(focusedRect, proposedCandidate, currentCandidate, direction) -> true  
        
        beamBeats(focusedRect, currentCandidate, proposedCandidate, direction) -> false  
        else -> weightedDistance(proposedCandidate) < weightedDistance(currentCandidate)  
    }  
}
```

Функция `isBetterCandidate` в Compose решает, является ли предложенный элемент более подходящим для фокуса по сравнению с текущим кандидатом. Она работает так:

1. **Проверка направления:** Элемент должен находиться хотя бы частично в нужном направлении (вправо, влево, вверх или вниз).
2. **Оценка расстояний:** Вычисляются:
    - **Основное расстояние** — по направлению фокуса (например, от правого края текущего элемента до левого края кандидата).
    - **Второстепенное расстояние** — между центрами элементов по перпендикулярной оси.
3. **Фокус по "лучу" (beam):** Если один из элементов частично перекрывает фокус в нужном направлении, он выигрывает сразу.
4. **Сравнение расстояний:** Если перекрытия нет, выбирается элемент с наименьшей взвешенной дистанцией (основное расстояние сильнее влияет на выбор).
   
#### Итог: полный цикл поиска фокуса в Compose

Когда вызывается `requestFocus`, Compose проходит такой путь:

1. **Вызов `requestFocus`:**  
    Процесс фокусировки начинается, когда вызывается `requestFocus` на какой-то ноде.
2. **Проверка текущей ноды:**
    - Если нода фокусируемая и доступная, она получает фокус сразу.
    - Если нет — запускается поиск среди дочерних элементов.
3. **Поиск дочернего кандидата (`findChildCorrespondingToFocusEnter`):**
    - Собираются все дочерние ноды, доступные для фокусировки, с помощью `collectAccessibleChildren`.
    - Если кандидат один — он сразу получает фокус.
    - Если несколько — продолжаем искать лучшего кандидата.
4. **Определение начальной точки:**  
    Выбирается точка фокусировки:
    - Для направления `Right` или `Down` — верхний левый угол.
    - Для направления `Left` или `Up` — нижний правый угол.
5. **Поиск лучшего кандидата (`findBestCandidate`):**  
    Для каждой фокусируемой ноды проверяется, подходит ли она для фокуса:
    - Нода должна находиться в нужном направлении относительно текущего фокуса.
    - Если таких кандидатов несколько, выбирается наиболее подходящая нода.
6. **Оценка кандидатов (`isBetterCandidate`):**
    - **Основное расстояние:** Проверка, насколько близко нода по направлению движения.
    - **Второстепенное расстояние:** Проверка смещения по перпендикулярной оси.
    - **Пересечение фокусных областей:** Если кандидаты пересекаются, приоритет у пересекающейся ноды.
    - **Взвешенное расстояние:** Если пересечений нет, побеждает ближайший элемент по специальной формуле.
7. **Передача фокуса:**
    - Если найден лучший кандидат — он получает фокус.
    - Если кандидатов нет или они недоступны — фокус остаётся на месте или теряется.


### Установка фокуса на элементе

Функция `requestFocus` отвечает за установку фокуса на элемент, учитывая направление (`FocusDirection`). Она запускает транзакцию через `FocusTransactionManager`, чтобы обеспечить атомарность операций с фокусом.

```kotlin
internal fun FocusTargetNode.requestFocus(focusDirection: FocusDirection): Boolean? {  
    return requireTransactionManager().withNewTransaction(  
        onCancelled = { if (node.isAttached) refreshFocusEventNodes() }  
    ) {  
        when (performCustomRequestFocus(focusDirection)) {  
            None -> performRequestFocus()  
            Redirected -> true  
            Cancelled, RedirectCancelled -> null  
        }  
    }  
}
```

🔸 **Основная логика:**

1. **Создание транзакции:** Через `requireTransactionManager().withNewTransaction`. Если начнется новый запрос фокуса, предыдущие транзакции отменятся.
2. **Обработка результата кастомного запроса фокуса (`performCustomRequestFocus`):**
    - `None` → кастомный фокус не настроен или не сработал → переход к стандартному `performRequestFocus`.
    - `Redirected` → фокус перенаправлен на другую ноду.
    - `Cancelled` или `RedirectCancelled` → фокус отменён.
3. **Обновление фокуса при отмене:** Если транзакция отменяется, вызывается `refreshFocusEventNodes`.


### Состояния фокуса в Compose

Состояние фокуса определяет, как элемент и его родители будут реагировать на запросы фокуса:
- **Active:** элемент активен и получает события ввода (например, нажатия клавиш).
- **ActiveParent:** элемент сам не в фокусе, но один из его дочерних элементов имеет фокус.
- **Captured:** элемент активен, но блокирует передачу фокуса (например, текстовое поле с ошибкой ввода).
- **Inactive:** элемент и его дочерние элементы не получают событий ввода.

##### Как это работает на практике

Рассмотрим простой пример с кнопками, чтобы наглядно увидеть, как меняются состояния фокуса.

В контейнере `Row` находятся две кнопки. Когда одна из них получает фокус (`Active`), её родитель (`Row`) переходит в состояние `ActiveParent`. Если мы вручную захватываем фокус с помощью `focusRequester.captureFocus()`, кнопка становится `Captured`, и фокус остаётся на ней до тех пор, пока мы явно не освободим его вызовом `focusRequester.freeFocus()`.

На примере ниже можно увидеть, как при перемещении между кнопками меняются их состояния фокуса:

![[focus_states_example.gif]]

Разобравшись в этих состояниях, будет проще понять, как работает механизм запроса фокуса в Compose.
#### Кастомная фокусировка: `performCustomRequestFocus`

Если у элемента задан кастомный `Modifier.focusProperties`, запрос фокуса обрабатывается по переопределенной логике. Эта функция проверяет текущее состояние элемента, какие параметры focusProperties переопределены и выполняет запрос фокуса, в соответствии с этим.

Рассмотрим её работу более детально:
```kotlin
internal fun FocusTargetNode.performCustomRequestFocus(  
    focusDirection: FocusDirection  
): CustomDestinationResult {  
    when (focusState) {  
        Active, Captured -> return None  
        ActiveParent ->  
            return requireActiveChild().performCustomClearFocus(focusDirection)  
        Inactive -> {  
            val focusParent = nearestAncestor(FocusTarget) ?: return None  
            return when (focusParent.focusState) {  
                Captured -> Cancelled  
                ActiveParent -> focusParent.performCustomRequestFocus(focusDirection)  
                Active -> focusParent.performCustomEnter(focusDirection)  
                Inactive ->  
                    focusParent.performCustomRequestFocus(focusDirection).takeUnless { it == None }  
                        ?: focusParent.performCustomEnter(focusDirection)  
            }  
        }  
    }  
}
```
🔸 Функция проходит по состояниям фокуса и решает, как обрабатывать запрос:

1. **Фокус уже установлен (**`**Active**`**,** `**Captured**`**)**
    - Если элемент уже в фокусе или захватил его (`Captured`), функция возвращает `None`. Это приведёт к вызову стандартной `performRequestFocus`, которая оставит фокус на месте.
2. **Элемент — родитель с фокусом у дочернего элемента (**`**ActiveParent**`**)**
    - Текущий элемент очищает фокус у активного ребёнка через `performCustomClearFocus` и вызывает `exit` из `focusProperties`, если он переопределён.
    - Если дочерний элемент успешно перенаправил фокус, работа функции завершается. В противном случае родитель пытается получить фокус через `performRequestFocus`.
3. **Элемент неактивен (**`**Inactive**`**)**
    - Ищется ближайший родитель с `FocusTarget`. Если он не найден, возвращается `None`.
    - Если родитель найден, анализируется его текущее состояние:
        - **Captured** → Родитель не хочет отдавать фокус (например, текстовое поле с валидацией) → возвращается `Cancelled`.
        - **ActiveParent** → Запрос фокуса выполняется через родителя, если у него переопределён `focusProperties`.
        - **Active** → Если родитель уже в фокусе, проверяется наличие `focusProperties.enter`. Если есть, вызывается `performCustomEnter`.
        - **Inactive** → Если родитель тоже неактивен, запрос уходит выше по дереву, пока не найдётся активный элемент с кастомной логикой входа (`focusProperties.enter`). Если никто не может принять фокус, возвращается `None`.

#### Дефолтная фокусировка: `performRequestFocus`

Если кастомной логики нет или она не сработала, вызывается стандартная функция захвата фокуса:
```kotlin
internal fun FocusTargetNode.performRequestFocus(): Boolean {  
   val success = when (focusState) {  
        Active, Captured -> true  
        ActiveParent -> clearChildFocus() && grantFocus()  
        Inactive -> {  
            val parent = nearestAncestor(FocusTarget)  
            if (parent != null) {  
                val prevState = parent.focusState  
                val success = parent.requestFocusForChild(this)  
                if (success && prevState !== parent.focusState) {  
                    parent.refreshFocusEventNodes()  
                }  
                success  
            } else {  
                requestFocusForOwner() && grantFocus()  
            }  
        }  
    }  
    if (success) refreshFocusEventNodes()  
    return success  
}
```
🔸 **Разбор логики:**

1. **Элемент уже активен или захвачен (`Active`, `Captured`)**
    - Если фокус уже установлен, то ничего не происходит.
2. **Элемент — активный родитель (`ActiveParent`)**
    - Очищается фокус у дочернего элемента (`clearChildFocus`).
    - Текущий элемент берет фокус через `grantFocus`.
3. **Элемент неактивен (`Inactive`)**
    - Ищется ближайший родитель с фокусом.
    - Если родитель найден:
        - **Вызывается `requestFocusForChild`:** Родитель передает фокус текущему элементу.
        - Если состояние родителя изменилось, обновляются состояния фокусов через `refreshFocusEventNodes`.
    - Если родитель не найден:
        - Фокус запрашивается у владельца Compose View через `requestFocusForOwner`.
        - Если владелец принял фокус, он закрепляется вызовом `grantFocus`.
4. **Обновление событий фокуса**
    - После успешного запроса фокуса вызывается `refreshFocusEventNodes`, чтобы система Compose обновила состояния фокусных нод.

#### Итог: как работает запрос фокуса в Jetpack Compose

1. Вызывается `requestFocus`.
2. Если у элемента есть кастомная логика (`focusProperties`), она обрабатывается в `performCustomRequestFocus`.
3. Если кастомный фокус не сработал, вызывается стандартный `performRequestFocus`.
4. Если элемент не может принять фокус, запрос уходит к родителю (`nearestAncestor(FocusTarget)`).
5. Если родитель тоже не может принять фокус, запрос передается дальше вверх по дереву.
6. В крайнем случае, фокус запрашивается у владельца Compose View через `requestFocusForOwner`.
7. После успешного получения фокуса вызывается `refreshFocusEventNodes` для обновления событий фокуса.

### Обновление работы фокуса на Compose версии 1.8

В обновлённой версии **Compose 1.8** изменилось API класса `FocusRequester`, и теперь функция `requestFocus` выглядит так:

```kotlin
fun requestFocus(focusDirection: FocusDirection = Enter): Boolean = 
	findFocusTargetNode {  
	    it.requestFocus(focusDirection)  
	}
```

Главное изменение — добавление параметра `FocusDirection`, который делает запрос фокуса более гибким.

Также была изменена логика работы метода `performRequestFocus()`, и теперь используется его оптимизированная версия:
```kotlin
private fun FocusTargetNode.performRequestFocusOptimized(): Boolean {  
    val focusOwner = requireOwner().focusOwner  
    val previousActiveNode = focusOwner.activeFocusTargetNode  
    val previousFocusState = focusState  
    if (previousActiveNode === this) {   
        dispatchFocusCallbacks(previousFocusState, previousFocusState)  
        return true  
    }  
  
    if (previousActiveNode?.clearFocus(refreshFocusEvents = true) == false) {  
        return false
    }  
   
    if (previousActiveNode == null && !requestFocusForOwner()) {  
        return false
    }  
    grantFocus()  
   
    var previousAncestorTargetNodes: MutableVector<FocusTargetNode>? = null
      
    if (previousActiveNode != null) {  
        previousAncestorTargetNodes = mutableVectorOf()  
        previousActiveNode.visitAncestors(Nodes.FocusTarget) { previousAncestorTargetNodes.add(it) }  
    } 
  
    val ancestorTargetNodes = mutableVectorOf<FocusTargetNode>()  
    visitAncestors(Nodes.FocusTarget) {  
        val removed = previousAncestorTargetNodes?.remove(it)  
        if (removed == null || !removed) {  
            ancestorTargetNodes.add(it)  
        }  
    }  
  
    previousAncestorTargetNodes?.forEachReversed {  
        if (focusOwner.activeFocusTargetNode !== this) {  
            return false  
        }  
        it.dispatchFocusCallbacks(ActiveParent, Inactive)  
    }  
  
    
    ancestorTargetNodes.forEachReversed {  
        if (focusOwner.activeFocusTargetNode !== this) {  
            return false  
        }  
        it.dispatchFocusCallbacks(Inactive, ActiveParent)  
    }  
  
    if (focusOwner.activeFocusTargetNode !== this) {  
        return false  
    }  
  
    dispatchFocusCallbacks(previousFocusState, Active)  
  
    if (focusOwner.activeFocusTargetNode !== this) {  
        return false  
    }  
  
    @OptIn(ExperimentalComposeUiApi::class, InternalComposeUiApi::class)  
    if (ComposeUiFlags.isViewFocusFixEnabled && requireLayoutNode().getInteropView() == null) {   
        requireOwner().focusOwner.requestFocusForOwner(FocusDirection.Next, null)  
    }  
  
    return true  
}
```
🔸 **Разбор логики:

1. **Получение текущего состояния фокуса**
    - Запрашивается `focusOwner`, который управляет фокусом внутри Compose.
    - Определяется текущая активная нода (`previousActiveNode`) и предыдущее состояние фокуса (`previousFocusState`).
2. **Обработка повторного запроса фокуса**
    - Если текущая нода (`this`) уже находится в фокусе (`previousActiveNode === this`),  
        то просто повторно отправляются события фокуса (`dispatchFocusCallbacks`), и функция завершает работу.
3. **Очистка фокуса у предыдущего активного элемента**
    - Если у предыдущей ноды (`previousActiveNode`) не получается сбросить фокус (`clearFocus(refreshFocusEvents = true) == false`), то новый фокус не устанавливается, и функция завершает работу.
4. **Запрос фокуса у владельца (`focusOwner`)**
    - Если ранее не было сфокусированного элемента (`previousActiveNode == null`), то вызывается `requestFocusForOwner()`, чтобы запросить фокус у ComposeView.
    - Если этот запрос не удался, то новый фокус также не назначается.
5. **Предоставление фокуса текущему элементу (`grantFocus`)**
    - Если все предыдущие шаги прошли успешно, вызывается `grantFocus()`, который присваивает состояние фокуса Active текущей ноде.
6. **Определение и сравнение списков родительских узлов**
    - Определяются все предки предыдущего и нового активного элемента, которые могли иметь состояние фокуса.
    - Для каждого предка проверяется, остался ли он активным (`ActiveParent`) или стал неактивным.
7. **Обновление состояний предков**
    - Узлы, которые потеряли статус `ActiveParent`, переводятся в `Inactive`, что означает что у них больше нет дочернего элемента, который имеет фокус.
    - Новые активные предки переводятся из `Inactive` в `ActiveParent`, что означает что у них появился дочерний элемент, который имеет фокус.
8. **Проверка успешности захвата фокуса**
    - Если в процессе вызовов `dispatchFocusCallbacks` фокус был сброшен или перенаправлен на другой элемент, функция прерывается.
9. **Отправка событий фокусировки новому активному узлу**
    - Вызывается `dispatchFocusCallbacks(previousFocusState, Active)`, который уведомляет систему о смене состояния фокуса у текущей ноды.
10. **Дополнительная проверка для элементов без `AndroidView`**
    - Если нода **не** является `AndroidView`, но Compose-флаг `isViewFocusFixEnabled` активен (который сейчас по дефолту включен), то вызывается `requestFocusForOwner(FocusDirection.Next, null)`, чтобы передать фокус в ComposeView.
11. **Возвращение успешного результата**
    - Если все этапы прошли успешно, функция возвращает `true`, указывая, что элемент успешно получил фокус.

#### Что изменилось?

- **Оптимизация прохода по дереву фокуса (diff-логика)** — несомненно **главное улучшение** в оптимизации обновления состояния фокусов.
- **Исправление бага фокусов при взаимодействии с View** (баг появился на версии Compose 1.7).

В итоге, это обновление является важным, так как значительно повысило надежность работы с фокусами, существенно уменьшив количество крашей, связанных с ними, на проекте PREMIER. Оно также позволило разработать более гибкую логику перемещения фокусов и улучшить производительность приложений, активно использующих эту функциональность.


## Заключение

Понимание того, как работает запрос фокуса в Jetpack Compose, важно не только для создания удобного UI, но и для решения сложных задач, связанных с доступностью, навигацией на TV-устройствах и кастомными сценариями ввода. Разобравшись в механизме поиска фокусируемых элементов, вы сможете точно контролировать, какой элемент должен получать фокус и когда это должно происходить.

С выходом Compose 1.8 этот процесс стал ещё гибче за счёт возможности передавать `FocusDirection` в `requestFocus()`, что позволяет лучше управлять направленной фокусировкой.

Если вам интересны подобные технические разборы Jetpack Compose, подписывайтесь на мой Telegram-канал (https://t.me/mobile_type), где я регулярно публикую статьи и полезные материалы по Android-разработке!
