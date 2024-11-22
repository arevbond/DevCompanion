## Определение
**Kafka** - гибрид распределенной базы данных и брокера сообщений с возможностью горизонтального масштабирования.
## Как устроен Apache Kafka

<details> <summary>TL;DR</summary> Основы кластера Kafka - это продюсеры, брокеры и консьюмеры. Продюсер пишет сообщение в лог брокера, а консьюмер его читает. </details>

**Лог** - упорядоченный поток событий во времени.
![[Pasted image 20241114111528.png]]

**Продюсеры** - приложения, которые записывают события в кластер Kafka.
![[Pasted image 20241114111648.png]]

**Брокеры** - система, состоящая из серверов, объединенных в кластеры, которая преобразует сообщение от источника данных (продюсера) в сообщение принимающей стороне (консьюмера).
![[Pasted image 20241114111913.png]]

**Консьюмеры** - приложения, которые читают данные,  записанные на локальные диски брокеров.

![[Pasted image 20241114112142.png]]
## Архитектура Kafka
<details> <summary>TL;DR</summary> Кластер Kafka позволяет изолировать консьюмера и продюсера друг от друга. </details>

![[Pasted image 20241114112715.png]]
*С версии 2.8(или 3.4?) необходимость в использовании ZooKeeper отпала: для арбитража появился собственный протокол KRaft. Он решает те же задачи, но на уровне брокеров.*

**Zookeeper** - выделенный кластер серверов для образования  кворума-согласия и поддержки внутренних процессов Kafka.
_Кворум-согласия_ — это метод достижения согласия в распределённых системах, который определяет минимальное число узлов (серверов или процессов), участвующих в системе, которые должны поддержать или подтвердить изменение данных, чтобы это изменение считалось действительным

**Функции Zookeeper**:
- Управление метаданными брокеров
- Контроль лидера партиции
- Отслеживание и балансировка консьюмеров (в последних версия перенесено в Kafka)
- Обеспечение распредленного блокирования и согласованности
- Управление топиками и конфигурацией кластеров
- Обнаружение сбоев и восстановление кластера Kafka
### Устройство брокеров

<details>
<summary>TLDR</summary>
Топики в Kafka разделены на <strong>партиции</strong>. Увеличение партиций увеличивает параллелизм чтения и записи. Партиция находится на одном или нескольких брокерах, что позволяет кластеру масштабироваться.
Партиции хранятся на локальных дисках брокеров и представлены набором лог-файлов - <strong>сегментов</strong>. Запись в них идёт в конец, а уже сохранённые события неизменны.
Каждое сообщение в таком логе определяется порядковым номером - <strong>оффсетом</strong>. Этот номер монотонно увеличивается при записи для каждой партиции.
Лог-файлы на диске устаревают по времени или размеру. Настроить это можно глобально или индивидуально в каждом топике.
Для отказоустойчивости, партиции могут реплицироваться. Число реплик или <strong>фактор репликации</strong> настраивается как глобально по умолчанию, так и отдельно в каждом топике.
Реплики партиций могут быть <strong>лидерами</strong> или <strong>фолловерами</strong>. Традиционно консьюмеры и продьюсеры работают с лидерами, а фолловеры только догоняют лидера.
</details>

**Топик** - это *логическое* разделение категорий сообщений на группы.
![[Pasted image 20241114133433.png]]

Топик удобно представлять как лог - сообщения пишутся в конец и не разрушают при этом цепочку старых сообщений.
- Один продюсер может писать в один или несколько топиков.
- Один консьюмер может читать один или несколько топиков.
- В один топик могут писать один или более продюсеров.
- Из одного топика могут читать один или более консьюмеров.

**Партиции** - разделение топика на части (разделы). Каждый топик состоит из одной или более партиций, каждая из которых может быть размещена на разных брокерах.
![[Pasted image 20241114134039.png]]
Формально партиция и есть строго упорядоченный лог сообщений. При этом сам топик в целом не имеет никакого порядка, но порядок сообщений всегда есть в одной из его партиций.

**Сегменты** - физическое представление партиций на диске.
![[Pasted image 20241114134615.png]]

Сегменты тоже удобно представить как лог-файл. Фактически это очередь FIFO (Firs-In-First-Out).
![[Pasted image 20241114135009.png]]
Семантически и физически сообщения внутри сегмента не могут быть удалены, они иммутабельны. Можно лишь указать, как долго Kafka-брокер будет хранить события через настройку политики устаревания данных или **Retention Policy**.

**Лаг** - расстояние между конечный оффсетом и текущим оффсетом консьюмера.
![[Pasted image 20241114135344.png]]

**События** (данные) - это пара ключ-значение, где ключ опционален и может повторяться. Значение может быть любым - число, строка или любой объект, который можно серилиазовать и хранить (JSON, Protobuf, и т.д.). *Заголовки* - как в [[Основы HTTP|HTTP]]-протоколе.
![[Pasted image 20241114140033.png]]
###### Устаревание данных
Когда сегмент достигает своего предела - он закрывается и вместо него открывается новый. **Активный сегмент** - в который сейчас записываются данные, по сути это файл, открытый процессом брокера. **Закрытые** сегменты - в которых больше нет записи.
![[Pasted image 20241114140400.png]]

###### Репликация данных
У каждой партиции есть настраиваемое число реплик. Один **лидер**, остальные **фолловеры**. 
![[Pasted image 20241114140829.png]]
![[Pasted image 20241114140905.png]]
С версии 2.4 Kafka поддерживает чтение консьюмера из фолловера, основываясь на их взаимном положении. Однако из-за асинхронной работы репликаций, от фолловеров можно получить менее актуальные данные, чем они есть в лидерской позиции.

Роли лидеров и фолловеров не статичны. Kafka автоматически выбирает роли для партиций в кластере. (Например при сбое одного брокера)
![[Pasted image 20241114141302.png]]

### Продюсеры
<details>
<summary>TLDR</summary>
Продюсеры самостоятельно партицируют данные в топиках и сами определяют алгоритм партицирования (round-robin,  hash-based). Важно помнить, что очередность сообщений гарантируется только для одной партиции.
Продюсер сам выбирает размер батча и число ретраев при отправке сообщений. Протокол Kafka предоставляет гарантии доставки всех трёх семантик: at-most once, at-least once и exactly once.
У exactly once есть цена. Для надёжной записи вам необходимо использовать подтверждение как от лидера, так и от реплик, включить идемпотентность и использовать транзакционный API. Всё это негативно влияет на время записи.
Не забывайте, что сломаться в пути может что угодно: например, просесть сеть или сломаться сам брокер. Переходящие процессы в кластере, как выбор лидера, редкость, но это случается, и клиенты должны уметь их грамотно обрабатывать.
Если вы хотите писать в Kafka надёжно, указывайте при создании топика min.insync.replicas меньше, чем общее количество реплик. В противном случае, лишившись брокера в случай аварии вы рискуете вовсе ничего не записать, т.к. не дождётесь подтверждения записи.
Если вы указываете acks=all то включайте и enable.idempotence. Накладных расходов на идемпотентность нет.
</details>

###### Балансировка и партицирование
![[Pasted image 20241114142930.png]]
Программа-продюсер может указать ключ сообщений, сообщений с одинаковым ключом всегда попадают в одну партицию. 
Если программа-продюсер не указывает ключ, то стратегия партицирования по умолчанию называется **round-robin** - сообщения будут попадать в партиции по очереди. Эта стратегия хорошо работает где важна не очерёдность событий, а равномерное распределение сообщений между партициями.
Также существуют другие алгоритмы. Вся логика реализации партицирования данных реализуется на стороне продюсера.
##### Дизайн продюсера
Типичная программа-продюсер работает так: пэйлоад упаковывается в структуру с указанием топика, партиции и ключа партицирования. Далее пэйлоад сериализуется в подходящий формат - JSON, Protobuf. Avro или ваш собственный формат с поддержкой схем. Затем сообщению назначается партиция согласно передаваемому ключу и выбранному алгоритму. После этого структуры группируются в пачки выбранных размеров и пересылаются брокеру Kafka для сохранения.
![[Pasted image 20241114143620.png]]
Если продюсер не смог записать сообщение, он может попытаться отправить сообщение повторно - и так по кругу.
##### Надёжность доставки
Продюсер сам определяет надёжность доставки сообщения до Kafka с помощью параметра **acks**.
![[Pasted image 20241114144807.png]]
Число реплик в которых должны сохраниться данные для параметра **all** устанавливает настройка **min.insync.replicas**. Частая ошибка при конфигурации топика устанавливать min.insync.replicas - по числу реплик. Тогда при выходе из строя брокера продюсер больше не сможет записывать сообщения в кластер, по скольку не дождётся подтверждения. Лучше предусмотрительно устанавливать min.insync.replicas на единицу меньше числа реплик.
##### Идемпотентные продюсеры
Даже с выбором **acks=all** возможны дубликаты сообщений.
![[Pasted image 20241114145534.png]]
Эта проблема решается в Kafka благодаря транзакционному API и использованию идемпотентности. Опция **enable.idempotence** - включает идемпотентность. Так каждому сообщению будет проставлен идентификатор продюсера или PID. и монотонно увеличивающийся sequence number. За счёт этого сообщения-дубликаты от одного продюсера с одинаковым PID будут отброшены на стороне брокера. Так вы добьётесь exactly once при записи в брокер. но запись будет идти дольше.
### Консьюмеры
<details>
<summary>TLDR</summary>
Партиции в консьюмер-группах распределяет автоматически Group Coordinator при помощи Group leader — первого участника в группе. Каждый консьюмер в группе может читать одну и более партиций разных топиков. Если консьюмеру не достанется партиции, то он будет бездействовать, что мешает масштабированию.
Основное преимущество консьюмер-группы перед обычным консьюмером состоит в хранении оффсета партиций на стороне брокера. Это позволяет консьюмерам прерывать работу, а после возобновлять её с того же места, где они окончили чтение.
Для проверки живости консьюмеры отправляют брокеру Heartbeat-сообщение. Если консьюмер не успел отправить его, то может покинуть группу сам, либо брокер, который не получил подтверждение, сам удалит консьюмера из группы, что запустит ребалансировку.
Любая смена композиции партиций в топиках и участников в группе запускает ребалансировку. Это болезненный процесс для консьюмеров. В этот момент все консьюмеры остановят чтение и не начнут его до полной синхронизации и стабилизации группы. Есть различные алгоритмы ребалансировки, которые позволяют смягчить процесс, но по умолчанию это Stop-The-World.
В новом консьюмере важно правильно выбрать политику оффсета. Иногда читать с начала не нужно и достаточно «перемотать» оффсет в конец, чтобы сразу получать только новые события.
Наконец, два и более консьюмера в группе не могут читать из одной и той же партиции. Чтобы не оказаться в ситуации, когда вам некуда масштабироваться при чтении, заранее установите достаточное число партиций.
</details>

##### Дизайн консьюмера
![[Pasted image 20241114151146.png]]
Программа-консьюмер поллит новые записи из партиций брокеров с помощью некоторого таймера. Десериализует полученные сообщения в батчах и следом обрабатывает. 
В конце чтения консьюмер может закоммитить оффсет - запомнить позицию считанных записей и продолжить чтение новой порции данных. 
Оффсет можно и не коммитить вовсе. Таким образом, если консьюмер прочитал сообщение, обработал, а зачем аварийно завершился, не закоммитив оффсет, то следующий консьюмер при следующем подключении вновь прочитает ту же порцию данных.
##### Консьюмер группы
![[Pasted image 20241114151733.png]]
Kafka сохраняет на своей стороне текущий оффсет по каждой партиции топиков, которые входят в состав консьюмер-групп. При подключении или отключении консьюмеров от группы, чтение продолжится с последней сохранённой позиции.
##### Ребалансировка консьюмер-групп
![[rebalancing.gif]]

