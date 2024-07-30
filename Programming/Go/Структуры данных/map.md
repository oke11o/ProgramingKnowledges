---
создал заметку: 2024-07-28
tags:
  - golang
  - internal
---
### Описание

**map** — это "ссылочный" тип данных, предназначенный для хранения пар ключ-значение. В других языках эту структуру так же называют: хэш-таблица, словарь, ассоциативный массив. Запись и чтение элементов происходят в основном за O(1).

```go
// Объявление map

m := make(map[key_type]value_type)
m := new(map[key_type]value_type)
var m map[key_type]value_type
m := map[key_type]value_type{key1: val1, key2: val2}

m[key] = value      // Вставка
delete(m, key)      // Удаление
value := m[key]     // Поиск
value, ok := m[key] // Поиск, но второе значение указывает есть ли элемент
```

Так как `map` это "ссылочный" тип и мало объявить переменную, необходимо ее проинициализировать, в противном случае будет *паника*:
```go
func main() {  
    var m map[int]string  
    m[0] = "123" // panic: assignment to entry in nil map
}

func main() {  
    var m map[int]string  
    m = make(map[int]string)  
    m[0] = "123"  
    fmt.Println(m) // map[0:123]
}
```

#### Обход map

```go
func main() {
	m := map[int]bool{}
	for i := 0; i < 5; i++ {
		m[i] = ((i % 2) == 0)
	}
	for k, v := range m {
		fmt.Printf("key: %d, value: %t\n", k, v)
	}
}
```

Запуск 1:  

```
key: 3, value: false
key: 4, value: true
key: 0, value: true
key: 1, value: false
key: 2, value: true
```

Запуск 2:  

```
key: 4, value: true
key: 0, value: true
key: 1, value: false
key: 2, value: true
key: 3, value: false
```

Вывод разнится от запуска к запуску. А все потому, что мапа в Go **unordered**, то есть не упорядоченная. Полагаться на порядок при обходе не стоит. Место поиска определяется **рандомно**:

```go
// mapiterinit initializes the hiter struct used for ranging over maps.
func mapiterinit(t *maptype, h *hmap, it *hiter) {
...
// decide where to start
r := uintptr(fastrand())
...
it.startBucket = r & bucketMask(h.B)...}
```
#### Передача в функции

Ситуация похожая со [слайсами](slice.md). `map` - это указатель на структуру `hmap`, поэтому при передаче `map` в функцию, копируется значение указателя:
```go
func foo(m map[int]int) { 
    m[10] = 10 
}

func main() {
	m := make(map[int]int)
	m[10] = 15
	println("m[10] before foo =", m[10])
	foo(m)
	println("m[10] after foo =", m[10])
}
```

Вывод:
```
m[10] before foo = 15
m[10] after foo = 10
```

#### Внутреннее устройство

`map` - это указатель на структуру `hmap`:
```go
type hmap struct {  
    count     int // # live cells == size of map. Must be first (used by len() builtin)
    flags     uint8
    B         uint8  // log_2 of # of buckets
    noverflow uint16 // approximate number of overflow buckets
    hash0     uint32 // hash seed
    
    buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.  
    oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
    nevacuate  uintptr // progress counter for evacuation (buckets less than this have been evacuated)
	
    extra *mapextra // optional fields  
}
```

- `count`: количество элементов
- `B`: логарифм от числа `buckets` (степень двойки, определяющая количество `buckets`)
- `hash0`: *seed* для рандомизации хэшей (чтобы было сложнее заddosить — попытаться подобрать ключи так, что будут сплошные коллизии)
- `buckets`: Указатель на массив бакетов, в котором и хранятся ключи-значения
- `oldbuckets`: Указатель на прошлый массив бакетов, необходим для расширения количества бакетов

![](https://i.imgur.com/fq4UbB4.png)

**Low order bits (LOB)** - необходим для поиска нужного бакета по предоставленному значению хеш-функции.

**Вычисление LOB**

Пусть всего количество бакетов = 4, тогда:
```
Hash(key) = 5461 // Число полученное в результате работы хеш-функции
5461 % 4 = 1 // Искомый индекс бакета - это остаток от деления числа на кол-во бакетов
```

Для ускорения вычисления операция деления выполняется побитово:
```
B = 2 // log_2(4) - Это хранится в структуре hmap
Hash(key) = 5461

a = 1010101010101 // Переводим число в двоичный вид

// Маска получается за счет операции: 1<<B - 1
mask = 0000000000011 // Маска будет состоять из двух младших битов, остальные биты — нули

// Операция AND оставляет только два младших бита числа 5461:
5161 % 4 = 1010101010101 & 0000000000011 = 01 = 1 // Получили искомый индекс бакета

LOB(Hash) = 01 // Вычисленный LOB
```

**Структура бакета**

![](https://i.imgur.com/cXCE0RH.png)

8 слотов для **High order bits (HOB)** - 8 старших бит хеша (или меньше, если хеш короче). Они нужны для быстрой проверки нахождения нужного ключа в бакете. Это будет быстрее, чем проверять все ключи, которые хранятся в бакете.

Если HOB не найден, то такого ключа в мапе нет, а если нашелся, то полноценно сравниваем каждый ключ и возвращаем значение.

**В каждом бакете хранится только 8 пар ключ-значение**.

**В памяти бакет хранится в таком виде:**
![](https://i.imgur.com/iZIyNfj.png)
Сначала идут ключи, потом значения. Такой порядок разработчики сделали неслучайно, это связано с *выравниванием типов (type alignment)*.

**Поиск значения в `map`**

Осуществляется за счет функций `mapaccess1` и `mapaccess2`

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {  
    
    ...
	
	// Если map пустая, то сразу прерываем поиск
    if h == nil || h.count == 0 {  
       if err := mapKeyError(t, key); err != nil {  
          panic(err) // see issue 23734  
       }  
       return unsafe.Pointer(&zeroVal[0])  
    }
	
	...
	
	// Получение хеша от переданного ключа
    hash := t.Hasher(key, uintptr(h.hash0))
    
	// Получение маски LOB
    m := bucketMask(h.B)
    
    // Эмуляция ссылочной арифметики для получения нужного бакета
    b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.BucketSize)))
	
	...
	
	// Получение HOB
    top := tophash(hash)  
	
bucketloop:  
    for ; b != nil; b = b.overflow(t) {
	   // Перебор всех записей в бакете
       for i := uintptr(0); i < bucketCnt; i++ {
          // Проверяем сначала HOB
          if b.tophash[i] != top {  
             if b.tophash[i] == emptyRest {  
                break bucketloop  
             }  
             continue  
          }
          // Если HOB был найден, то получаем искомый ключ
          k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))  
          if t.IndirectKey() {  
             k = *((*unsafe.Pointer)(k))  
          }
	      // Сравниваем полученный ключ и если они равны, то возвращаем значение
          if t.Key.Equal(key, k) {  
             e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.KeySize)+i*uintptr(t.ValueSize))  
             if t.IndirectElem() {  
                e = *((*unsafe.Pointer)(e))  
             }  
             return e  
          }  
       }  
    }
    // Если не нашли - вернем Zero Value
    return unsafe.Pointer(&zeroVal[0])  
}
```

`mapaccess2` - работает примерно также как и `mapaccess1`, но возвращает вторым значением есть ли ключ в мапе:
```go
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool)
```

**Переполнение бакета**

Если возникла ситуация, когда в бакет нужно положить 9ое значение, то в таком случае создается новый бакет и ссылка на него сохраняется в изначальном бакете:
![](https://i.imgur.com/W3BpOuJ.png)

Если бакеты будут таким образом растягиваться, то поиск будет уже не константным, а линейным. Для того, чтобы этого избежать необходимо в определенные моменты расширять количество бакетов. Запускается это, когда среднее количество элементов во всех бакетах = **6.5**:

```go
// Picking loadFactor: too large and we have lots of overflow  
// buckets, too small and we waste a lot of space. I wrote  
// a simple program to check some stats for different loads:  
// (64-bit, 8 byte keys and elems)  
//  loadFactor    %overflow  bytes/entry     hitprobe    missprobe  
//        4.00         2.13        20.77         3.00         4.00  
//        4.50         4.05        17.30         3.25         4.50  
//        5.00         6.85        14.77         3.50         5.00  
//        5.50        10.55        12.94         3.75         5.50  
//        6.00        15.27        11.67         4.00         6.00  
*//        6.50        20.90        10.79         4.25         6.50*
//        7.00        27.14        10.15         4.50         7.00  
//        7.50        34.03         9.73         4.75         7.50  
//        8.00        41.10         9.40         5.00         8.00  
//  
// %overflow   = percentage of buckets which have an overflow bucket  
// bytes/entry = overhead bytes used per key/elem pair  
// hitprobe    = # of entries to check when looking up a present key  
// missprobe   = # of entries to check when looking up an absent key
```

**Эвакуация данных из бакета** - создается новый список бакетов, который будет в *два* раза больше предыдущего и данные из старого бакета будут скопированы в новый, но этот процесс не быстрый. Поэтому сначала выделится память, а операции переноса будут происходить в те моменты, когда происходят операции *сохранения/удаления* ключей из мапы, то есть это выполняется *инкрементально*

![](https://i.imgur.com/SIkDGkS.png)

В момент эвакуации данные будут искаться в двух местах (`oldbuckets` и `buckets`) поэтому это займет больше времени. Единственный способ избежать этого - выделять память мапе заранее:

```go
m := make(map[int]int, 1000)
```

Из-за эвакуации данных и нельзя взять указатель на элемент мапы:
```go
m := make(map[int]int, 100)  
a := &m[1]

invalid operation: cannot take address of m[1] (map index expression of type int)
```
Если мы берем ссылку на какой-то бакет, то при эвакуации данных этот ссылка на этот бакет больше не будет актуальной, потому что данные будут лежать уже в другом бакете и в другой, соответственно, ячейке памяти.
### Краткое содержание

**map** — это "ссылочный" тип данных, предназначенный для хранения пар ключ-значение. В других языках эту структуру так же называют: хэш-таблица, словарь, ассоциативный массив. Запись, чтение и удаление элементов происходят в основном за O(1). ^sum-map
