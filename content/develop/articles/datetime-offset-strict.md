+++
title="DateTimeOffset(strict)"
description="Немного странные попытки победить DateTime"
+++
<a href="https://habr.com/ru/post/438946/">Оригинал на хабре</a>

Сегодня утром мой приятель @kirillkos столкнулся с проблемой.

## Проблемный код
Вот его код:
```cs
class Event {
   public string Message {get;set;}
   public DateTime EventTime {get;set;}
}

interface IEventProvider {
   IEnumerable<Event> GetEvents();
}
```

И дальше много-много реализаций `IEventProvider`, достающие данные из разных таблиц и баз. 

**Проблема**: во всех этих базах все в разных временных зонах. Соответственно, при попытке вывести события на UI все ужасно перепутано.

Слава Хейлсбергу, у нас есть типы, пусть они спасут нас!

## Попытка 1

```cs
class Event {
   public string Message {get;set;}
   public DateTimeOffset EventTime {get;set; }
}
```

`DateTimeOffset` замечательный тип, он хранит информацию о смещении относительно UTC. Он прекрасно поддерживается MS SQL и Entity Framework (а в версии 6.3 будет поддерживаться [еще лучше](https://github.com/aspnet/EntityFramework6/pull/429)). У нас в code style он обязательный для всего нового кода.

Теперь мы можем собрать информацию с этих самых provider и консистентно, полагаясь на типы, вывести все на UI. Победа!

**Проблема**: `DateTimeOffset` умеет _неявно_ преобразовываться из `DateTime`.
Следующий код прекрасно скомпилируется:
```cs
class Event {
   public string Message {get;set;}
   public DateTimeOffset EventTime {get;set; }
}

IEnumerable<Event> GetEvents() 
{
   return new[] {
     new Event() {EventTime = DateTime.Now, Message = "Hello from unknown time!"},
   };
}
```

Это потому, что у `DateTimeOffset` определен оператор неявного приведения типов:
```cs
// Local and Unspecified are both treated as Local
public static implicit operator DateTimeOffset (DateTime dateTime);
```
Это совсем не то, что нам нужно. Мы-то хотели, чтобы программист при написании кода был **вынужден** задуматься: «а в какой собственной временной зоне случилось это событие? Откуда взять зону?». Часто совсем из других полей, иногда из связанных таблиц. А тут совершить ошибку *не задумавшись* очень легко.

Проклятые неявные преобразования!

## Попытка 2

С тех пор, как я услышал про ~~молоток~~ [статические анализаторы](https://habr.com/ru/post/335792/), мне все кажется ~~гвоздями~~ подходящими случаями для них. Нам надо написать статический анализатор, который запрещает это неявное преобразование, и объясняет почему... Выглядит, как многовато работы. Да и вообще, это работа компилятора, проверять типы. Пока отложим эту идею, как многословную.

## Попытка 3

Вот если мы были бы в мире F#, сказал @kirillkos.
Мы бы тогда:
```fs
type DateTimeOffsetStrict = Value of DateTimeOffset
```
И дальше <s>не придумал импровизируй</s> какая-то магия нас спасла бы. Жаль, что у нас в конторе не пишут на F#, да и мы с @kirillkos его толком не знаем :-)

## Попытка 4

Неужели что-то такое нельзя сделать на C#? Можно, но замучаешься преобразовывать туда-сюда. Стоп, но ведь мы только что видели, как можно сделать неявные преобразования!

```cs
/// <summary>
/// Same as <see cref="DateTimeOffset"/>
/// but w/o implicit conversion from <see cref="DateTime"/>
/// </summary>
public readonly struct DateTimeOffsetStrict
{
  private DateTimeOffset Internal { get; }
  private DateTimeOffsetStrict(DateTimeOffset @internal)
  {
    Internal = @internal;
  }
  
 public static implicit operator DateTimeOffsetStrict(DateTimeOffset dto) 
   => new DateTimeOffsetStrict(dto);

 public static implicit operator DateTimeOffset(DateTimeOffsetStrict strict) 
   => strict.Internal;
}
```

Самое интересное в этом типе, что он неявно преобразуется туда-сюда из `DateTimeOffset`, а вот попытка неявно преобразовать его из `DateTime` вызовет ошибку компиляции, преобразования из DateTime возможны только явные. Компилятор не может вызвать «цепочку» неявных преобразований, если они определены в нашем коде, это ему запрещает стандарт ([цитата на SO](https://stackoverflow.com/questions/6001854/chaining-implicit-operators-in-generic-c-sharp-classes)). То есть, вот так работает:

```cs
class Event {
   public string Message {get;set;}
   public DateTimeOffsetStrict EventTime {get;set; }
}

IEnumerable<Event> GetEvents() 
{
   return new[] {
     new Event() {EventTime = DateTimeOffset.Now, Message = "Hello from unknown time!"},
   };
}
```

а вот так нет:
```cs
IEnumerable<Event> GetEvents() 
{
   return new[] {
     new Event() {EventTime = DateTime.Now, Message = "Hello from unknown time!"},
   };
}
```

Что нам и требовалось!

## Итог

Пока не знаем, будем ли внедрять. Только всех приучили к DateTimeOffset, и теперь его заменять на *наш* тип — стремновато. Да и наверняка всплывут проблемы на уровне EF, ASP.NET parameter binding и еще в тысяче мест. Но самое решение кажется мне интересным. Аналогичные трюки я использовал, чтобы следить за безопасностью пользовательского ввода — делал тип `UnsafeHtml`, который неявно преобразуется *из* строки, а вот обратно его преобразовать в строку или `IHtmlString` можно только путем вызова sanitizer.