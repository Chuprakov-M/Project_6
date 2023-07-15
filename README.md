# Кластеризация изображений транспортных средств

## Оглавление  
[1. Описание проекта](README.md#1.-Описание-проекта)  
[2. Информация о данных](README.md#Информация-о-данных)  
[3. Обработка исходного набора данных](README.md#3.-Обработка-исходного-набора-данных)  
[4. Выбор оптимального количества кластеров](README.md#4.-Выбор-оптимального-количества-кластеров)  
[5. Результаты кластеризации](README.md#5.-Результаты-кластеризации)  
* [5.1 EfficientNet](README.md#5.1-EfficientNet)  
* [5.2 OSNet](README.md#5.2-OSNet)  
* [5.3 vdc_color](README.md#5.3-vdc_color)  
* [5.4 vdc_type](README.md#5.4-vdc_type)  

### 1. Описание проекта
Имеется набор из 416 314 изображений транспортных средств различных типов, цветов и снятых с разных ракурсов.

Для них получен набор дескрипторов, полученных с помощью нескольких моделей глубокого обучения (свёрточных нейронных сетей) - по четыре варианта вектора признаков дескрипторов для каждого изображения.

Необходимо проанализировать дескрипторы на предмет определения характеристик автомобиля на фотографии:
* тип автомобиля (кузова),
* ракурс снимка (вид сзади/спереди),
* цвет автомобиля,
* другие характеристики.
Также необходимо автоматизировать поиск выбросов в данных (засветы и блики на изображениях, изображения, на которых отсутствуют автомобили и т. д.).

Проверяемая гипотеза: разметку исходных данных можно эффективно провести с помощью методов кластеризации.

### 2. Информация о данных
Дескрипторы получены для каждого из изображений с помощью соответствующих нейронных сетей в формате numpy-массивов, сохранённых в файлах pickle:
* `efficientnet-b7.pickle` — дескрипторы, выделенные моделью классификации с архитектурой EfficientNet версии 7. Эта модель является свёрточной нейронной сетью, предобученной на на датасете ImageNet, в котором содержатся изображения более 1000 различных классов. Эта модель при обучении не видела датасета veriwiId. 
* `osnet.pickle` — дескрипторы, выделенные моделью OSNet, обученной для детектирования людей, животных и машин. Модель не обучалась на исходном датасете veriwiId.
* `vdc_color.pickle` — дескрипторы, выделенные моделью регрессии для определения цвета транспортных средств в формате RGB. Частично обучена на исходном датасете veriwild.
* `vdc_type.pickle` — дескрипторы, выделенные моделью классификации транспортных средств по типу на десяти классах. Частично обучена на исходном датасете veriwild.

Информация о дескрипторах представлена в таблице:

|модель         | дескрипторы |	минимум  | срелнее |максимум  |
|---------------|-------------|----------|---------|----------|
|efficientnet-b7|2560	      |-0.265245 |0.040661 |2.020044  |
|osnet	        |512          |0.000000  |0.886720 |33.502060 |
|vdc_color      |128          |-3.771573 |-0.413759|2.152340  |
|vdc_type       |512          |0.000000  |0.137769 |2.567548  |

Больше всего дескрипторов - у модели EfficientNet. Но и 128 дескрипоторов на 400+k наблюдений - доаольно много, поэтому для работы с данными нужно понизить размерность, чтобы расчеты велись за приемлемое время.

### 3. Обработка исходного набора данных

Снижение размерности проведено при помощи метода главных компонент. Для этого оперделено количество необходимых компонент для объяснения 90% дисперсии, а также в ноутбуке построены графики зависимости доли объясненной дисперсии от количества взятых главных компонент.

График EfficientNet имеет излом на уровне 75-80% объяснения дисперсии, можно бы взять ~300-350 главных компонент, соответствующих этой области. Но так как это самая большая база по количеству дескрипторов и по объему (4 Гб), надо уменьшить посильнее, поэтому бузет взять 250 главных компонент.

Для OSNet график более плавный, изломов не видно. В принципе можно взять ~150 рекомендуемых главных компонент, объясняющих 90% дисперcии.

В vdc_color есть 4 очень главных компоненты, объясняющие 50% дисперсии, а всего 12 компонент с объяснением 1+% дисперсии каждая дают 63% (см. код ниже). Возьмем 70 объясняющих 90% дисперсии компонент.

Интереснее всего ситуация в vdc_type: есть 1 компонента на 34%, 2 - 63%. а для 90% достаточно 20 компонент. Также возьмем оба (2 и 20) варианта для анализа, благо с таким количеством компонент считаться долго не должно.

Итого имеем 6 наборов главных компонент:
* 250 для EfficientNet,
* 150 для OSNet
* 4 и 70 для vdc_color
* 2 и 20 для vdc_type

### 4. Выбор оптимального количества кластеров
Модель EfficientNet:

Индекс Калински — Харабаса убывает с замедляющейся скоростью. Оптимум если и будет, то неблизко. Индекс Дэвиса — Болдина (если не считать начальные точки) стабильно находится в диапазоне 5-5.5. Скорее всего. толковой интерпретации кластеров не получить. Но посмотрим, например, 5, 10 и 20 кластеров.

Модель OSNet:

Ситуация с индексом Калински — Харабаса похожа на EfficientNet - плавное убывание. Индекс Дэвиса — Болдина в целом убывает до ~13 кластеров, далее снова стабилизируется. Вот и посмотрим, какая интерпретация будет у 13 и 15 (там статистика чуть пониже, чем у 13) кластеров.

Модель vdc_color:

Индекс Дэвиса — Болдина для 4 и 70 компонент ведет себя похожим образом - минимум достигается около 3-4 кластеров. С индексом Калински — Харабаса сложнее: для 4 компонент есть максимум на 4 кластерах, для 70 компонент значение снова монотонно убывает. Я бы посмотрел 4-5 кластеров на 4 компонентах. Ожидаю черное - белое, красно-желтое - сине-зеленое и, может быть, какое-то дополнительное типа серебристого. Интересно бы получить весь спектр по радуге (с учетом объединения голубого и синего) + черное и белое, т.е. 8. Но зеленых и желтых машин встречается очень мало.

3.1.4. Модель vdc_type
Графики в зависимости от числа компонент сильно отличаются.

Для 2 компонент нужно брать 8 кластеров по индексу Калински — Харабаса и 4 - по индексу Дэвиса — Болдина.

Для 20 компонент по индексу Калински — Харабаса снова монотонное убывание, по по индексу Дэвиса — Болдина - минимум в районе 6.

Но сами значения для индекса Дэвиса — Болдина находятся в очень узком коридоре в обоих случаях.

Можно проверить от 4 до 8 кластеров для 2 и 20 компонент. Интересно, на что способна модель при таком минимализме.

### 5. Результаты кластеризации
#### 5.1 EfficientNet

Модель EfficientNet не выделяет каких-то четких кластеров. С ростом числа кластеров наблюдентся попытка определить ракурс (прямо/под углом) и сторону (вид спереди/сзади/под углом). Забавное достижение - кластер в модели на 20 кластеров: машины, которые едут в направлениии слева сверху вправо вниз на фоне черно-желтой разметки.

Возможно, сеть научилась определять машины и отличать их от других объектов, но не сами машины между собой. Показателен кластер с желто-черной разметкой: пчел она там что ли нашла, шмелей?..

При этом входных параметров 2560 - что-то из разряда "хоспади, а разговоров-то было!"
#### 5.2 OSNet

Очень хорошая модель, с ростом количества кластеров только раскрывается.

#### 5.3 vdc_color

Делает то, что требуется исходя из названия, но не так хорошо, как хотелось бы.

#### 5.4 vdc_type

Средняя модель - типы выделяет с трудом. Возможно улучшение при росте количества кластеров, но зачем, если есть OSNet.

### 6. Итоги
Результаты кластеризации представлены в таблице:

|модель         | гл. компоненты |	количество кластеров (оптимальное) |качество      | примечание |
|---------------|----------------|-------------------------------------|--------------|------------|
|efficientnet-b7|250	         | 5, 10, 20                           |низкое        |есть кластер с разметкой, а не типом машины|
|osnet	        |150             | 10, 13, 15 (15)                     |высокое       |определяет даже цвет|
|vdc_color      |4               | 3, 5, 8 (3)                         |высокое       |в части цвета: четкое выделение темных, светлых, красного|
|vdc_color      |70              | 3, 5, 8 (3)                         |высокое       |в части цвета: четкое выделение темных, светлых, красного|
|vdc_type       |2 / 20 / 512    | 3, 5, 8                             |среднее       |большее количество кластеров не помогает улучшить разделение|
|**osnet**	    |150             | 30, 40, 50 (30)                     |очень высокое |определяет типы, ракурсы и такие цвета, какие vdc_color и не снились|

Для дальнейшего моделирования предлагается использовать результаты кластеризации дескрипторов OSNet, т.к. она очень неплохо разделяет машины на типы и цвета.

Более того, она еще и определила некачественные изображения (темное время суток, яркие фары, блики, когда не всегда можно понять тип автомобиля, не говоря уже о цвете).
(Если отделить задачу определения цвета, причем ограниченных градаций темные - разноцветные - белые, то можно взять vdc_color, тем более что там нужно совсем немного компонент, а потом ее результат объединить с типами по OSNet.)

Общее впечаление и описание кластеров от OSNet:

Выполнил деление на больше, чем 20 кластеров - 30, 40. Описал 30. 

Кластеры хоть и могут повторяться, в целом объединяемы. Можно изучать их структуру по каждому варианту количества кластеров от 20 и далее. Ну и переразмечать кластеры на основе полученных, если потребуется, все же легче, чем размечать весь массив.

Интерпретация 30 кластеров (если не уточнен тип, то классические легковые):
0. темные, вид спереди (перед камерой какие-то ветки свисают).
1. серебристые, светлые спереди - легковые и минивэны
2. черные, вид сзади
3. рыжевато-оранжевые (!!!)
4. белые, вид сзади
5. белые, вид спереди, джипы, мощные
6. черные, вид спереди, представительские, джипы
7. темные, но не черные, представительские, джипы
8. белые, вид спереди, под небольшим углом
9. яркий свет фар - выбросы
10. черные, вид под $45^\circ$
11. ярко-красные, для определения вида (ракурса) можно попытаться построить 2 кластера на нем
12. белые, во сновном без выступающего багажника или с коротким багажником, сзади
13. черные, сзади
14. синие - в основном автобусы
15. зеленые - в основном автобусы, такси (Гонконг?)
16. темные темно-синие
17. синие, голубые, 50% автобусы
18. не очень удачный кластер по типу, но вид сзади под $45^\circ$
19. светлые, белые, спереди, джипы
20. тоже не очень удачный кластер по типу, но вид спереди
21. светр=лые, вид спереди, те же камеры со свисающими ветками, как и кластер 0
22. темные, сзади, представительские (вижк кольца ауди, ягуар ягуара,
23. вид спереди под $45^\circ$, на фоне черно-желтой разметки (привет, Efficientnet)
24. светлые, белые, сзади
25. четко серебристые, сзади
26. белые, часто джипы, спереди под $45^\circ$, и снова местами полосатая разметка.
27. желтые - тут как и с рыжими, для ракурса надо подмодель. но сам факт четкого выделения очередного цвета, которое даже vdc_color не поймал - это да...
28. красные - побледнее что ли, чем другой красные кластер?
29. серые, серебристые, фургончиков много.
#   P r o j e c t _ 6  
 