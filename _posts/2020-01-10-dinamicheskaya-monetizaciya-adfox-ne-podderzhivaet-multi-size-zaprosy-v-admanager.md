---
title: "Динамическая монетизация ADFOX не поддерживает multi-size запросы в AdManager"
excerpt_separator: "<!--excerpt_separator-->"
categories:
  - adops
tags:
  - adfox
  - yandex
  - admanager
  - gpt
header:
  teaser: /assets/images/posts_teasers/dinamicheskaya-monetizaciya-adfox-ne-podderzhivaet-multi-size-zaprosy-v-admanager.webp

---

**Из-за этого ваш доход может быть меньше. Но есть одно хитрое решение.**
<!--excerpt_separator-->

**Дисклеймер.** Я не сотрудник ADFOX, Яндекса и Google, и не могу гарантировать абсолютную точность дальнейших утверждений. Все выводы сделаны на основании открытых источников и собственных исследований.
{: .notice}

# Как ADFOX направляет запросы в AdManager

Как именно ADFOX направляет рекламные запросы в AdManager не описано в документации ADFOX и уж тем более в документации AdManager. Но, пожалуй, есть только один способ — библиотека [Google Publisher Tag](https://developers.google.com/publisher-tag/guides/get-started).

Скорее всего, вызов осуществляется для каждой площадки ADFOX, к которой привязан набор Динамической монетизации. Вызов делает сам ADFOX, поэтому вам не нужно специально размечать сайт с помощью GPT. Но всё равно полезно понимать, как это работает внутри.

Пример вызова ad unit'a в AdManager выглядит так:
```javascript
<head>
 <meta charset="utf-8">
 <title>Hello GPT</title>
 <script async src="https://securepubads.g.doubleclick.net/tag/js/gpt.js"></script>
 <script>
   window.googletag = window.googletag || {cmd: []};
   googletag.cmd.push(function() {
     googletag
         .defineSlot(
             'ad-unit-path', [width, height], 'div-id')
         .addService(googletag.pubads());
     googletag.enableServices();
   });
 </script>
</head>
```

# Что такое и в чём польза `multi-size`

Допустим, на сайте имеется рекламное место максимальным размером `300x600`. Появляются опции: выводить в нём креативы строго `300x600` или же добавить меньшие размеры. Давайте разберемся на примерах.

## Fixed-size 

В простейшем случае рекламное место с одним фиксированным размером может быть определено следующим кодом GPT:

```javascript
googletag
    .defineSlot(
    	'/1234567/your-ad-slot-name',
    	[300, 600],
    	'div-gpt-fixed-size-ad'
    )
    .addService(googletag.pubads());
```

Для таких `fixed-size` мест ротатор рекламы запрашивает и получает от монетизаторов креатив размером `300x600`.

Если ротатор вдруг пришлет креатив другого размера, то он будет [автоматически отфильтрован AdManager](https://support.google.com/admanager/answer/1143651):

![Креативы, которые не соответствуют размеру запроса объявления, отфильтровываются.](\assets\images\posts_images\2020-01-10-dinamicheskaya-monetizaciya-adfox-ne-podderzhivaet-multi-size-zaprosy-v-admanager\kreativy-kotorye-ne-sootvetstvuyut-razmeru-zaprosa-obyavleniya-otfiltrovyvayutsya.webp)

**Минусы:** если мало рекламодателей с таким размером креатива, то давление на аукцион будет меньше, соответственно и доход может быть меньше.

**Плюсы:** вёрстка рекламных мест не дёргается, сайт выглядит более органично.

## Multi-size

Код GPT `multi-size` отличается от `fixed-size` только указанием словаря допустимых размеров креатива:
```javascript
googletag
    .defineSlot(
    	'/1234567/your-ad-slot-name',
    	[[300, 250], [160, 600] [240, 400], [240, 600], [300, 600]],
    	'div-gpt-multi-size-ad'
    )
    .addService(googletag.pubads());
```

 Для таких `multi-size` мест ротатор запрашивает все указанные в настройках размеры, вписывающиеся в `300x600`. Например: `300x250`, `160x600`, `240x400`, `240x600`, `300x600`.

В этом случае, в AdManager должны быть настроены размеры для вызываемого из ADFOX ad unit'a:
![Размеры ad unit в AdManager](\assets\images\posts_images\2020-01-10-dinamicheskaya-monetizaciya-adfox-ne-podderzhivaet-multi-size-zaprosy-v-admanager\admanager-ad-unit-sizes.webp)

Таким образом, с применением `multi-size` доход может быть больше, так как появлется аукцион среди размеров креативов.

Не зря же РСЯ предлагает настроить `multi-size` при создании блока:
![РСЯ предлагает настроить multi-size при создании блока](\assets\images\posts_images\2020-01-10-dinamicheskaya-monetizaciya-adfox-ne-podderzhivaet-multi-size-zaprosy-v-admanager\yan-multi-size-set-up.webp)

Как вы понимаете, интеграция ADFOX и РСЯ полная и запрос `multi-size` креативов работает по умолчанию.

# Как ADFOX поддерживает `multi-size`  для запросов в AdManager

Я узнал об этом случайно. Я настроил в ADFOX **Динамическую монетизацию** (далее — «ДМ») по шаблону DFP с отправкой запросов на подбор креатива в AdManager.
Еще я настроил кампании **Внешней монетизации** (далее — «ВМ») с отправкой запросов в тот же AdManager.

Изучая отчёты AdManager, я обратил внимание, что ДМ запрашивает в AdManager креативы только одного размера, например, `300x600`.
![multi-size выключен](\assets\images\posts_images\2020-01-10-dinamicheskaya-monetizaciya-adfox-ne-podderzhivaet-multi-size-zaprosy-v-admanager\multi-size-ad-requests-disabled.webp)

Запрашивается размер, указанный в настройках набора ДМ. Здесь так и просится добавить перечень допустимых размеров:
![Настройки набора Динамической монетизации в ADOFX](\assets\images\posts_images\2020-01-10-dinamicheskaya-monetizaciya-adfox-ne-podderzhivaet-multi-size-zaprosy-v-admanager\nastrojki-nabora-dinamicheskoj-monetizacii-v-adfox.webp)

При этом, кампании ВМ спокойно запрашивают `multi-size`.
![multi-size включен](\assets\images\posts_images\2020-01-10-dinamicheskaya-monetizaciya-adfox-ne-podderzhivaet-multi-size-zaprosy-v-admanager\multi-size-ad-requests-enabled.webp)

Код вызова `multi-size` я сам указываю в коде баннера, ассоциированного с кампанией ВМ (внимание, код неполный):
```javascript
window.googletag = window.googletag || {cmd: []};
googletag.cmd.push(function() {
    googletag.defineSlot(
    	'/1234567/ad-unit-name',
    	[[160, 600], [240, 400], [300, 600], [240, 600], [300, 250]],
    	'div-gpt-ad-some-long-random-number-0'
    	).addService(googletag.pubads());
    googletag.pubads().enableSingleRequest();
    googletag.enableServices();
});
```

**Вывод.** Выходит, что в случае с ДМ имеется недостаток в реализации запросов в AdManager, не дающий возможности **заработать в Google по максимуму**, так как исключается аукцион среди размеров креативов. С другой стороны, **в РСЯ с этим проблем нет**.

Я искренне считал, что кампании типа ДМ — эволюционное развитие ВМ, впитавшее в себя все лучшее. Выходит, что не всё так просто.

И тут я задался вопросом: как это исправить?

# Можно ли для ДМ включить отправку `multi-size` запросов

Экспресс-ресёрсч по документации ничего не дал. Эта тема никак не освещена.

Далее я обратился в саппорт ADFOX. Отмечу, что ребята там грамотные, разбираются в сложных вопросах устройства ротатора. Я даже спросил, может быть есть какая-то недокументированная возможность в ядре ADFOX, чтобы включить нужный мне функционал.

Результат такой: **`multi-size` в ДМ сейчас не поддерживается**, размеры только фиксированные. 

По итогу я попросил добавить такую фичу в продукт в будущем. Это будет системное решение, которое положительно повлияет на доходы всех паблишеров, использующих ADFOX.

Я отдаю себе отчёт, что там вряд ли прислушаются к единичной просьбе. Но если мотивированных запросов будет много, то им придётся сделать это.

Поэтому, **прямо сейчас напишите** в саппорт просьбу сделать для ДМ 2.0 отправку `multi-size` запросов в AdManаger. Это ваши незаработанные деньги!
{: .notice--success}


# Запомнить
1. ADFOX размечает ваш сайт тегами GPT и отправляет запросы в AdManager за вас. Чтобы лучше понять, как всё работает, смотрите GPT.
1. Запросы Динамической монетизации отправляются в Google AdManager только `fixed-size`. Так меньше давление на аукцион. Хотя GPT позволяет отправлять `multi-size`.
1. Для запросов в  РСЯ `multi-size` включен по умолчанию.
1. В дополнение к ДМ, можно отправлять `multi-size` запросы в AdManager из кампаний Внешний монетизации. Но так придётся самостоятельно управлять ставками кампаний: вручную или посредством **API**. Это и есть «хитрое» решение, которое я применяю.