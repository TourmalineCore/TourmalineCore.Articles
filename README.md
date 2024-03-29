# TourmalineCore.Articles

## Шаги по созданию статьи

1. Создать ветку статьи 

2. Создать папку с кратким названием статьи по пути **TourmalineCore.Articles/articles/ru**

3. В этой папке создаем файл **metadata.json**, в нем пишем:

```
{
  "title": "Полное название статьи ( на языке статьи )",
  "description": "Полное название статьи ( на языке статьи )",
  "slug": "Ключевые слова",
  "datePublication": "Дата публикации в формате дд.мм.гггг",
  "author": "Имя и фамилия автора ( на языке статьи )",
  "categories": ["Тег направления разработки, которые будут отображаться на сайте", "тег"],
  "previewImage": "Название файла обложки статьи" 
}
```

пример заполнения:

```
{
  "title": "E2E testing user's flow between DApp and MetaMask using WalletConnect protocol",
  "description": "E2E-тестирование подключения по WalletConnect между DApp и мобильным приложением Metamask",
  "slug": "react-native-detox-e2e",
  "datePublication": "11.10.2022",
  "author": "Илья Сапронов",
  "categories": ["Quality assurance"],
  "previewImage": "Preview.webp" 
}
```

В этой же папке создаем файл **name.md**, где **name** - краткое название статьи (такое же как название папки). В этом файле пишем статью.

Для вставки изображений создаем папку **images** по пути TourmalineCore.Articles/articles/ru/краткое_название_статьи и добавляем в неё изображения, на которые будем ссылаться.


## Шаги по публикации статьи

После того, как сделали последний commit & push (используйте корректное наименование)

1. Заходим в GitHub [репозиторий](https://github.com/TourmalineCore/TourmalineCore.Articles)

2. Нажимаем зеленую кнопку `Compare & pull request`

3. Дальше нажимаем кнопку `Create pull request`

4. Нажимаем кнопку `Squash and merge`

5. Нажимаем кнопку `Confirm squash and merge`

6. Проверяем на корп. сайте, если ошибок нет, оно само запаблишится


## Разметка файла readme.md

### 1. Заголовки

```
# Название статьи - заголовок 1 уровня

## Название раздела - заголовок 2 уровня

### Название небольшого смыслового блока в разделе - Заголовок 3 уровня

и так далее до 6 уровня
```

# Название статьи - заголовок 1 уровня

## Название раздела - заголовок 2 уровня

### Название небольшого смыслового блока в разделе - Заголовок 3 уровня

### 2. Выделение блоков кода:

1. В начале и в конце поставить 3 символа **`**
2. Название языка программирования, на котором будем писать
3. Код

### 3. Выделение текста

```
~~Зачеркнутый текст~~

**Жирный текст**

*Наклонный текст*

***Жирный наклонный текст***

~~***Жирный наклонный зачеркнутый текст***~~
```

### 4. Изображения 

Вставить изображение и отцентровать его по горизонтали:

```
<p align="center">
  <img src="images/sprint.gif" loop=infinite>
</p>
```
(где images - папка в которой лежит изображение, а sprint.gif - название изображения)

Если центрировать не нужно то пишем так:
```
<p>
  <img src="images/sprint.gif" loop=infinite>
</p>
```

**!Для картинок обязательно использовать тег `<img>`**

**!Тег `style` не работает**

### 5. Таблицы

Вставка таблицы:

```
<table>
    <thead>
        <tr>
            <th>заголовок1</th>
            <th>заголовок2</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td valign="top">контент1</td>
            <td valign="top">контент2</td>
        </tr>
    </tbody>
</table>
```

Выглядеть будет так:

<table>
    <thead>
        <tr>
            <th>заголовок1</th>
            <th>заголовок2</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td valign="top">контент1</td>
            <td valign="top">контент2</td>
        </tr>
    </tbody>
</table>

### Ссылки

```
[текст ссылки](адрес ссылки)
```


### 6. Списки

```
Маркированный:
* раз
* два
* три 
```

Маркированный:
* раз
* два
* три 


```
Нумерованный:
1. раз
2. два
3. три 
```

Нумерованный:
1. раз
2. два
3. три 

### Более подробное описание разметки файла README.md можно посмотреть по [ссылке](https://github.com/GnuriaN/format-README)