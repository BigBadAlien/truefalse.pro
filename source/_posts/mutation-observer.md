---
title: Опции MutationObserver в правильном порядке
date: 2016-07-15 12:22:49
tags: WebAPI
---

Дополнение к [MDN/Web APIs/MutationObserver](https://developer.mozilla.org/en/docs/Web/API/MutationObserver) о событиях изменения DOM с различными конфигурациями наблюдателя.

#### Элементарный пример
Наблюдаются все элементы и атрибуты:
``` html
<div id="root" data-custom-attribute="Root attribute">
  Root text
  <ul class="parent" data-custom-attribute="Parent attribute">
    <li class="child" data-custom-attribute="Child attribute" contenteditable="true">Child text</li>
    <input type="text">
  </ul>
</div>
```

``` js
var observer = new MutationObserver(function(mutations) {
  mutations.forEach(function(mutation) {
    console.log(mutation);
  });
});

$(function() {
  var root = document.getElementById("root");

  observer.observe(root, {
    childList: true,
    characterData: true,
    characterDataOldValue: true,
    attributes: true,
    attributeOldValue: true,
    attributeFilter: ['data-new-attr'],
    subtree: true
  });
});

```
### Опции в правильном порядке
#### childList: true
Срабатывает в случаях:
``` js
document.querySelector('.parent').remove();
document.querySelector('#root').removeChild(document.querySelector('.parent'))
jQuery('#root').append('<div>Appended</div>');
jQuery('.parent').detach();
```

Неожиданный результат:
``` js
jQuery('#root').text('Text');
```
Событие происходит дважды – оператор jQuery.text работает таким бразом, что текстовая нода сначала удаляется, после чего на её место вставляется новая. Поведение наблюдатся именно с опцией childList, при этом с одинокой опцией characterData от подобной замены текста событие не происходит вовсе.

#### characterData: true
Срабатывает при изменении innerText, innerHTML, пользовательском редактировании элемента с атрибутом contenteditable.
Причём если innerText или innerHTML присваивается элементу с вложенными элементами, то событие не генерируется, вместо этого срабатывает опция childList потому, что происходит операция замены вложенных элементов на текстовый блок, а не изменение текстового блока.
Опция не отслеживает измененинияс деланные функцией jQuery.text(), потому что оператор работает методом удаления текстовой ноды и вставки на её место новой, что находится в области отслеживания childList. Вследствие этого нет никакого способа отслеживать только вставку и удаление текстовых нод.
Так же наблюдатель characterData не реагирует на изменение поля value элемента типа input.

##### characterDataOldValue: true
Используется только вместе с characterData. В объекте MutationRecord передаётся предыдущее значение текстовой ноды в поле oldValue.

#### attributes: true
Отслеживает изменения атрибутов. Событие происходит в случаях:
``` js
document.querySelector('#root').setAttribute('data-new-attr', 'Atribute added by Element.setAttribute');
$('#root').attr('data-new-attr', 'Atribute added by jQuery.attr');
```
##### attributeOldValue: true
Работает только с опцией attributes. В объекте MutationRecord передаётся предыдущее значение атрибута в поле oldValue.

##### attributeFilter: ['data-new-attr']
Работает только с опцией attributes. Массив - список атрибутов, которые необходимо отслеживать. Перечисленные в списке атрибуты будут проигнорированы наблюдателем.

#### subtree: true
Эта опция увеличивает область наблюдения для всех перечисленных опций до всех вложенных элементов.
С "subtree: true" будут наблюдаться изменения во всём дереве элементов.
Область наблюдения без subtree:
<figure class="highlight html"><table><tbody><tr><td class="gutter"><pre><div class="line">1</div><div class="line">2</div><div class="line">3</div><div class="line">4</div><div class="line">5</div><div class="line">6</div><div class="line">7</div></pre></td><td class="code"><pre><div class="line"><span class="tag"><span class="disabled">&lt;</span><span class="name"><span class="disabled">div</span></span> <span class="attr">id</span>=<span class="string">“root”</span> <span class="attr">data-custom-attribute</span>=<span class="string">“Root attribute”</span><span class="disabled">&gt;</span></span></div><div class="line">  Rooot text</div><div class="line">  <span class="tag">&lt;<span class="name">ul</span><span class="disabled"> <span class="attr">class</span>=<span class="string">“parent”</span> <span class="attr">data-custom-attribute</span>=<span class="string">“Parent attribute”</span></span>&gt;</span></div><div class="line"><span class="disabled">    <span class="tag">&lt;<span class="name">li</span> <span class="attr">class</span>=<span class="string">“child”</span> <span class="attr">data-custom-attribute</span>=<span class="string">“Child attribute”</span> <span class="attr">contenteditable</span>=<span class="string">“true”</span>&gt;</span>Child text<span class="tag">&lt;/<span class="name">li</span>&gt;</span></span></div><div class="line"><span class="disabled">    <span class="tag">&lt;<span class="name">input</span> <span class="attr">type</span>=<span class="string">“text”</span>&gt;</span></span></div><div class="line">  <span class="tag">&lt;/<span class="name">ul</span>&gt;</span></div><div class="line"><span class="tag">&lt;/<span class="name">div</span>&gt;</span></div></pre></td></tr></tbody></table></figure>

#### Метод takeRecords
Метод возвращает изменения, которые накопились в очереди наблюдателя. Работает синхронно, и нужен для того, чтобы получить изменения не в коллбэке, а не выходя в глобальную область видимости и не передавая поток выполнения другой функции.
В следующем случае takeRecords вернёт пустой массив:
``` js
var observer = new MutationObserver(function(mutations) {
  mutations.forEach(function(mutation) {
    console.log('From observe', mutation);
  });
});

$(function() {
  observer.observe(root, {
    childList: true
  });

  document.querySelector('.parent').remove();

  setTimeout(function() {
    var mutations = observer.takeRecords();
    console.log('From takeRecords', mutations);
  }, 0);
});
```

А в этом takeRecords вернёт изменение удаления и управление в коллбэк передано не будет:
``` js
var observer = new MutationObserver(function(mutations) {
  mutations.forEach(function(mutation) {
    console.log('From observe', mutation);
  });
});

$(function() {
  observer.observe(root, {
    childList: true
  });

  document.querySelector('.parent').remove();

  var mutations = observer.takeRecords();
  console.log('From takeRecords', mutations);
});
```
