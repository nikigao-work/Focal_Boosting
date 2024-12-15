# Focal_Boosting
Classic gradient boosting with a twist


# Бустинг с фокальным лоссом для задачи классификации

## Описание фокального лосса

Фокальный лосс (Focal Loss) помогает модели сосредотачиваться на трудно различимых примерах за счёт модификации стандартной кросс-энтропийной функции потерь. Он снижает влияние легко классифицируемых объектов и увеличивает вклад сложных примеров.

Функция фокального лосса для классификации:

$$
FL(p_t) = -(1 - p_t)^\gamma \cdot \log(p_t),
$$


где:

- `p_t` — вероятность правильного класса, вычисленная как `softmax(z)_t`, где `z` — логиты модели,
- `gamma > 0` — фокусирующий параметр, который управляет вкладом сложных примеров.

---

## Градиент фокального лосса

Для обучения деревьев бустинга необходимо вычислить градиент фокального лосса по логитам `z_k`. Для мультиклассовой задачи функция потерь записывается как:

**FL_i = - Σ (y_i,k * (1 - p_i,k)^gamma * log(p_i,k)),**

где:

- `y_i,k` — one-hot представление истинного класса объекта `i`,
- `p_i,k` — вероятность класса `k`, вычисленная как `softmax(z_i,k)`.

Градиент по логитам `z_k` вычисляется по формуле:

**dFL_i/dz_i,k = (y_i,k - p_i,k) * gamma * (1 - p_i,k)^(gamma - 1) * p_i,k.**

### Компоненты градиента:

1. `(y_i,k - p_i,k)` — ошибка (residual) между истинной меткой и предсказанной вероятностью.
2. `(1 - p_i,k)^(gamma - 1)` — фокусирующий фактор, усиливающий вклад сложных примеров.
3. `p_i,k` — корректирующий множитель, учитывающий вероятность текущего класса.

1. \((y_{i,k} - \hat{p}_{i,k})\) — ошибка (residual) между истинной меткой и предсказанной вероятностью.
2. \((1 - \hat{p}_{i,k})^{\gamma - 1}\) — фокусирующий фактор, усиливающий вклад сложных примеров.
3. \(\hat{p}_{i,k}\) — корректирующий множитель, учитывающий вероятность текущего класса.


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

- `residuals = one_hot_y - probs` соответствует `(y_i,k - p_i,k)`, то есть разнице между истинным значением метки и предсказанной вероятностью для каждого класса.
- `(1 - probs) ** (self.gamma - 1)` реализует фокусирующий фактор `(1 - p_i,k)^(gamma - 1)`, который увеличивает влияние сложных примеров (когда модель ошибается).
- `probs` — это предсказанные вероятности для каждого класса, что соответствует \(p_i,k\).

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
   Упрощённая формула сохраняет ключевые компоненты исходного градиента, такие как `gamma`, остатки (`residuals`) и фокусирующий множитель `(1 - probs)^(gamma - 1)`, но опускает некоторые дополнительные элементы (например, `log(probs)`). Это позволяет минимизировать вычислительные затраты.

2. **Оптимизация скорости обучения**  
   Для каждой итерации бустинга градиенты используются в качестве целевых значений для обучения деревьев решений. Упрощённая формула быстрее вычисляется, что особенно важно при больших объёмах данных.

3. **Использование деревьев решений**  
   Деревья решений в бустинге обучаются на линейных отклонениях (`residuals`), а не на точных значениях функции потерь. Упрощённый градиент учитывает разницу между истинными метками и предсказаниями, что достаточно для корректной работы деревьев.

4. **Снижение сложности многоклассовой задачи**  
   В задачах с несколькими классами вычисление полного градиента для каждого класса увеличивает вычислительную сложность. Упрощение градиента позволяет снизить эту сложность без значительной потери качества.

5. **Эффективность для сложных примеров**  
   Ключевая цель фокального лосса — усиление влияния сложных примеров — достигается за счёт фокусирующего множителя `(1 - probs)^(gamma - 1)`. Эта часть формулы сохранена в упрощённой версии.
