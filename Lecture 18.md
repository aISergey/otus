<h2 class="learning-near__header">Домашнее задание</h2>
          
<div class="text text_p-small text_default learning-markdown js-learning-markdown"><p>Триггеры, поддержка заполнения витрин</p></div>

<div class="text text_p-small text_default text_bold">Цель:</div>
<div class="text text_p-small text_default learning-markdown js-learning-markdown"><p> Создать триггер для поддержки витрины в актуальном состоянии.</p></div>
<br>
<div class="text text_p-small text_default text_bold">Описание/Пошаговая инструкция выполнения домашнего задания:</div> 

<div class="text text_p-small text_default learning-markdown js-learning-markdown">
  <p><br>Скрипт и развернутое описание задачи – в ЛК (файл hw_triggers.sql) или по ссылке:
    <a target="_blank" href="https://disk.yandex.ru/d/l70AvknAepIJXQ" title="https://disk.yandex.ru/d/l70AvknAepIJXQ">https://disk.yandex.ru/d/l70AvknAepIJXQ</a>
  </p>
  <p><br>В БД создана структура, описывающая товары (таблица goods) и продажи (таблица sales).</p>
  <p><br>Есть запрос для генерации отчета – сумма продаж по каждому товару.</p>
  <p><br>БД была денормализована, создана таблица (витрина), структура которой повторяет структуру отчета.</p>
  <p><br>Создать триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину)</p>
  <p><br>Подсказка: не забыть, что кроме INSERT есть еще UPDATE и DELETE</p>
  <hr>
    <p>Задание со звездочкой* </p>
    <p>Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?<br>Подсказка: В реальной жизни возможны изменения цен.</p>
</div>
