[Назад](https://macs-club.github.io/ScalaLectures/index)
## Лабораторная 4: Чисто функциональное состояние

### Мотивационный пример
Предположим, что нам необходимо создать генератор псевдослучайных чисел, который будет отвечать всем требованиям функционального программирования. Для этого нам понадобится, чтобы он не держал менял свое состояние после каждого нового числа, а позволял создавать нам новый элемент с новым состоянием.

Как трейт мы можем описать это так:

```
  trait RNG {
    def nextInt: (Int, RNG) 
  }
```

Пусть мы уже создали некоторую реализцию генератора псевдослучайных целых (например такую же как в коде лабораторной). Далее нам может захотеться написать реализации для других типов внутри нашей реализации:

```
  def nonNegativeInt(rng: RNG): (Int, RNG) = {
    val (i, r) = rng.nextInt
    (if (i < 0) -(i + 1) else i, r)
  }

  def double(rng: RNG): (Double, RNG) = {
    val (i, r) = nonNegativeInt(rng)
    (i / (Int.MaxValue.toDouble + 1), r)
  }

  def boolean(rng: RNG): (Boolean, RNG) =
    rng.nextInt match { case (i,rng2) => (i%2==0,rng2) }
```

Можно заметить, что все эти генераторы имеют очень похожую сигнатуры. Мы можем обозначить алиас для типа `RNG => (A, RNG)` следующим образом:

```
  type Rand[+A] = RNG => (A, RNG)
```

Это позволяет нам удобно работать со всеми перечиленными выше функциями как с представителями одного типа. Метод `nextInt` мы тоже можем привести к подобной сигнатуре используя запись:

```
  val int: Rand[Int] = _.nextInt
```

Мы если мы добавим пустой генератор, не меняющий состояние, то сможем написать ряд полезных общих функций известных нам ранее из конструкци `List`, `Option` и `Either`

```
  def unit[A](a: A): Rand[A] =
    rng => (a, rng)

  def map[A,B](s: Rand[A])(f: A => B): Rand[B] =
    rng => {
      val (a, rng2) = s(rng)
      (f(a), rng2)
  }

  def flatMap[A,B](s: Rand[A])(f: A => Rand[B]): Rand[B] =
    rng => {
      val (a, r1) = s(rng)
      f(a)(r1) 
  }

  def map2[A,B,C](ra: Rand[A], rb: Rand[B])(f: (A, B) => C): Rand[C] =
    rng => {
      val (a, r1) = ra(rng)
      val (b, r2) = rb(r1)
      (f(a, b), r2)
    }

  def sequence[A](fs: List[Rand[A]]): Rand[List[A]] =
    fs.foldRight(unit(List[A]()))((f, acc) => map2(f, acc)(_ :: _))

```

где `unit` -- наш пустой случайный генератор.

Через подобные функции мы легко можем работать с функциональным типом возвращающим состояние, но очевидно, что генераторами случайных чиселп подобные типы не ограничены.

### Общий случай

Общее описание типа функции возвращающей состояние выглядит так:

```
case class State[S, +A](run: S => (A, S)) {
  def map[B](f: A => B): State[S, B] =
    flatMap(a => State.unit(f(a)))

  def map2[B,C](sb: State[S, B])(f: (A, B) => C): State[S, C] =
    flatMap(a => sb.map(b => f(a, b)))
  def flatMap[B](f: A => State[S, B]): State[S, B] = State(s => {
    val (a, s1) = run(s)
    f(a).run(s1)
  })
}

object State {

  def unit[S, A](a: A): State[S, A] =
    State(s => (a, s))

  def sequence[S,A](sas: List[State[S, A]]): State[S, List[A]] =
    sas.foldRight(unit[S, List[A]](List()))((f, acc) => f.map2(acc)(_ :: _))


  def modify[S](f: S => S): State[S, Unit] = for {
    s <- get 
    _ <- set(f(s)) 
  } yield ()

  def get[S]: State[S, S] = State(s => (s, s))

  def set[S](s: S): State[S, Unit] = State(_ => ((), s))
}
}
```

Где нам не знакомы только функция `modify`, и вспомогательные для неё `get` и `set`.

Этот метод позволяет нам менять состояние объекта (точнее создавать объект с новым состоянием) не возвращая результат типа `A`. Её описание активно использует `for-comprehension`, можете дополнительно попробовать представить как она будет выглядит при переписывание через `map` и `flatMap`.

**Задание:** Лабораторная находится в [гитхабе](https://github.com/MaCS-Club/ScalaExercises) в ветке `state`.
Необходимо реализовать конечный автомат (подробнее про конечные автоматы можно прочитать [здесь](https://github.com/MaCS-Club/lambda-calculus-lectures/releases/download/v0.0.1A/lambda-calculus-lectures-0.0.1A.pdf)), симулирующий работу простого конфетного автомата.

Машина имеет два типа ввода -- можно бросить в неё монету или повернуть ручку.
Она может находится в двух состояниях -- открыта или закрыта, Также она отмечает сколько в ней конфет и монет.

```
sealed trait Input
case object Coin extends Input
case object Turn extends Input
case class Machine(locked: Boolean, candies: Int, coins: Int)
```

Правила работы следующие:
* Монетка брошенная в закрытую машину открывает её, если в неё есть конфеты
* Поворот ручки на открытой машины выбрасывает из машины одну конфету и делает машину закрытой
* Поворот ручки на закрытой машине или бросание монетки в открытую не делает ничего.
* Машина, в которой не осталось конфет игнорирует все входы.

Конечный автомат должен быть описан, либо вызываться в методе класса `Machine`:

```
def simulateMachine(inputs: List[Input]): State[Machine, (Int, Int)]
```

который принимает список входов и возвращает состояние новой машины и результат -- число конфет и монет в машине.

[Назад](https://macs-club.github.io/ScalaLectures/index)
