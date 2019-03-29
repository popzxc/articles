_"Какого дьявола я должен помнить наизусть все эти чёртовы алгоритмы и структуры данных?"._

Примерно к этому сводятся комментарии большинства статей про прохождение технических интервью. Основной тезис, как правило, заключается в том, что всё так или иначе используемое уже реализовано по десять раз и с наибольшей долей вероятности заниматься этим рядовому программисту вряд ли придётся. Что ж, в какой-то мере это верно. Но, как оказалось, реализовано не всё, и мне, к сожалению (или к счастью?) создавать Структуру Данных всё-таки пришлось. 

Загадочное Modified Merkle Patricia Trie.

Так как на хабре информации об этом дереве нет вообще, а на медиуме -- немногим больше, хочу поведать о том, что же это за зверь, и с чем его едят.

[КПДВ]

Для начала, давайте разберемся, что это за структура и зачем она вообще нужна. Делать мы будем это по частям.

<cut />

_Disclaimer: основным источником информации при реализации для меня являлись [Yellow paper](https://ethereum.github.io/yellowpaper/paper.pdf), а также исходные коды [parity-ethereum](https://github.com/paritytech/parity-ethereum) и [go-ethereum](https://github.com/ethereum/go-ethereum). Теоретической информации по поводу обоснования тех или иных решений было минимум, поэтому все выводы по поводу причин принятия тех или иных решений - мои личные. В случае, если я в чем-то заблуждаюсь - буду рад исправлениям в комментариях._

## Что это?

_Дерево_ - структура данных, представляющая собой связный ациклический граф. Тут всё просто, все с этим знакомы.

_Префиксное дерево_ - корневое дерево, в котором можно хранить пары ключ-значение за счёт того, что узлы делятся на два типа: те, что содержат часть пути (префикс), и конечные узлы, которые содержат хранимое значение. Значение присутствует в дереве тогда и только тогда, когда мы, используя ключ, можем пройти от корня дерева весь путь и в конце найти узел со значением.

_PATRICIA-дерево_ - это префиксное дерево, в котором префиксы бинарны - то есть, каждый узел-ключ хранит информацию об одном бите.

_Дерево Мёркла_ - это дерево хешей, построенное над какой-то цепочкой данных, агрегирующее эти самые хеши в один (корневой), хранящий информацию о состоянии всех блоков данных. То есть, корневой хеш - это эдакая "цифровая подпись" состояния цепи блоков. Используется эта штука активно в блокчейне, и подробнее про неё можно почитать [тут](https://habr.com/ru/company/bitfury/blog/346398/).

[картинка-демонстрация всех 4 деревьев]

Итого: Modified Merkle Patricia Trie (далее MPT для краткости) - это дерево хешей, хранящее пары ключ-значение, при этом ключи представлены в бинарном виде. А в чем именно заключается "Modified" мы узнаем чуть позже, когда будем обсуждать реализацию.

## Зачем это?

MPT используется в проекте Ethereum для хранения данных об аккаунтах, транзакциях, результатах их выполнения и прочих данных, необходимых для функционирования системы.
В отличие от Bitcoin, в котором состояние неявно и вычисляется каждым узлом самостоятельно, в эфире баланс каждого аккаунта (а также ассоциированные с ним данные) хранятся непосредственно в блокчейне. Более того, нахождение и неизменность данных должны быть обеспечены криптографически -- мало кто станет пользоваться криптовалютой, в которой баланс случайного аккаунта может измениться без объективных на то причин. 

Основной проблемой, с которой столкнулись разработчики `Ethereum` - это создание структуры данных, которая позволяет эффективно хранить пары ключ-значение и при этом обеспечивать верификацию хранимых данных. Так и появилось MPT.

## Как это?

MPT является префиксным PATRICIA-деревом, в котором ключами являются последовательности байтов.

Ребрами в данном дереве являются последовательности нибблов (половинок байтов). Соответственно, у одного узла может быть до шестнадцати потомков (соответствующих веткам от 0x0 до 0xF).

Узлы делятся на 3 вида:

- Branch node. Узел, используемый для ветвления. Содержит до от 1 до 16 ссылок на дочерние узлы. Также может содержать значение.
- Extension node. Вспомогательный узел, который хранит какую-то часть пути, общую для нескольких дочерних узлов, а также ссылку на branch node, лежащий далее.
- Leaf node. Узел, содержащий часть пути и хранимое значение. Является конечным в цепочке.

Как уже было упомянуто, MPT строится поверх другого kv-хранилища, в котором хранятся узлы в виде "ссылка" => "`RLP`-закодированный узел".

_И тут мы встречаемся с новым понятием: RLP. Если коротко - то это метод кодирования данных, представляющих собой списки или байтовые последовательности. Пример: `[ "cat", "dog" ] = [ 0xc8, 0x83, 'c', 'a', 't', 0x83, 'd', 'o', 'g' ]`. Вдаваться в подробности я особо не буду, и в реализации для этого использую готовую библиотеку, поскольку освещение еще и данной темы слишком раздует и без того немаленькую статью. Если же вам всё же интересно, более подробно вы можете почитать [тут](https://github.com/ethereum/wiki/wiki/RLP). Мы же ограничимся тем, что мы можем закодировать данные в `RLP` и раскодировать их обратно._

Ссылка на узел же определяется следующим образом: в случае, если длина `RLP`-закодированного узла составляет 32 или более байт, то ссылкой является `keccak`-хеш от `RLP`-представления узла. Если же длина меньше 32 байт, то ссылкой является само `RLP`-представление узла.
Очевидно, что во втором случае сохранять узел в базу данных не нужно, т.к. он целиком будет сохранен внутри родительского узла.

[ картинка с видами узлов ]

Комбинация трёх видов узлов позволяет эффективно хранить данные и в случае, когда ключей мало (тогда большая часть путей будет храниться в extension и leaf нодах, а branch-узлов будет мало), и в случае, когда узлов много (пути не будут храниться явно, а будут "собираться" во время прохода по branch нодам).

Полный пример дерева, использущего все виды узлов:

[ картинка с деревом ]

Как вы могли заметить, хранимые части путей имеют префиксы. Префиксы нужны для нескольких целей:
1. Чтобы отличать extension-узлы от leaf-узлов. 
2. Чтобы выравнивать последовательности, состоящие из нечетного количества нибблов.

Примеры путей с разными префиксами:

[ картинка с 4 возможными префиксами и объяснением ]

### Удаление, которое не удаление

TODO

Это не все оптимизации. Там есть ещё, но об этом не будем -- и так статья большая. Впрочем, можете [почитать](https://blog.ethereum.org/2015/06/26/state-tree-pruning/) сами.

## Реализация

С теорией покончено, переходим к практике. Использовать будем лингва франка от мира IT, то бишь `python`.

Так как кода будет много, и для формата статьи многое придется сокращать и разделять, сразу же оставлю ссылку на [github](https://github.com/popzxc/merkle-patricia-trie).
В случае необходимости, там можно посмотреть на картину целиком.

Для начала определим интерфейс дерева, который мы хотим получить в итоге:

```python
class MerklePatriciaTrie:
    def __init__(self, storage, root=None):
        pass

    def root(self):
        pass

    def get(self, encoded_key):
        pass

    def update(self, encoded_key, encoded_value):
        pass

    def delete(self, encoded_key):
        pass
```

Интерфейс предельно прост. Доступные операции - получение, удаление, вставка и измененение (объединенные в update), а также получение корневого хеша.

В метод `__init__` будет передаваться хранилище - `dict`-like структура данных, в которой мы будем хранить узлы, а также `root` - "вершина" дерева. В случае, если в качестве `root` передан `None` - считаем, что дерево пустое и работаем с нуля.

_Ремарка: возможно, вам интересно, почему переменные в методах названы как `encoded_key` и `encoded_value`, а не просто `key`/`value`. Ответ прост: по спецификации все ключи и значения должны быть закодированы в `RLP`. Мы же этим себя утруждать не будем и оставим сие занятие на плечи пользователей библиотеки._

Однако, прежде чем мы приступим к реализации самого дерева, необходимо сделать две важных вещи:

1. Реализовать класс `NibblePath`, представляющий собой цепочки нибблов, чтобы не кодировать их вручную.
2. Реализовать класс `Node` и в рамках данного класса - `Extension`, `Leaf` и `Branch`.

### NibblePath

Итак, `NibblePath`. Так как по дереву мы будем активно передвигаться, основой функциональности нашего класса должна быть возможность задавать "смещение" от начала пути, а также получать конкретный ниббл. Зная это, определим основу нашего класса (а также пару полезных констант для работы с префиксами далее):

```python
class NibblePath:
    ODD_FLAG = 0x10
    LEAF_FLAG = 0x20

    def __init__(self, data, offset=0):
        self._data = data  # Данные, по которым итерируемся.
        self._offset = offset  # Насколько мы сдвинулись от начала

    def consume(self, amount):
        # "Поглощение" N нибблов можно реализовать через увеличение смещения.
        self._offset += amount
        return self

    def at(self, idx):
        # Определяем индекс нужного нам ниббла
        idx = idx + self._offset

        # Зная номер нужного ниббла, рассчитываем номер байта, в котором он лежит,
        # а также определяем, нужный нам ниббл - первый или второй в этом байте.
        byte_idx = idx // 2
        nibble_idx = idx % 2

        # Берем нужный байт.
        byte = self._data[byte_idx]

        # Берем нужный ниббл в этом байте.
        nibble = byte >> 4 if nibble_idx == 0 else byte & 0x0F

        return nibble
```

Всё довольно просто, не так ли?

Осталось написать только функции для кодирования и декодирования последовательности нибблов.

```python
class NibblePath:
    # ...

    def decode_with_type(data):
        # Смотрим на флаги: определяем, четное или нечетное количество нибблов, а также тип узла.
        is_odd_len = data[0] & NibblePath.ODD_FLAG == NibblePath.ODD_FLAG
        is_leaf = data[0] & NibblePath.LEAF_FLAG == NibblePath.LEAF_FLAG

        # Если количество нибблов четное, то за нибблом префикса идёт пустой ниббл выравнивания.
        # offset укажет нам, сколько нибблов нужно пропустить до начала "актуальных" данных.
        offset = 1 if is_odd_len else 2

        return NibblePath(data, offset), is_leaf

    def encode(self, is_leaf):
        output = []

        # Для начала нужно определить, четное или нечетное количество у нас нибблов.
        nibbles_len = len(self._data) * 2 - self._offset
        is_odd = nibbles_len % 2 == 1

        # Определяем префикс.
        prefix = 0x00
        # Если количество нибблов нечетное, то с нибблом префикса количество станет четным.
        # Сразу же берем первый ниббл (self.at(0)) и добавляем к байту префикса.
        # В противном случае за нибблом префикса должен быть пустой ниббл (0x0).
        prefix += self.ODD_FLAG + self.at(0) if is_odd else 0x00
        # Устанавливаем флаг, обозначающий Leaf node, если нужно.
        prefix += self.LEAF_FLAG if is_leaf else 0x00

        output.append(prefix)

        # Определяем, добавили ли мы уже один ниббл, или нет.
        pos = nibbles_len % 2

        # К этому моменту количество нибблов уже выравнено и кратно двум, поэтому просто берем по 2 ниббла,
        # формируем из них байт, и добавляем к закодированной последовательности, пока путь не кончится.
        while pos < nibbles_len:
            byte = self.at(pos) * 16 + self.at(pos + 1)
            output.append(byte)
            pos += 2

        return bytes(output)
```

В принципе, это минимум, необходимый для удобной работы с нибблами. Разумеется, в настоящей реализации есть еще некоторое количество вспомогательных методов (таких как `combine`, сливающий два пути в один), но их реализация весьма тривиальна. Если интересно -- полную версию можно найти [тут](https://github.com/popzxc/merkle-patricia-trie/blob/master/mpt/nibble_path.py).

### Node

Как мы уже знаем, узлы у нас делятся на три вида: Leaf, Extension и Branch. Все они могут быть закодированы и декодированы, а единственное отличие - данные, которые хранятся внутри. Если честно, тут так и просятся алгебраические типы данных, и в `Rust`, например, я бы написал что-то в духе:

```rust
pub enum Node<'a> {
    Leaf(NibblesSlice<'a>, &'a [u8]),
    Extension(NibblesSlice<'a>, NodeReference),
    Branch([Option<NodeReference>; 16], Option<&'a [u8]>),
}
```

Однако, в питоне как таковых АТД нет, поэтому мы определим класс `Node`, а внутри него -- три класса, соответствующие типам узлов. Кодирование реализуем непосредственно в классах узлов, а декодирование -- в классе `Node`.

Реализация, тем не менее, элементарна:

Leaf:

```python
class Leaf:
    def __init__(self, path, data):
        self.path = path
        self.data = data

    def encode(self):
        # Закодированная версия узла -- список из двух элементов,
        # где первый - путь нибблов, а второй - хранимые данные.
        return rlp.encode([self.path.encode(True), self.data])
```

Extension:
```python
class Extension:
    def __init__(self, path, next_ref):
        self.path = path
        self.next_ref = next_ref

    def encode(self):
        # Закодированная версия узла -- список из двух элементов,
        # где первый - путь нибблов, а второй - ссылка на следующий узел.
        next_ref = _prepare_reference_for_encoding(self.next_ref)
        return rlp.encode([self.path.encode(False), next_ref])
```

Branch:
```python
class Branch:
    def __init__(self, branches, data=None):
        self.branches = branches
        self.data = data

    def encode(self):
        # Закодированная версия узла -- список из семнадцати элементов,
        # где первые 16 - ссылки на дочерние узлы (некоторые могут отсутствовать),
        # а семнадцатый - хранимые данные (также могут отсутствовать).
        branches = list(map(_prepare_reference_for_encoding, self.branches))
        return rlp.encode(branches + [self.data])
```

Всё элементарно. Единственное, что может вызвать вопросы -- функция `_prepare_reference_for_encoding`.

_Тут каюсь, пришлось использовать небольшой костыль. Дело в том, что используемая библиотека `rlp` декодирует данные рекурсивно, а ссылка на другой узел, как мы знаем, может быть `rlp`-данными (в случае, если длина закодированного узла менее 32 символов). Работать с ссылками в двух форматах -- байты хеша и декодированная нода -- весьма неудобно. Поэтому я написал две функции, которые после расшифровки узла возвращают ссылки в формат байтов, а перед сохранением, если нужно, декодируют их. Вот эти функции:_

```python
def _prepare_reference_for_encoding(ref):
    # Если ссылка короткая (то есть, не является хешом) -- раскодируем её.
    # Перед сохранением она будет закодирована обратно :)
    if 0 < len(ref) < 32:
        return rlp.decode(ref)

    return ref


def _prepare_reference_for_usage(ref):
    # Если ссылка была раскодирована - кодируем её обратно.
    # В результате вне зависимости от типа ссылки работаем с байтиками.
    if isinstance(ref, list):
        return rlp.encode(ref)

    return ref
```

Закончим с узлами, написав класс `Node`. В нём будет всего 2 метода: раскодировать узел и превратить узел в ссылку.

```python

class Node:
    # class Leaf(...)
    # class Extension(...)
    # class Branch(...)

    def decode(encoded_data):
        data = rlp.decode(encoded_data)

        # 17 элементов - гарантированно Branch узел.
        if len(data) == 17:
            branches = list(map(_prepare_reference_for_usage, data[:16]))
            node_data = data[16]
            return Node.Branch(branches, node_data)

        # Если узлов не 17, значит их 2. И первый - путь. Декодируем его и смотрим на то, что же это за узел.
        path, is_leaf = NibblePath.decode_with_type(data[0])
        if is_leaf:
            return Node.Leaf(path, data[1])
        else:
            ref = _prepare_reference_for_usage(data[1])
            return Node.Extension(path, ref)

    def into_reference(node):
        # Превращаем узел в ссылку.
        # Если длина закодированного узла меньше 32 байтов, то ссылка - сам закодированный узел.
        # В противном случае считаем хеш от данных.
        encoded_node = node.encode()
        if len(encoded_node) < 32:
            return encoded_node
        else:
            return keccak_hash(encoded_node)
```

## Перерыв

Фух! Информации получается много. Думаю, самое время отдохнуть и посмотреть на котиков.

[картинка с котиками]

Милота, правда? Ладно, вернемся к нашим деревьям.

## MerklePatriciaTrie

Ура -- вспомогательные элементы готовы, переходим к самому вкусному. На всякий случай напомню интерфейс нашего дерева. Заодно реализуем самые простые части.

```python
class MerklePatriciaTrie:
    def __init__(self, storage, root=None):
        self._storage = storage
        self._root = root

    def root(self):
        return self._root

    def get(self, encoded_key):
        pass

    def update(self, encoded_key, encoded_value):
        pass

    def delete(self, encoded_key):
        pass
```

А вот с оставшимися методами будем разбираться по одному.

### get

Метод `get` (как, в принципе, и остальные методы) будут у нас состоять из двух частей. Сам метод будет производить подготовку данных и приведение результата к ожидаемой форме, тогда как настоящая работа будет происходить внутри вспомогательного метода.

Основной метод чрезвычайно прост:

```python
class MerklePatriciaTrie:
    # ...
    def get(self, encoded_key):
        if not self._root:
            raise KeyError

        path = NibblePath(encoded_key)

        # Вспомогательному методу нужно передать ссылку на корневой узел, а так же путь, по которому нужно пройти.
        result_node = self._get(self._root, path)

        if type(result_node) is Node.Extension or len(result_node.data) == 0:
            raise KeyError

        return result_node.data
```

Впрочем, `_get` не сильно сложнее: для того, чтобы добраться до искомого узла, нам нужно пройти от корня весь предоставленный путь. Если в конце мы обнаружили узел с данными (Leaf или Branch) -- ура, данные получены. Если же пройти не удалось -- то искомый ключ отсутствует в дереве.

Реализация:

```python
class MerklePatriciaTrie:
    # ...
    def _get(self, node_ref, path):
        # Получаем ноду по ссылке из хранилища.
        node = self._get_node(node_ref)

        # Если путь пустой -- наше приключение закончено. Родительский метод проверит, есть ли в этой ноде данные.
        if len(path) == 0:
            return node

        if type(node) is Node.Leaf:
            # Если мы дошли до Leaf-узла, то либо это искомый узел, либо нужный ключ отсутсвует.
            if node.path == path:
                return node

        elif type(node) is Node.Extension:
            # Если текущий узел -- Extension, нам нужно спускаться дальше.
            if path.starts_with(node.path):
                rest_path = path.consume(len(node.path))
                return self._get(node.next_ref, rest_path)

        elif type(node) is Node.Branch:
            # Если текущий узел -- Branch, просто идём в соответствующую ветку.
            # Заботиться о том, что путь окончен и нам нужны данные из этого узла не нужно:
            # это уже проверено в начале метода.
            branch = node.branches[path.at(0)]
            if len(branch) > 0:
                return self._get(branch, path.consume(1))

        # Если мы оказались тут, значит ни один из приемлемых вариантов не сработал, и ключ в дереве отсутствует.
        raise KeyError
```

Ну и заодно напишем методы для сохранения и загрузки узлов. Они простые:

```python
class MerklePatriciaTrie:
    # ...
    def _get_node(self, node_ref):
        raw_node = None
        if len(node_ref) == 32:
            raw_node = self._storage[node_ref]
        else:
            raw_node = node_ref
        return Node.decode(raw_node)

    def _store_node(self, node):
        reference = Node.into_reference(node)
        if len(reference) == 32:
            self._storage[reference] = node.encode()
        return reference
```

### update

Метод `update` уже интереснее. Просто пройти до конца и вставить Leaf-узел получится далеко не всегда. Вполне вероятна ситуация, что точка разделения ключей будет находиться где-то внутри уже сохраненного Leaf или Extension узла. В таком случае придется их разделить и создать несколько новых узлов.

В целом, всю логику можно описать следующими правилами:
1. Пока путь целиком совпадает с уже имеющимися узлами, рекурсивно спускаемся по дереву.
2. Если путь окончен и мы находимся в Branch или Leaf узле -- значит, `update` просто обновляет значение, соответствующее данному ключу.
3. Если пути разделились (то есть, мы не обновляем значение, а вставляем новое), и мы находимся в Branch-узле -- создаем Leaf-узел и указываем ссылку на него в соответствующей ветке Branch-узла.
4. Если пути разделились и мы находися в Leaf или Extension узле -- нам нужно создать Branch узел, разделяющий пути, а также, если необходимо -- Extension узел для общей части пути.

Давайте постепенно выражать это в коде. Почему постепенно? Потому что метод большой и скопом его понять будет тяжеловато.
Впрочем, ссылку на полный метод я оставлю [тут](https://github.com/popzxc/merkle-patricia-trie/blob/master/mpt/mpt.py#L179).

```python
class MerklePatriciaTrie:
    # ...
    def update(self, encoded_key, encoded_value):
        path = NibblePath(encoded_key)

        result = self._update(self._root, path, encoded_value)

        self._root = result

    def _update(self, node_ref, path, value):
        # Если ссылка на узел не предоставлена (например, дерево пока пустое),
        # значит нам нужно просто создать новый узел.
        if not node_ref:
            return self._store_node(Node.Leaf(path, value))

        # В противном случае мы смотрим на тип текущего узла и ориентируемся по ситуации.
        node = self._get_node(node_ref)

        if type(node) == Node.Leaf:
            ...

        elif type(node) == Node.Extension:
            ...

        elif type(node) == Node.Branch:
            ...
```

Общей логики довольно мало, все самое интересное находится внутри `if`ов.

##### `if type(node) == Node.Leaf`

```python
if type(node) == Node.Leaf:
    # If we're updating the leaf there are 2 possible ways:
    # 1. Path is equals to the rest of the key. Then we should just update value of this leaf.
    # 2. Path differs. Then we should split this node into several nodes.

    if node.path == path:
        # Path is the same. Just change the value.
        node.data = value
        return self._store_node(node)

    # If we are here, we have to split the node.

    # Find the common part of the key and leaf's path.
    common_prefix = path.common_prefix(node.path)

    # Cut off the common part.
    path.consume(len(common_prefix))
    node.path.consume(len(common_prefix))

    # Create branch node to split paths.
    branch_reference = self._create_branch_node(path, value, node.path, node.data)

    # If common part isn't empty, we have to create an extension node before branch node.
    # Otherwise, we need just branch node.
    if len(common_prefix) != 0:
        return self._store_node(Node.Extension(common_prefix, branch_reference))
    else:
        return branch_reference
```

##### `if type(node) == Node.Extension`

```python
elif type(node) == Node.Extension:
    # If we're updating an extenstion there are 2 possible ways:
    # 1. Key starts with the extension node's path. Then we just go ahead and all the work will be done there.
    # 2. Key doesn't start with extension node's path. Then we have to split extension node.

    if path.starts_with(node.path):
        # Just go ahead.
        new_reference = self._update(node.next_ref, path.consume(len(node.path)), value)
        return self._store_node(Node.Extension(node.path, new_reference))

    # Split extension node.

    # Find the common part of the key and extension's path.
    common_prefix = path.common_prefix(node.path)

    # Cut off the common part.
    path.consume(len(common_prefix))
    node.path.consume(len(common_prefix))

    # Create an empty branch node. It may have or have not the value depending on the length
    # of the rest of the key.
    branches = [b''] * 16
    branch_value = value if len(path) == 0 else b''

    # If needed, create leaf branch for the value we're inserting.
    self._create_branch_leaf(path, value, branches)
    # If needed, create an extension node for the rest of the extension's path.
    self._create_branch_extension(node.path, node.next_ref, branches)

    branch_reference = self._store_node(Node.Branch(branches, branch_value))

    # If common part isn't empty, we have to create an extension node before branch node.
    # Otherwise, we need just branch node.
    if len(common_prefix) != 0:
        return self._store_node(Node.Extension(common_prefix, branch_reference))
    else:
        return branch_reference
```

##### `if type(node) == Node.Branch`

```python
elif type(node) == Node.Branch:
    # For branch node things are easy.
    # 1. If key is empty, just store value in this node.
    # 2. If key isn't empty, just call `_update` with appropiate branch reference.

    if len(path) == 0:
        return self._store_node(Node.Branch(node.branches, value))

    idx = path.at(0)
    new_reference = self._update(node.branches[idx], path.consume(1), value)

    node.branches[idx] = new_reference

    return self._store_node(node)
```

### delete

### Остальное

## Тестирование

*ссылка на тесты от эфира*

## Результаты