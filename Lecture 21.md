<div class="learning-near__item">
  <h2 class="learning-near__header">Домашнее задание</h2>
  <div class="text text_p-small text_default learning-markdown js-learning-markdown"><p>Репликация</p></div>
  <div class="text text_p-small text_default text_bold">Цель:</div>
  <div class="text text_p-small text_default learning-markdown js-learning-markdown">
    <ul>
      <li>реализовать свой миникластер на 3 ВМ.</li>
    </ul>
  </div>
  <br>

  <div class="text text_p-small text_default text_bold">Описание/Пошаговая инструкция выполнения домашнего задания:</div> 
  <div class="text text_p-small text_default learning-markdown js-learning-markdown">
    <ol>
      <li>На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение. </li>
      <li>Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2. </li>
      <li>На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение. </li>
      <li>Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1. </li>
      <li>3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ). </li>
    </ol>
    <p><br>ДЗ сдается в виде миниотчета на гитхабе с описанием шагов и с какими проблемами столкнулись.</p>
    <p><br>* реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. Написать с какими проблемами столкнулись.</p>
  </div>
  <br>
  
  <div class="text text_p-small text_default text_bold">Критерии оценки:</div>
  <div class="text text_p-small text_default learning-markdown js-learning-markdown"><p>Выполнение ДЗ: 10 баллов</p>
    <ul>
      <li>5 баллов за задание со *<br>плюс 2 балл за красивое решение<br>минус 2 балл за рабочее решение, и недостатки указанные преподавателем не устранены</li>
    </ul>
  </div>
</div>
