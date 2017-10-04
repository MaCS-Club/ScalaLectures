[Назад](https://macs-club.github.io/ScalaLectures/index)
## Примечание 1: Отклонения от типа
_На основе перевода [статьи](http://blog.originate.com/blog/2016/08/10/cheat-codes-for-contravariance-and-covariance/)_



В Scala при наследовании типов имеющих дженерики есть три варианта _отклонений от типа_ (variance) -- ко-. контр- и инварианты. Рассмотрим их подробнее.
Пусть у нас такая схема наследования:

```
trait Life

class Bacterium extends Life

trait Animal extends Life {
  def sound: String
}

class Dog(name: String, likesFrisbees: Boolean) extends Animal {
  val sound = "bark"
}

class Cat(name: String, likesHumans: Boolean) extends Animal {
  val sound = "meow"
}

def whatSoundDoesItMake(animal: Animal): String =
  animal.sound
```

### П1.1 Коварианты
Лучший пример ковариантов -- контейнерные типы:

```
val dogs: Seq[Dog] = Seq(
  new Dog("zoe", likesFrisbees = true),
  new Dog("james vermillion borivarge III", likesFrisbees = false)
)

val cats: Seq[Cat] = Seq(
  new Cat("cheesecake", likesHumans = true),
  new Cat("charlene", likesHumans = false)
)

def whatSoundsDoTheyMake(animals: Seq[Animal]): Seq[String] =
  animals map (_.sound)
```

Метод `whatSoundsDoTheyMake` ожидает получить `Seq[Animal]`, чтобы вызвать метод `.sound` на этих животных. Мы знаем, что все представители `Animal` обладают методом `.sound`, и мы применяем его к списку `Animal`, так что абсолютно нормально передать в `whatSoundsDoTheyMake`  последоватетельности `Seq[Dog]` или `Seq[Cat]`.

```
Dog <: Animal, следовательно Seq[Dog] <: Seq[Animal]
```
Обратим внимание на то где происходит вызов метода. Он производится не внутри определения `Seq`, а внутри функции принимающей `Animal` как аргумент. Предположим теперь. что мы передаем `Seq[Life]` в `whatSoundsDoTheyMake`. Компилятор не допустит этого и напишет, в случае использования scalac: 

```
error: value sound is not a member of Life.
```

(в случае использования dotty сообщение возможно было бы более подробным)

Иначе бы мы моли бы попытаться вызвать `bacterium.sound` не смотря на отстутвие такого метода в данном классе. В динамически типизированных языках мы бы к своему несчастью смогли бы сделать подобное и получить ошибку уже в рантайме подобную:

```
TypeError: Object #<Bacterium> has no method 'sound'.
```

Заметим. что ошибка возникает не в `Seq`, а глубже. в `Animal`. Причина в том, что дженерики предоставляют гарантии другим классам и типам работающим с ними. Объявить класс ковариантом над `T` это то же самое, что сказать “если вы вызовется функции использующие меня и предоставлю экземпляр `T`, вы можете ожидать, что все необходимые методы там будут”.

Синтаксис для обозначения коварианта в коде :`[+A]`.

### П1.2 Контрварианты

Функции -- это основной пример контрвариантоы (заметим, что контвариантны они только по аргументам и ковариантны по результатам): 

```
class Dachshund(
  name: String,
  likesFrisbees: Boolean,
  val weinerness: Double
) extends Dog(name, likesFrisbees)

def soundCuteness(animal: Animal): Double =
  -4.0/animal.sound.length

def weinerosity(dachshund: Dachshund): Double =
  dachshund.weinerness * 100.0

def isDogCuteEnough(dog: Dog, f: Dog => Double): Boolean =
  f(dog) >= 0.5
```


Можем ли мы использовать `weinerosity` как аргумент в `isDogCuteEnough`? Нет потому что `isDogCuteEnough` может гарантировать, что передасть не более узкий класс чем `Dog` в функцию `f`. Если `f` будет ожидать чего более специфичного чем `isDogCuteEnough` предоставляет, она может попытаться вызвать функцию которой не обладают представители `Dog`.

Можеи ли теперь передать `soundCuteness` в `isDogCuteEnough`? Теперь да, так как если `isDogCuteEnough` передаст `Dog` в `soundCuteness`, `soundCuteness` принимает `Animal`, а значит будет вызвать только те методы которые гарантировано есть в `Dog`.

```
Dog <: Animal, следовательно Function1[Animal, Double] <: Function1[Dog, Double]
```

Функция принимающая менее специфичный аргумент может быть замещена в выражении функцией принимающей более специфичный аргумент.

Синтаксис для обозначения контрварианта в коде :`[-A]`.

### П1.3 Инварианты

По умолчанию дженерики являются инвариантам в Scala. Это означает, что они не являются ни ковариантами ни контрвариантами. Привем пример класса `Container` являющегося инваренантом. `Container[Cat]` не является `Container[Animal]`, обратное также не верно.

```
class Container[A](value: A) {
  private var _value: A = value
  def getValue: A = _value
  def setValue(value: A): Unit = {
    _value = value
  }
}
```

Кажется что как в первом случае `Container[Cat]` должен относится к `Container[Animal]`, то есть быть ковариантом но есть большая разница. Этот класс является _изменяемым_ контейнером. Предположим, что это ковариант:

```
val catContainer: Container[Cat] = new Container(Cat("Felix"))
val animalContainer: Container[Animal] = catContainer
animalContainer.setValue(Dog("Spot"))
val cat: Cat = catContainer.getValue // Oops, we'd end up with a Dog assigned to a Cat
```

К счастью компилятор остановит на раньше чем мы дойдем до этого.

### П1.4 Памятка по контр-\ковариантам

Как быстро определить может ли ваш `ParametricType[T]` быть контр-\ковариантом:

Тип может быть ковариантом если он не вызвает методы типа над которым он дженерик. 

```
Классические примеры: Seq[+A], Option[+A], Future[+T]
```

Тип может быть контвариантом если он не возвращает значения типа над которым он дженерик. 

```
Классические примеры: Function1[-T1, +R], CanBuildFrom[-From, -Elem, +To], OutputChannel[-Msg]
```


[Назад](https://macs-club.github.io/ScalaLectures/index)

