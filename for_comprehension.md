[Назад](https://macs-club.github.io/ScalaLectures/index)
## Лабораторная 1: Списковое включение

В программировании сушествует понятие _спискового включения_ -- некой синаксической конструкции позволяющей компактно обрабатывать списки, сводя несколько базовых функций в одну конструкции. Многи функциональные языки, например Haskell или Scala обобщают это понятие на _монадическое включение_ -- синтаксическую конструкцию позволяющую кратко обрабатываеть произвольные монады (подробнее о монадах будет сказано отдельно, сейчас достаточно понимать, что это некая обертка в которой определены пустой элемент и функция `flatMap`). Перед тем как рассмотреть списковое включение, посмотрим какие вообще фунции списка нам полезно будет определить.

* Первая функция -- это `map` функция высшего порядка позволяющая применить некоторую другую функцию к каждому элементу списка. Она имеет следующую сигнатуру.

```
def map[B] (f:A=>B):List[B]
```

Пример использования:

```
List("foo","bar","foobar").map(_.length)
```
Выведет

```
List(3,3,6)
```

* Если мы хотим изменить структуру списка или использую функцию результут действия которой сам является списком, то вместо `map` нам понадобится функция `flatMap`, применяющая к каждому элементу списка функцию возвращающую новый список и конкатенирующая результаты. Она имеет следующую сигнатуру.

```
def flatMap[B] (f:A=>List[B]):List[B]
```

Пример использования:

```
List("foo","bar","foobar").flatMap(List(_,_))
```
Выведет

```
List("foo","foo","bar","bar","foobar","foobar")
```

* Более частный случай функции `flatMap` -- это функция `filter` сохраняющая в списке только те элемнты которые удовлетворяют предикату (почему её можно рассмотретькак частный случай `flatMap`?).

```
def filter (p:A=>Boolean):List[A]
```

Пример использования:

```
List(1,2,3).flatMap(_%2!=0)
```
Выведет

```
List(1,3)
```

* Если мы захотим применить к элементам списка функцию с побочным эффектом и не возвращать нкиакой результат мы используем функцию `foreach`.

```
def foreach[U] (f:A=>U):Unit
```

Пример использования:

```
List(1,2,3).foreach(println_)
```
Выведет

```
1
2
3
```

Таким образом у нас есть четыре функции позволяющие обрабатывать списки, однако для работы нам хотелось бы иметь какой-то общий единообразный интерфейс. Его предоставляет нам конструкция `for`, в случае если описанные выше функции определены, мы можем кратко записать выражение

```
xs.filter { x =>
  x % 2 == 0
}.flatMap { x =>
  ys.map { y =>
    (x, y)
  }
}
```

, как

```
for {
  x <- xs if x % 2 == 0
  y <- ys
} yield (x, y)
```

И, аналогично с `foreach`

```
xs.foreach(println_)
```

то же самое, что

```
for {x<-xs} {println(x)}
```

Самое главное, что если вы определите эти функции в своей структуре, то компилятор Scala автоматические развернет применение `for` в некоторую комбинацию четырх описанных функций. Так как вы может определить их не только над списками, `for` является монадическим включением в Scala.

**Задание:** Лабораторная находится в [гитхабе](https://github.com/MaCS-Club/ScalaExercises) в ветке `lab`. Вам нужно будет описать методы необходимые для спискового включения используя функцию `foldRight`. Обратите внимение, что так как это функции вызываемые на конкретном списке, а не из статического контекста, то они описываютя прямо в трейте `List`. Чтобы работать с текущим объектом используется ссылка на него `this`, аналогично Java.

[Назад](https://macs-club.github.io/ScalaLectures/index)