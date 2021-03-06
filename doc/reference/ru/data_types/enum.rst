Enum
----

Enum8 или Enum16. Представляет собой конечное множество строковых значений, сохраняемых более эффективно, чем это делает тип данных ``String``. 

Пример:
.. code-block:: text

  Enum8('hello' = 1, 'world' = 2)
  
- тип данных с двумя возможными значениями - 'hello' и 'world'.

Для каждого из значений прописывается число в диапазоне ``-128 .. 127`` для ``Enum8`` или в диапазоне ``-32768 .. 32767`` для ``Enum16``. Все строки должны быть разными, числа - тоже. Разрешена пустая строка. При указании такого типа (в определении таблицы), числа могут идти не подряд и в произвольном порядке. При этом, порядок не имеет значения.

В оперативке столбец такого типа представлен так же, как ``Int8`` или ``Int16`` соответствующими числовыми значениями.
При чтении в текстовом виде, парсит значение как строку и ищет соответствующую строку из множества значений Enum-а. Если не находит - кидается исключение.
При записи в текстовом виде, записывает значение как соответствующую строку. Если в данных столбца есть мусор - числа не из допустимого множества, то кидается исключение. При чтении и записи в бинарном виде, оно осуществляется так же, как для типов данных Int8, Int16.
Неявное значение по умолчанию - это значение с минимальным номером.

При ``ORDER BY``, ``GROUP BY``, ``IN``, ``DISTINCT`` и т. п., Enum-ы ведут себя так же, как соответствующие числа. Например, при ORDER BY они сортируются по числовым значениям. Функции сравнения на равенство и сравнения на отношение порядка двух Enum-ов работают с Enum-ами так же, как с числами.

Сравнивать Enum с числом нельзя. Можно сравнивать Enum с константной строкой - при этом, для строки ищется соответствующее значение Enum-а; если не находится - кидается исключение. Поддерживается оператор IN, где слева стоит Enum, а справа - множество строк. В этом случае, строки рассматриваются как значения соответствующего Enum-а.

Большинство операций с числами и со строками не имеет смысла и не работают для Enum-ов: например, к Enum-у нельзя прибавить число.
Для Enum-а естественным образом определяется функция ``toString``, которая возвращает его строковое значение.

Также для Enum-а определяются функции ``toT``, где T - числовой тип. При совпадении T с типом столбца Enum-а, преобразование работает бесплатно.
При ALTER, есть возможность бесплатно изменить тип Enum-а, если меняется только множество значений. При этом, можно добавлять новые значения; можно удалять старые значения (это безопасно только если они ни разу не использовались, так как это не проверяется). В качестве "защиты от дурака", нельзя менять числовые значения у имеющихся строк - в этом случае, кидается исключение.

При ALTER, есть возможность поменять Enum8 на Enum16 и обратно - так же, как можно поменять Int8 на Int16.
