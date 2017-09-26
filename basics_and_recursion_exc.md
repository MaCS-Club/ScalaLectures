Проверка на упорядоченность массива

`def isOrdered[A](as:Array[A], ordering: (A,A)=>Boolean) :Boolean`

Каррирование

`def curry[A,B,C](f: (A,B)=>C) :A => (B=>C)`

Анкаррирование

`def uncurry[A,B,C](f: A => (B=>C)) (A,B)=>C`

Композиция

`def compose[A,B,C](f: B=>C, g:A => B) A=>C`
