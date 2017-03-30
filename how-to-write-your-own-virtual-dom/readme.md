# Как написать Virtual Dom

## Перевод статьи [how-to-write-your-own-virtual-dom](https://medium.com/@deathmood/how-to-write-your-own-virtual-dom-ee74acc13060)

Есть 2 вещи, которые необходимо знать. Вам не нужно погружаться в исходный код React’а или других библиотек,
они довольно большие и сложные, на самом деле Virtual DOM может быть написан в меньше чем 50 строк кода.

1. Virtual DOM это аналог настоящего DOM
2. Когда мы меняем что-то в дереве Virtual DOM, мы создаем новый Virtual DOM.
Алгоритм сравнения двух деревьев вычисляет разницу и вносит только необходимые, минимальные правки в настоящий DOM.

## Представление нашего DOM дерева
Ну, во-первых, нам нужно как-то хранить наше DOM дерево в памяти. И мы можем сделать это с помощью обычных JS объектов.
Предположим, у нас есть это дерево:

```
<ul class="list">
    <li>item 1</li>
    <li>item 2</li>
</ul>
```

Как это будет выглядеть в JS:
```
{
  type: 'ul', props: { 'class': 'list' }, children: [
    { type: 'li', props: {}, children: ['item 1'] },
    { type: 'li', props: {}, children: ['item 2'] }
  ]
}
```

Здесь вы можете заметить две вещи:
* Мы описываем DOM элементы как JS объекты
```
{ type: '…', props: { … }, children: [ … ] }
```
* Мы описываем текстовые узлы как JS строки

Но писать большие деревья таким способом довольно сложно. Так давайте напишем вспомогательную функцию,
которая упростит понимание структуры:
```
function helper(type, props, …children) {
  return { type, props, children };
}
```
Теперь деревья можно создавать так:

```
helper('ul', { class: 'list' },
  helper('li', {}, 'item 1'),
  helper('li', {}, 'item 2'),
);
```

Стало лучше? Мы можем пойти дальше. Вы ведь слышали про JSX, не так ли?
Если вы читали документацию babel [тут](https://babeljs.io/docs/plugins/transform-react-jsx/), 
вы должны знать, что данный код:

```
<ul className=”list”>
  <li>item 1</li>
  <li>item 2</li>
</ul>
```
будет скомпилирован в

```
React.createElement('ul', { className: 'list' },
  React.createElement('li', {}, 'item 1'),
  React.createElement('li', {}, 'item 2'),
);
```
Заметили сходство с нашей функцией? Мы можем заменить React.createElement на helper, используя комментарий из документации:
```
/** @jsx helper */
<ul className=”list”>
  <li>item 1</li>
  <li>item 2</li>
</ul>
```
Так мы заставим babel заменить React.createElement на наш helper.

## Применение нашего DOM дерева

Теперь, когда мы имеем DOM представление в обычном JS объекте, имеющем свою собственную структуру, нам нужно создать механизм, который сможет строить из нее настоящий DOM.
Сначала давайте сделаем некоторые предположения и введем терминологию:
* Названия переменных с реальными DOM элементами будут начинаться с символа — $.
* Virtual DOM будет храниться в переменной с именем node.
* Аналогично React все узлы будут хранится в корневом элементе

Теперь напишем функцию, которая вернет DOM элемент, пока без «props» и «children»:
```
function createElement(node) {
  const isString = typeof node === 'string';
  return isString && document.createTextNode(node)
    || document.createElement(node.type);
}
```
Так мы можем создавать текстовые узлы и элементы из JS объектов.
Теперь используем нашу функцию для создания дочерних элементов, при помощи рекурсии 🙂

```
function createElement(node) {
  if (typeof node === ‘string’) {
    return document.createTextNode(node);
  }
  const $el = document.createElement(node.type);
  node.children
    .map(createElement)
    .forEach($el.appendChild.bind($el));
  return $el;
}
```
Добавление обработки «props» пока что опустим, чтобы не усложнять.

## Обработка изменений

Ок, теперь мы можем превратить наш виртуальный дом в настоящий,
пора подумать про сравние наших виртуальных деревьев. 
Нам нужно написать алгоритм, который будет сравнивать два виртуальных дерева
— старое и новое и делать только необходимые изменения в настоящем доме.

Для получения нового дерева нам нужно написать класс, предоставляющий слудующие методы:
* appendChild(…)
* removeChild(…)
* replaceChild(…)
* Метод который будет «заглядывать внутрь», в случае если узлы одинаковые

Ок, давайте напишем функцию updateElement(), которая принимает 3 параметра: $parent, oldNode, newNode. 
Где $parent элемент реального DOM узла родительского блока. Ниже рассмотрим все случаи обработки узлов.

### Если нет oldNode

Все довольно просто:

```
function updateElement($parent, newNode, oldNode) {
  if (!oldNode) {
    $parent.appendChild(
      createElement(newNode)
    );
  }
}
```

### Если нет нового узла (newNode)

Тут проблема — если нет узла для текущего места, мы должны удалить это из реального DOM. Мы также можем передавать
позицию узла для удаления

```
function updateElement($parent, newNode, oldNode, index = 0) {
  if (!oldNode) {
    $parent.appendChild(
      createElement(newNode)
    );
  } else if (!newNode) {
    $parent.removeChild(
      $parent.childNodes[index]
    );
  }
}
```

### Измененный узел

Напишем функцию, которая сравнивает 2 узла. Мы должны учитывать, что это могут быть как элементы так и текстовые узлы:


```
function changed(node1, node2) {
  return typeof node1 !== typeof node2 ||
         typeof node1 === ‘string’ && node1 !== node2 ||
         node1.type !== node2.type
}
```

И теперь, имея индекс текущего узла родителя, мы можем легко заменить его на вновь созданный узел:

```
function updateElement($parent, newNode, oldNode, index = 0) {
  if (!oldNode) {
    $parent.appendChild(
      createElement(newNode)
    );
  } else if (!newNode) {
    $parent.removeChild(
      $parent.childNodes[index]
    );
  } else if (changed(newNode, oldNode)) {
    $parent.replaceChild(
      createElement(newNode),
      $parent.childNodes[index]
    );
  }
}
```

## Различия дочерних элементов

И последнее, но не менее важное — мы должны пройти через каждого ребенка на обоих узлах и сравнивать их 
— если есть отличия, то вызвать updateElement(…) для каждого из них. Да, рекурсия снова.

```
function updateElement($parent, newNode, oldNode, index = 0) {
  if (!oldNode) {
    $parent.appendChild(
      createElement(newNode)
    );
  } else if (!newNode) {
    $parent.removeChild(
      $parent.childNodes[index]
    );
  } else if (changed(newNode, oldNode)) {
    $parent.replaceChild(
      createElement(newNode),
      $parent.childNodes[index]
    );
  } else if (newNode.type) {
    const newLength = newNode.children.length;
    const oldLength = oldNode.children.length;
    for (let i = 0; i < newLength || i < oldLength; i++) {
      updateElement(
        $parent.childNodes[index],
        newNode.children[i],
        oldNode.children[i],
        i
      );
    }
  }
}
```

## Собираем все вместе
Как я и обещал < 50 строк кода

```
function helper(type, props, ...children) {
  return { type, props, children };
}

function createElement(node) {
  if (typeof node === 'string') {
    return document.createTextNode(node);
  }
  const $el = document.createElement(node.type);
  node.children
    .map(createElement)
    .forEach($el.appendChild.bind($el));
  return $el;
}

function changed(node1, node2) {
  return typeof node1 !== typeof node2 ||
         typeof node1 === 'string' && node1 !== node2 ||
         node1.type !== node2.type
}

function updateElement($parent, newNode, oldNode, index = 0) {
  if (!oldNode) {
    $parent.appendChild(
      createElement(newNode)
    );
  } else if (!newNode) {
    $parent.removeChild(
      $parent.childNodes[index]
    );
  } else if (changed(newNode, oldNode)) {
    $parent.replaceChild(
      createElement(newNode),
      $parent.childNodes[index]
    );
  } else if (newNode.type) {
    const newLength = newNode.children.length;
    const oldLength = oldNode.children.length;
    for (let i = 0; i < newLength || i < oldLength; i++) {
      updateElement(
        $parent.childNodes[index],
        newNode.children[i],
        oldNode.children[i],
        i
      );
    }
  }
}

// ---------------------------------------------------------------------

const a = (
  <ul>
    <li>item 1</li>
    <li>item 2</li>
  </ul>
);

const b = (
  <ul>
    <li>item 1</li>
    <li>hello!</li>
  </ul>
);

const $root = document.getElementById('root');
const $reload = document.getElementById('reload');

updateElement($root, a);
$reload.addEventListener('click', () => {
  updateElement($root, b, a);
});
```

## Вывод
Поздравляю! Мы сделали это.
Мы написали реализацию Virtual DOM. И это работает.
Я надеюсь, что прочитав эту статью, вы поняли основные понятия и то, как Virtual DOM должен работать под капотом.

Однако, есть вещи, которые мы пропустили:

* Установка атрибутов, сравнение и замена
* Обработчики событий для наших элементов
* Создание компонент аналогичных React
* Получение ссылок на реальный дом узлы
* Использование Virtual DOM с библиотеками, которые непосредственно мутируют настоящий дом — такие, как jQuery и ее плагины.
* И много другое…


