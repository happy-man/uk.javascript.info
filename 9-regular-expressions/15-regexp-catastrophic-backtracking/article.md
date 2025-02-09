# Катастрофічний пошук з поверненням

Деякі регулярні вирази виглядають простими, але можуть виконуватись дууууууже тривалий час, та навіть змусити "зависнути" рушій JavaScript.

Рано чи пізно більшість розробників стикаються з подібною ситуацією. Типовий показник – часом регулярний вираз працює добре, але на конкретних рядках він "зависає", споживаючи 100% CPU.

В такому випадку браузер пропонує зупинити скрипт та перезавантажити сторінку. Не найкращий розвиток подій.

Навіть гірше, для серверного JavaScript подібний регулярний вираз може спричинити зависання серверного процесу. Тож варто звернути увагу на цей момент.

## Приклад

Скажімо, ми маємо рядок і хотіли б перевірити його на наявність слів `pattern:\w+` з необов’язковим пробілом `pattern:\s?` після кожного.

Очевидний спосіб сконструювати регулярний вираз – взяти слово, за яким йде необов’язковий пробіл `pattern:\w+\s?`, та прописати повтор `*`.

Це приведе нас до регулярного виразу `pattern:^(\w+\s?)*$`, який описує 0+ таких слів, починається з `pattern:^` та закінчується на `pattern:$`.

На прикладі:

```js run
let regexp = /^(\w+\s?)*$/;

alert( regexp.test("Хороший рядок") ); // true
alert( regexp.test("Погані символи: $@#") ); // false
```

Регулярний вираз наче працює. Результат коректний. Та все ж, певні рядки забирають багато часу. Достатньо, аби рушій JavaScript "зависнув", споживаючи 100% CPU.

Якщо виконати приклад нижче, можливо, ви нічого не побачите, бо JavaScript просто "зависне". Браузер перестане реагувати на події, UI перестане працювати (більшість браузерів лишають лише можливість прокрутки). За деякий час, вам запропонують перезавантажити сторінку. Тож будьте обережними:

```js run
let regexp = /^(\w+\s?)*$/;
let str = "Введений рядок, який довго оброблятиметься, або навіть спровокує зависання регулярного виразу!";

// займе дуже багато часу
alert( regexp.test(str) );
```

Варто відмітити, що деякі рушії регулярних виразів можуть ефективно провести подібний пошук. Наприклад, версія рушія V8 починаючи з 8.8 здатна на це (тобто Google Chrome 88 не зависне), а Firefox все ж матиме проблеми.

## Спрощений приклад

В чому ж справа? Чому регулярний вираз спричинює зависання?

Аби це зрозуміти, спростимо приклад: приберемо пробіли `pattern:\s?`, отримаємо `pattern:^(\w+)*$`.

Також, для більшої очевидності, замінимо `pattern:\w` на `pattern:\d`. Отриманий регулярний вираз все одно зависає:

```js run
let regexp = /^(\d+)*$/;

let str = "012345678901234567890123456789z";

// займе дуже багато часу (обережніше!)
alert( regexp.test(str) );
```

То що з ним не так?

По-перше, помітно, що регулярний вираз `pattern:(\d+)*` є трішки дивним. Квантифікатор `pattern:*` виглядає недоречним. Якщо нам потрібно число, можна використати `pattern:\d+`.

Дійсно, регулярний вираз штучний; ми отримали його після спрощення попереднього прикладу. Але причина повільної роботи лишається тією ж. Тож розберемося в ній, і тоді попередній приклад стане очевидним.

Що коїться під час пошуку `pattern:^(\d+)*$` в рядку `subject:123456789z`? Чому він займає так багато часу? Цей приклад трішки скорочений для ясності, зауважте нецифровий символ `subject:z` наприкінці – він важливий.

Ось що робить рушій регулярних виразів:

1. Для початку, рушій регулярного виразу намагається знайти вміст дужок: число `pattern:\d+`. За замовчуванням, режим `pattern:+` є жадібним, тому він поглинає всі цифри:

    ```
    \d+.......
    (123456789)z
    ```

    Після поглинання всіх цифр, `pattern:\d+` вважається знайденим (як `match:123456789`).

    Далі, застосовується квантифікатор `pattern:(\d+)*`. В тексті не залишилось цифр, тож зірочка нічого не дає.

    Наступний символ шаблону – кінець рядка `pattern:$`. Але в тексті маємо `subject:z`, тож збігу немає:

    ```
               X
    \d+........$
    (123456789)z
    ```

2. Без збігу, жадібний квантифікатор `pattern:+` зменшує кількість повторів та повертається на один символ назад.

    Тепер `pattern:\d+` приймає усі цифри, окрім останньої (`match:12345678`):
    ```
    \d+.......
    (12345678)9z
    ```
3. Потім рушій намагається продовжити пошук з наступної позиції (одразу після `match:12345678`).

    Можна застосувати `pattern:(\d+)*` -- це дасть ще один збіг для `pattern:\d+`, число `match:9`:

    ```

    \d+.......\d+
    (12345678)(9)z
    ```

    Рушій невдало намагається знову шукати збіг для `pattern:$`, натомість зустрічає `subject:z`:

    ```
                 X
    \d+.......\d+
    (12345678)(9)z
    ```


4. Збігу нема, тому рушій продовжить пошук з поверненням, зменшуючи кількість повторень. Зазвичай, це працює наступним чином: останній жадібний квантифікатор зменшує кількість повторень доти, доки не досягнутий мінімум. Потім спрацьовує попередній жадібний квантифікатор, і так далі.

    Ми спробували всі можливі комбінації. Далі – їх приклади.

    Перше число `pattern:\d+` має 7 цифр та двозначне число опісля:

    ```
                 X
    \d+......\d+
    (1234567)(89)z
    ```

    Перше число має 7 цифр та два одноцифрових числа опісля:

    ```
                   X
    \d+......\d+\d+
    (1234567)(8)(9)z
    ```

    Перше число має 6 цифр та трицифрове число за ними:

    ```
                 X
    \d+.......\d+
    (123456)(789)z
    ```

    Перше число має 6 цифр та два числа за ними:

    ```
                   X
    \d+.....\d+ \d+
    (123456)(78)(9)z
    ```

    ...І так далі.


Існує багато шляхів поділу послідовності цифр `123456789` на номери. Казати точно, усього <code>2<sup>n</sup>-1</code>, де `n` - довжина послідовності.

- Для `123456789`, маємо `n=9`, що дає 511 комбінацій.
- Довша послідовність `n=20` дасть приблизно мільйон (1 048 575) комбінацій.
- Для `n=30` - в тисячу разів більше (1 073 741 823 комбінацій).

Проходження кожною з них і є причиною повільного пошуку.

## Повертаючись до слів та рядків

Подібне відбувається в нашому першому прикладі, коли ми шукаємо слова за шаблоном `pattern:^(\w+\s?)*$` в рядку `subject:Рядок, що висне!`.

Причина в тому, що слово може бути представлене як один чи кілька `pattern:\w+`:

```
(input)
(inpu)(t)
(inp)(u)(t)
(in)(p)(ut)
...
```

З точки зору людини, збігу не може бути, бо рядок закінчується знаком `!`, але регулярний вираз в кінці очікує символ “слова” `pattern:\w` або пробіл `pattern:\s`. Але рушій цього не знає.

Він перебирає всі комбінації "поглинання" рядку регулярним виразом `pattern:(\w+\s?)*`, включаючи варіанти з пробілами `pattern:(\w+\s)*` та без `pattern:(\w+)*` (бо пробіли `pattern:\s?` необов’язкові). Пошук є тривалим через велику кількість комбінацій (як на прикладі цифр).

Що робити?

Чи варто включати лінивий режим?

Нажаль, це не допоможе: якщо замінити `pattern:\w+` на `pattern:\w+?`, регулярний вираз все одно висне. Зміниться порядок комбінацій, але не загальна кількість.

Деякі рушії регулярних виразів мають хитро побудовані тести та скінченні автомати, що дозволяють оминути проходження всіма комбінаціями або пришвидшити, але це стосується меншості рушіїв та не завжди допомагає.

## Як це виправити?

Існує два основних підходи до вирішення проблеми.

Перший – знизити кількість можливих комбінацій.

Зробимо пробіл обов’язковим, переписавши регулярний вираз як `pattern:^(\w+\s)*\w*$` - ми шукатимемо будь-яку кількість слів та пробіл опісля `pattern:(\w+\s)*`, далі (необов’язково) останнє слово `pattern:\w*`.

Цей регулярний вираз рівноцінний попередньому (збіг той самий) та працює відмінно:

```js run
let regexp = /^(\w+\s)*\w*$/;
let str = "Введений рядок, який оброблятиметься довго або навіть спровокує зависання регулярного виразу!";

alert( regexp.test(str) ); // false
```

Чому проблема зникла?

Бо тепер пробіл є обов’язковим.

Попередній регулярний вираз, якщо не врахувати пробіл, стає `pattern:(\w+)*`, призводячи до багатьох комбінацій `\w+` посеред єдиного слова.

Тож `subject:input` може мати збіг у вигляді двох повторень `pattern:\w+`:

```
\w+  \w+
(inp)(ut)
```

Новий шаблон інший: `pattern:(\w+\s)*` описує повторення слів, за якими йде пробіл! Рядок `subject:input` не буде збігом для двох повторень `pattern:\w+\s`, оскільки пробіл є обов’язковим.

Час проходження великою кількістю (взагалі-то, більшістю) комбінацій зекономлено.

## Запобігання поверненню

Все ж, не завжди зручно переписувати регулярний вираз. Попередній приклад був простим, інші бувають не надто очевидними.

До того ж, змінений регулярний вираз зазвичай більш складний, що не є добре. Регулярні вирази й так достатньо складні.

На щастя, мається альтернативний підхід. Можна заборонити повернення для квантифікатора.

Корінь проблеми в рушії регулярного виразу, що пробує багато очевидно невірних (для людини) комбінацій.

Наприклад, для людини очевидно, що в `pattern:(\d+)*$`, `pattern:+` не потребує пошуку з поверненням. Якщо замінити `pattern:\d+` на два окремих `pattern:\d+\d+`, нічого не зміниться:

```
\d+........
(123456789)!

\d+...\d+....
(1234)(56789)!
```

Також, можливо, всередині початкового прикладу `pattern:^(\w+\s?)*$` ми б хотіли заборонити пошук з поверненням в `pattern:\w+`. Отже, `pattern:\w+` має знаходити збіг цілому слову, з максимально можливою довжиною. Нема потреби знижувати кількість повторень в `pattern:\w+`, ділити на два слова `pattern:\w+\w+`, і таке інше.

Для цього, сучасні рушії регулярних виразів підтримують присвійні квантифікатори. Звичайні квантифікатори стають присвійними, якщо після них додати `pattern:+`. Саме так, ми використаємо `pattern:\d++` замість `pattern:\d+`, аби зупинити пошук з поверненням для `pattern:+`.

Насправді, присвійні квантифікатори є простішими за "звичайні". Вони просто шукають стільки збігів, скільки можуть, без будь-якого повернення. Такий процес пошуку, звичайно, простіший.

Також існують так звані "атомні групи захоплення" – спосіб відключити повернення всередині дужок.

...Погані новини: нажаль, JavaScript їх не підтримує.

Замість них можна використати "перетворення переглядом вперед".

### Вперед по допомогу!

Тож, ми підібрались до дійсно передової теми. Нам потрібен квантифікатор, як-то `pattern:+`, без пошуку з поверненням, тому що іноді це не має ніякого сенсу.

Шаблон з максимальною кількістю повторів `pattern:\w` без повернення: `pattern:(?=(\w+))\1`. Звісно, можемо взяти інший шаблон, замість `pattern:\w`.

Здається дивним, але це дуже просте перетворення.

Розберемо його:

- Перегляд вперед `pattern:?=` шукає найдовше слово `pattern:\w+`, починаючи з поточної позиції.
- Вміст дужок з `pattern:?=...` не запам’ятовується рушієм, тож огорнемо дужками `pattern:\w+`. В цьому випадку, рушій запам’ятає вміст дужок.
- ...Це дозволяє нам посилатись на них всередині шаблону: `pattern:\1`.

Тобто, ми дивимось вперед – якщо там є слово `pattern:\w+`, то воно відмітиться як `pattern:\1`.

Чому? Все через те, що перегляд вперед знаходить слово `pattern:\w+` повністю та ми беремо його в шаблон разом з `pattern:\1`. Отож, по суті, ми застосовуємо присвійний квантифікатор `pattern:+`. Він охоплює все слово `pattern:\w+`, а не якусь частину.

Для прикладу, в слові `subject:JavaScript` він не тільки знайде збіг `match:Java`, але й залише `match:Script` для пошуку збігу з рештою шаблону.

Порівняння двох шаблонів:

```js run
alert( "JavaScript".match(/\w+Script/)); // JavaScript
alert( "JavaScript".match(/(?=(\w+))\1Script/)); // null
```

1. В першому випадку, `pattern:\w+` спочатку бере ціле слово `subject:JavaScript`, але потім `pattern:+` символ за символом проводить пошук з поверненням, намагаючись знайти збіг для решти шаблону доти, доки не досягне цілі (коли `pattern:\w+` відповідає `match:Java`).
2. В другому випадку, `pattern:(?=(\w+))` дивиться вперед та знаходить слово `subject:JavaScript`, повністю включене в шаблон за допомогою `pattern:\1`, тож опісля нема ніякої можливості для пошуку `subject:Script`.

Ми можемо помістити більш комплексний регулярний вираз у `pattern:(?=(\w+))\1` замість `pattern:\w`, коли нам потрібно заборонити пошук з поверненням для `pattern:+` після нього.

```smart
Більше про зв’язок між присвійними квантифікаторами та переглядом вперед в статтях [Regex: Emulate Atomic Grouping (and Possessive Quantifiers) with LookAhead](http://instanceof.me/post/52245507631/regex-emulate-atomic-grouping-with-lookahead) та [Mimicking Atomic Groups](http://blog.stevenlevithan.com/archives/mimic-atomic-groups).
```

Перепишемо перший приклад, використовуючи перегляд вперед для запобігання пошуку з поверненням:

```js run
let regexp = /^((?=(\w+))\2\s?)*$/;

alert( regexp.test("Хороший рядок") ); // true

let str = "Введений рядок, який оброблятиметься довго або навіть спровокує зависання регулярного виразу!";

alert( regexp.test(str) ); // false, працює швидко!
```

Як бачимо, `pattern:\2` використовується замість `pattern:\1` через додаткові зовнішні дужки. Щоб уникнути плутанини з числами, ми можемо назвати дужки, наприклад, `pattern:(?<word>\w+)`.

```js run
// дужки мають ім’я ?<word>, на що посилається \k<word>
let regexp = /^((?=(?<word>\w+))\k<word>\s?)*$/;

let str = "Введений рядок, який оброблятиметься довго або навіть спровокує зависання регулярного виразу!";

alert( regexp.test(str) ); // false

alert( regexp.test("Правильний рядок") ); // true
```

Проблема, описана в статті, має назву "катастрофічний пошук з поверненням".

Ми розглянули два шляхи її вирішення:
- Переписати регулярний вираз для зменшення кількості можливих комбінацій.
- Запобігти поверненню.