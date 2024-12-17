# Бустинг с фокальным лоссом для задачи регрессии

## Описание фокального лосса

Фокальный лосс помогает модели сосредотачиваться на трудно различимых примерах за счёт модификации стандартной функции потерь. В случае регрессии он снижает влияние малых ошибок и усиливает вклад больших ошибок.

Функция фокального лосса для регрессии:

$$
L(r) = |r|^\gamma \cdot \log(|r| + 1),
$$

где:

- $$r = y - \hat{y}$$ — остаток (разница между истинным значением и предсказанием);
- $$\gamma > 0$$ — параметр, управляющий вкладом больших ошибок;
- $$|r|^\gamma $$ усиливает влияние значительных ошибок;
- $$log(|r| + 1)$$ ограничивает рост функции потерь для больших значений \( r \), сглаживая влияние выбросов.

---

## Производная фокального лосса

Для обучения модели (например, деревьев решений в бустинге) необходимо вычислить градиент фокального лосса по остаткам \( r \).

Используем правило производной для произведения двух функций:

$$
L(r) = f(r) \cdot g(r),
$$

где:

- $$f(r) = |r|^\gamma$$,
- $$g(r) = \log(|r| + 1)$$.

Производная по \( r \):

$$
\frac{\partial L(r)}{\partial r} = \frac{\partial f(r)}{\partial r} \cdot g(r) + f(r) \cdot \frac{\partial g(r)}{\partial r}.
$$


### Шаг 1: Производная $$f(r) = |r|^\gamma$$

Производная $$( |r|^\gamma \)$$ по \( r \):

$$
\frac{\partial f(r)}{\partial r} = \gamma |r|^{\gamma - 1} \cdot \text{sign}(r),
$$

где $$\text{sign}(r) \$$ — знак остатка:

$$
\text{sign}(r) = \begin{cases} 
1, & r > 0, \\
0, & r = 0, \\
-1, & r < 0.
\end{cases}
$$

### Шаг 2: Производная $$g(r) = \log(|r| + 1)$$

Производная $$\( \log(|r| + 1) \)$$ по \( r \):

$$
\frac{\partial g(r)}{\partial r} = \frac{1}{|r| + 1} \cdot \text{sign}(r).
$$

### Шаг 3: Подставляем в правило произведения

Теперь объединяем результаты для вычисления градиента:

$$
\frac{\partial L(r)}{\partial r} = \gamma |r|^{\gamma - 1} \cdot \text{sign}(r) \cdot \log(|r| + 1) + \frac{|r|^\gamma}{|r| + 1}.
$$

---

## Итоговая формула градиента

Градиент фокального лосса для регрессии принимает следующий вид:

$$
\boxed{g(r) = \gamma |r|^{\gamma - 1} \cdot \text{sign}(r) \cdot \log(|r| + 1) + \frac{|r|^\gamma}{|r| + 1}}
$$

## Компоненты градиента

1. $$\gamma |r|^{\gamma - 1} \cdot \text{sign}(r) \cdot \log(|r| + 1)$$ — усиливает вклад больших ошибок и масштабирует их с учётом логарифма.
2. $$\frac{|r|^\gamma}{|r| + 1}$$ — сглаживает влияние выбросов, ограничивая рост функции потерь.

---

## Реализация в коде

В коде градиент можно вычислить следующим образом:

```python
import numpy as np

def grad(residuals, gamma):
    abs_r = np.abs(residuals)
    return (gamma * abs_r ** (gamma - 1) * np.sign(residuals) * np.log(abs_r + 1) +
            (abs_r ** gamma) / (abs_r + 1))
```

Здесь:
- `residuals` — вектор остатков (ошибок);
- `gamma` — параметр фокального лосса;
- Комбинация логарифмического и степенного слагаемого учитывает как масштаб, так и сглаживание ошибок.


# Бустинг с фокальным лоссом для задачи классификации

## Описание фокального лосса

Фокальный лосс (Focal Loss) помогает модели сосредотачиваться на трудно различимых примерах за счёт модификации стандартной кросс-энтропийной функции потерь. Он снижает влияние легко классифицируемых объектов и увеличивает вклад сложных примеров.

Функция фокального лосса для классификации:

$$
FL(p_t) = -(1 - p_t)^\gamma \cdot \log(p_t),
$$


где:

- $$p_t$$ — вероятность правильного класса, вычисленная как $$softmax(z)_t$$, где $$z$$ — логиты модели,
- $$\gamma > 0$$ — фокусирующий параметр, который управляет вкладом сложных примеров.


- $$(1 - p_t)^\gamma$$ — фокусирующий модификатор.
- $$log(p_t)$$ — стандартный логарифмический штраф, используемый в кросс-энтропии.

### Основные элементы формулы

1. **Классический логарифмический штраф**:
   - Это базовый компонент фокального лосса, взятый из стандартной кросс-энтропии. Он штрафует модель за отклонение от истинного распределения.
   - Чем дальше вероятность от 1 (идеального предсказания), тем больше штраф.
   - Этот компонент помогает оптимизировать модель, минимизируя разницу между предсказаниями и реальными метками.

2. **Фокусирующий модификатор**:
   - **Цель**: уменьшить вклад легко классифицируемых примеров, где $$p_t$$ близко к 1, и увеличить влияние трудных примеров, где $$p_t$$ мало.
   - Если пример легко классифицируется $$p_t -> 1$$, то $$(1 - p_t)^\gamma \to 0$$, и его вклад в градиент будет минимальным.
   - Если пример трудный $$(p_t -> 0)$$, то $$(1 - p_t)^\gamma \to 1$$, и он получит больший вес в оптимизации.

3. **Параметр gamma**:
   - Фокусирующий парамет регулирует степень уменьшения вклада лёгких примеров.
   - При $$\gamma = 0$$ модификатор отсутствует, и фокальный лосс превращается в обычную кросс-энтропию.
   - С увеличением gamma, модель всё больше сосредотачивается на сложных примерах.

### Градиент фокального лосса

Для вывода градиента по логитам $$z_k$$, мы дифференцируем FL $$p_t$$:

$$
\frac{\partial FL}{\partial z_k} = \frac{\partial FL}{\partial p_k} \cdot \frac{\partial p_k}{\partial z_k}
$$

1. Производная по $$p_k$$:
Вероятность для класса $$t$$ определяется как:

$$
p_t = \frac{\exp(z_k)}{\sum_k \exp(z_k)}
$$


Где $$z_k$$ — логит для класса $$k$$, а $$sum_k \exp(z_k)$$ — нормировка по всем классам.

Фокальный лосс состоит из двух частей:

$$- (1 - p_t)^{\gamma} \log(p_t)$$ — это основной компонент, который штрафует за ошибки.
Производная по pₖ даёт комбинированный штраф, который зависит от параметра $$\gamma, (1 - p_t), p_t$$

2. Производная по $$z_k$$:
Для вычисления градиента по логитам zₖ используется производная softmax, которая даёт корректировку для вероятностей:

$$
\frac{\partial p_k}{\partial z_k} = p_k \cdot (1 - p_k)
$$


Где $$p_k$$ — вероятность для класса $$k$$.

Итоговый градиент фокального лосса:
Итоговый градиент для обновления логита zₖ выглядит так:

$$
\frac{\partial FL}{\partial z_k} = (y_k - p_k) \cdot \gamma \cdot (1 - p_k)^{\gamma - 1} \cdot p_k
$$





### Почему выбрана именно такая структура?

1. **Сохраняется физический смысл**:
    - $$(y_k - \hat{p}_k)$$: Ошибка между истинным распределением и предсказанным.
    - $$(1 - \hat{p}_k)^{\gamma - 1} \cdot \hat{p}_k $$: Усиливает вклад трудных примеров (когда p̂ₖ мало) и уменьшает влияние лёгких примеров (когда p̂ₖ близко к 1).


3. **Стабильная оптимизация**:
   - Softmax гарантирует, что все вероятности суммируются в 1.
   - Фокусирующий параметр $$\gamma\$$ регулирует сложность обучения, не требуя жёстких гиперпараметров.

4. **Простота реализации**:
   - Легко интегрируется в бустинг.
   - В градиенте выделены ошибки, модификатор и корректирующий фактор.

---

## Градиент фокального лосса

Для обучения деревьев бустинга необходимо вычислить градиент фокального лосса по логитам $$z_k$$. Для мультиклассовой задачи функция потерь записывается как:

$$FL_i = - Σ (y_i,k * (1 - p_i,k)^\gamma * log(p_i,k)),$$

где:

- $$y_i,k$$ — one-hot представление истинного класса объекта $$i$$,
- $$p_i,k$$ — вероятность класса `k`, вычисленная как $$softmax(z_i,k)$$.

Градиент по логитам $$z_k$$ вычисляется по формуле:

$$dFL_i/dz_i,k = (y_i,k - p_i,k) * gamma * (1 - p_i,k)^{\gamma - 1} * p_i,k.$$

### Компоненты градиента:

1. $$(y_i,k - p_i,k)$$ — ошибка (residual) между истинной меткой и предсказанной вероятностью.
2. $$(1 - p_i,k)^(\gamma - 1)$$ — фокусирующий фактор, усиливающий вклад сложных примеров.
3. $$p_i,k$$ — корректирующий множитель, учитывающий вероятность текущего класса.

---

Полная формула фокального лосса для задачи классификации с учётом параметра $\alpha$ выглядит так:

$$
FL(p_t) = -\alpha_t (1 - p_t)^\gamma \log(p_t),
$$

где:
- $$\alpha_t$$ — вес для класса $t$, который компенсирует дисбаланс классов.
- Остальные элементы $$(1 - p_t)^\gamma$ и $\log(p_t)$$ остаются без изменений.

### Зачем нужен $\alpha$?

1. **Дисбаланс классов**:  
   В задачах, где один класс сильно преобладает над другими, модель может игнорировать редкие классы, поскольку оптимизация фокусируется на "главном" классе.  
   $\alpha_t$ вводится, чтобы назначить больший вес редким классам, увеличивая их вклад в обучение.

2. **Гибкость контроля**:  
   $\alpha_t$ позволяет вручную регулировать значимость каждого класса. Например:
   - Для сильно несбалансированных данных: $\alpha_t$ задаётся пропорционально обратной частоте класса.
   - Для сбалансированных данных: $\alpha_t = 1$ для всех классов.


Градиент с учётом $\alpha$

С добавлением $\alpha_t$ градиент по логитам $z_k$ (для класса $k$) становится:

$$frac{\partial FL}{\partial z_k} = \alpha_i \cdot (y_k - p_k) \cdot \gamma \cdot (1 - p_k)^{\gamma - 1} \cdot p_k$$

Здесь $\alpha_t$ непосредственно масштабирует вклад градиента для каждого класса.


### Почему $\alpha$ нет в коде?

Код не включает параметр $\alpha$, потому что он рассчитан на задачи, где дисбаланс классов не является ключевой проблемой. В таких случаях:
- Все классы считаются равнозначными.
- $\alpha_t = 1$ для всех $t$, и $\alpha$ фактически выпадает из формулы.


---

## Реализация в коде

В коде градиент фокального лосса вычисляется следующим образом:

```python
# Вычисление ошибок (residuals)
residuals = one_hot_y - probs

# Градиент с учётом фокального лосса
gradient = (self.gamma * residuals * (1 - probs) ** (self.gamma - 1)) * probs
```

Здесь:

- `residuals = one_hot_y - probs` соответствует $$(y_i,k - p_i,k)$$, то есть разнице между истинным значением метки и предсказанной вероятностью для каждого класса.
- $$(1 - probs) ** (self.gamma - 1)$$ реализует фокусирующий фактор $$(1 - p_i,k)^(gamma - 1)##, который увеличивает влияние сложных примеров (когда модель ошибается).
- `probs` — это предсказанные вероятности для каждого класса, что соответствует $$\(p_i,k\)$$.

---


## Особенности реализации

В данной реализации фокальный лосс интегрирован в алгоритм бустинга с использованием деревьев решений. В процессе обучения происходит несколько ключевых шагов:

1. **Softmax**: Вычисляются вероятности классов через softmax-функцию. Эти вероятности затем используются для оценки ошибки (residuals).
2. **Модификация ошибок**: Остатки (ошибки) между истинными и предсказанными значениями классов модифицируются с учётом фокального лосса, что позволяет усилить вклад сложных примеров.
3. **Градиенты как целевые значения**: Полученные градиенты по формулам фокального лосса передаются как целевые значения для обучения деревьев.

В итоге этот подход позволяет модели более эффективно справляться с трудными для классификации примерами и улучшать точность, особенно в условиях несбалансированных классов.

---


В коде используется следующая упрощённая формула градиента:

$$
\text{gradient} = \gamma \cdot (\text{residuals}) \cdot (1 - \text{probs})^{\gamma - 1} \cdot \text{probs},
$$


## Почему используется упрощённая формула?

1. **Упрощение вычислений**  
   Упрощённая формула сохраняет ключевые компоненты исходного градиента, такие как $$\gamma$$, остатки (`residuals`) и фокусирующий множитель $$(1 - probs)^(gamma - 1)$$, но опускает некоторые дополнительные элементы (например, `log(probs)`). Это позволяет минимизировать вычислительные затраты.

2. **Оптимизация скорости обучения**  
   Для каждой итерации бустинга градиенты используются в качестве целевых значений для обучения деревьев решений. Упрощённая формула быстрее вычисляется, что особенно важно при больших объёмах данных.

3. **Использование деревьев решений**  
   Деревья решений в бустинге обучаются на линейных отклонениях (`residuals`), а не на точных значениях функции потерь. Упрощённый градиент учитывает разницу между истинными метками и предсказаниями, что достаточно для корректной работы деревьев.

4. **Снижение сложности многоклассовой задачи**  
   В задачах с несколькими классами вычисление полного градиента для каждого класса увеличивает вычислительную сложность. Упрощение градиента позволяет снизить эту сложность без значительной потери качества.

5. **Эффективность для сложных примеров**  
   Ключевая цель фокального лосса — усиление влияния сложных примеров — достигается за счёт фокусирующего множителя $$(1 - probs)^(gamma - 1)$$. Эта часть формулы сохранена в упрощённой версии.
