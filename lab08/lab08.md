# Практика 8. Транспортный уровень

## Реализация протокола Stop and Wait (8 баллов)
Реализуйте свой протокол надежной передачи данных типа Stop and Wait на основе ненадежного
транспортного протокола UDP. В вашем протоколе, реализованном на прикладном уровне,
отправитель отправляет пакет (frame) с данными, а затем ожидает подтверждения перед
продолжением.

**Клиент**
- отправляет пакет и ожидает подтверждение ACK от сервера в течение заданного времени (тайм-аута)
- если ACK не получен, пакет отправляется снова
- все пакеты имеют номер (0 или 1) на случай, если один из них потерян

**Сервер**
- ожидает пакеты, отправляет ACK, когда пакет получен
- отправленный ACK должен иметь тот же номер, что и полученный пакет

Вы можете использовать схему, которая была рассмотрена на лекции в рамках обсуждения
протокола rdt3.0, как пример:

<img src="images/rdt.png" width=700 />

### А. Общие требования (5 баллов)
- В качестве базового протокола используйте UDP. Поддержите имитацию 30% потери
  пакетов. Потеря может происходить в обоих направлениях (от клиента серверу и от
  сервера клиенту).
- Должен быть поддержан настраиваемый таймаут.
- Должна быть обработка ошибок (как на сервере, так и на клиенте).
- В качестве демонстрации работоспособности вашего решения передайте через свой
  протокол файл (от клиента на сервер), разбив его на несколько пакетов перед отправкой
  на стороне клиента и собрав из отдельных пакетов в единый файл на стороне сервера.
  Файл и размеры пакетов выберите самостоятельно.

Приложите скриншоты, подтверждающие работоспособность программы.

#### Демонстрация работы

Клиент
```
import java.io.File
import java.net.DatagramPacket
import java.net.DatagramSocket
import java.net.InetAddress
import kotlin.random.Random

fun main(args: Array<String>) {
    if (args.size != 1) {
        println("Usage: <filename>")
        return
    }
    sendFile(args[0])
}

fun sendFile(filename: String) {
    val serverAddress = InetAddress.getByName("127.0.0.1")
    val serverPort = 12345
    DatagramSocket().use { socket ->
        socket.soTimeout = 2000 // Тайм-аут в миллисекундах
        val file = File(filename)
        val fileSize = file.length()
        println("Sending file $filename of size $fileSize bytes")

        // Отправляем метаданные файла (имя и размер) в первом пакете
        val metadata = "$filename:$fileSize"
        sendPacket(socket, metadata.toByteArray(), serverAddress, serverPort)

        var sequenceNumber = 0
        file.inputStream().use { fis ->
            var bytesRead: Int
            do {
                val data = ByteArray(512) // Уменьшенный размер пакета
                bytesRead = fis.read(data)
                if (bytesRead > 0) {
                    val packetData = ByteArray(bytesRead + 1)
                    packetData[0] = sequenceNumber.toByte()
                    System.arraycopy(data, 0, packetData, 1, bytesRead)
                    if (Random.nextFloat() <= 0.7) { // Симуляция потери пакета
                        println("Sending packet with sequence number $sequenceNumber")
                        sendPacket(socket, packetData, serverAddress, serverPort)
                    } else {
                        println("Packet with sequence number $sequenceNumber lost")
                    }
                    sequenceNumber = 1 - sequenceNumber
                }
            } while (bytesRead > 0)

            val eofPacketData = "EOF".toByteArray()
            sendPacket(socket, eofPacketData, serverAddress, serverPort)
            println("EOF sent, file transfer completed.")
        }
        println("File sent successfully.")
    }
}

fun sendPacket(socket: DatagramSocket, data: ByteArray, address: InetAddress, port: Int): Boolean {
    try {
        val packet = DatagramPacket(data, data.size, address, port)
        socket.send(packet)
        // Ожидаем ACK
        val buffer = ByteArray(1024)
        val responsePacket = DatagramPacket(buffer, buffer.size)
        socket.receive(responsePacket)
        return String(responsePacket.data, 0, responsePacket.length).startsWith("ACK")
    } catch (e: Exception) {
        return false
    }
}
```

Лог клиента
```
Sending file file_to_send.txt of size 14663 bytes
Packet with sequence number 0 lost
Packet with sequence number 1 lost
Sending packet with sequence number 0
Sending packet with sequence number 1
Packet with sequence number 0 lost
Packet with sequence number 1 lost
Sending packet with sequence number 0
Sending packet with sequence number 1
Sending packet with sequence number 0
Sending packet with sequence number 1
Sending packet with sequence number 0
Packet with sequence number 1 lost
Sending packet with sequence number 0
Sending packet with sequence number 1
Sending packet with sequence number 0
Sending packet with sequence number 1
Sending packet with sequence number 0
Packet with sequence number 1 lost
Sending packet with sequence number 0
Sending packet with sequence number 1
Sending packet with sequence number 0
Sending packet with sequence number 1
Sending packet with sequence number 0
Packet with sequence number 1 lost
Sending packet with sequence number 0
Sending packet with sequence number 1
Packet with sequence number 0 lost
Sending packet with sequence number 1
Sending packet with sequence number 0
EOF sent, file transfer completed.
File sent successfully.
```

Скрин отправленного файла


<img src="images/screen_a.png" width=700 />

Сервер
```
import java.io.ByteArrayOutputStream
import java.io.File
import java.io.FileOutputStream
import java.net.DatagramPacket
import java.net.DatagramSocket
import kotlin.random.Random

fun main() {
    val serverPort = 12345
    val saveDirectory = "./received/"
    DatagramSocket(serverPort).use { socket ->
        println("Server is running and waiting for connections...")

        while (true) {
            val fileOutputStream = ByteArrayOutputStream()
            var fileName = ""
            var fileSize = 0L
            var isFileMetaReceived = false
            var packet: DatagramPacket

            while (true) {
                val receiveBuffer = ByteArray(1024)
                packet = DatagramPacket(receiveBuffer, receiveBuffer.size)
                socket.receive(packet)
                val data = packet.data.copyOf(packet.length)
                val receivedString = data.decodeToString().trim { it <= ' ' }

                if (receivedString == "EOF") {
                    println("EOF received, file transfer completed.")
                    break // Выход из внутреннего цикла при получении EOF
                }

                if (!isFileMetaReceived) {
                    // Обработка метаданных файла
                    val fileInfo = receivedString.split(":")
                    if (fileInfo.size == 2) {
                        fileName = fileInfo[0]
                        fileSize = fileInfo[1].toLong()
                        isFileMetaReceived = true
                        println("Receiving file $fileName of size $fileSize bytes")
                    }
                } else {
                    // Накопление полученных данных файла
                    fileOutputStream.write(data, 1, data.size - 1)
                }
            }

            // Сохранение файла
            val directory = File(saveDirectory).apply { if (!exists()) mkdirs() }
            val file = File(directory, fileName).apply { createNewFile() }
            FileOutputStream(file).use { it.write(fileOutputStream.toByteArray()) }
            println("$fileName saved successfully.")
        }
    }
}
```

Лог сервера
```
Server is running and waiting for connections...
Receiving file file_to_send.txt of size 14663 bytes
EOF received, file transfer completed.
file_to_send.txt saved successfully.
```

Скрин полученного файла

<img src="images/screen_b.png" width=700 />


### Б. Дуплексная передача (2 балла)
Поддержите возможность пересылки данных в обоих направлениях: как от клиента к серверу, так и
наоборот. 

Продемонстрируйте передачу файла от сервера клиенту.

#### Демонстрация работы
todo

### В. Контрольные суммы (1 балл)
UDP реализует механизм контрольных сумм при передаче данных. Однако предположим, что
этого нет. Реализуйте и интегрируйте в протокол свой способ проверки корректности данных
на прикладном уровне (для этого вы можете использовать результаты из следующего задания
«Контрольные суммы»).

## Контрольные суммы (2 балла)
Методы, основанные на использовании контрольных сумм, обрабатывают $d$ разрядов данных как
последовательность $k$-разрядных целых чисел.
Наиболее простой метод заключается в простом суммировании этих $k$-разрядных целых чисел и
использовании полученной суммы в качестве битов определения ошибок. Так работает алгоритм
вычисления контрольной суммы, принятый в Интернете, — байты данных группируются в $16$-
разрядные целые числа и суммируются. Затем от суммы берется обратное значение (дополнение
до $1$), которое и помещается в заголовок сегмента.

Получатель проверяет контрольную сумму, складывая все числа из данных (включая контрольную
сумму), и сравнивает результат с числом, все разряды которого равны $1$. Если хотя бы один из
разрядов результата равен $0$, это означает, что произошла ошибка.
В протоколах TCP и UDP контрольная сумма вычисляется по всем полям (включая поля заголовка и
данных).

Реализуйте функцию для подсчета контрольной суммы, а также функцию для проверки, что
данные соответствуют контрольной сумме.

**Требования**
- Функция 1 принимает на вход массив байт и возвращает контрольную сумму (число).
- Функция 2 принимает на вход массив байт и контрольную сумму и проверяет,
соответствует ли сумма переданным данным. Размер входного массива ограничен сверху
числом байтов ($= L$), однако данные могут поступать разной длины ($\le L$).

Добавьте два-три теста, покрывающих как случаи
корректной работы, так и случаи ошибки в данных (сбой битов). Вы можете не использовать
тестовые фреймворки и реализовать тестовые сценарии в консольном приложении.

## Задачи

### Задача 1 (2 балла)
Пусть $T$ (измеряется в RTT) обозначает интервал времени, который TCP-соединение тратит на
увеличение размера окна перегрузки с $\frac{W}{2}$ до $W$, где $W$ – это максимальный размер окна
перегрузки. Докажите, что $T$ – это функция от средней пропускной способности TCP.

#### Решение
todo

### Задача 2 (3 балла)
Рассмотрим задержку, полученную в фазе медленного старта TCP. Рассмотрим клиент и веб-сервер, напрямую соединенные одним каналом со скоростью передачи данных $R$.
Предположим, клиент хочет получить от сервера объект, размер которого точно равен $15 \cdot S$,
где $S$ – это максимальный размер сегмента.
Игнорируя заголовки протокола, определите время извлечения объекта (общее время
задержки), включая время на установление TCP-соединения (предполагается, что RTT - константа), если:
1. $\dfrac{4S}{R} > \dfrac{S}{R} + RTT > \dfrac{2S}{R}$
2. $\dfrac{𝑆}{𝑅} + 𝑅𝑇𝑇 > \dfrac{4𝑆}{𝑅}$
3. $\dfrac{𝑆}{𝑅} > 𝑅𝑇𝑇$

#### Решение
todo

### Задача 3 (2 балла)
Рассмотрим модификацию алгоритма управления перегрузкой протокола TCP. Вместо
аддитивного увеличения, мы можем использовать мультипликативное увеличение. TCP-отправитель увеличивает размер своего окна в $(1 + a)$ раз (где $a$ - небольшая положительная
константа: $0 < a < 1$), как только получает верный ACK-пакет.
Найдите функциональную зависимость между частотой потерь $L$ и максимальным размером окна
перегрузки $W$. Утверждается, что для этого измененного протокола TCP, независимо от средней
пропускной способности TCP-соединения всегда требуется одинаковое количество времени для
увеличения размера окна перегрузки с $\frac{W}{2}$ до $W$.

#### Решение
todo

### Задача 4. Расслоение TCP (2 балла)
Для облачных сервисов, таких как поисковые системы, электронная почта и социальные сети,
желательно обеспечить малое время отклика если конечная система расположена далеко от датацентра, то значение RTT будет большим, что может привести к неудовлетворительному времени
отклика, связанному с этапом медленного старта протокола TCP.
Рассмотрим задержку получения ответа на поисковый запрос. Обычно серверу требуется три окна
TCP на этапе медленного старта для доставки ответа. Таким образом, время с момента, когда
конечная система инициировала TCP-соединение, до времени, когда она получила последний
пакет в ответ, составляет примерно $4$ RTT (один RTT для установления TCP-соединения, плюс три
RTT для трех окон данных) плюс время обработки в дата-центре. Такие RTT задержки могут
привести к заметно замедленной выдаче результатов поиска для многих запросов. Более того,
могут присутствовать также и значительные потери пакетов в сетях доступа, приводящие к
повторной передаче TCP и еще большим задержкам.

Один из способов смягчения этой проблемы и улучшения восприятия пользователем
производительности заключается в том, чтобы:
- развернуть внешние серверы ближе к пользователям
- использовать расслоение TCP путем разделения TCP-соединения на внешнем сервере.
При расслоении клиент устанавливает TCP-соединение с ближайшим внешним сервером, который
поддерживает постоянное TCP-соединение с дата-центром с очень большим окном перегрузки TCP.

При использовании такого подхода время отклика примерно равно:
$$4 \cdot RTT_{FE} + RTT_{BE} + \text{ время обработки}~~~~~~~(1)$$
где $RTT_{FE}$ — время оборота между клиентом и внешним сервером, и $RTT_{BE}$ — время оборота
между внешним сервером и дата-центром (внутренним сервером). Если внешний сервер закрыт
для клиента, то это время ответа приближается к $RTT$ плюс время обработки, поскольку значение
$RTT_{FE}$ ничтожно мало и значение $RTT_{BE}$ приблизительно равно $RTT$. В итоге расслоение TCP
может уменьшить сетевую задержку, грубо говоря, с $4 \cdot RTT$ до $RTT$, значительно повышая
субъективную производительность, особенно для пользователей, которые расположены далеко
от ближайшего дата-центра.

Расслоение TCP также помогает сократить задержку повторной передачи TCP, вызванную
потерями в сетях.

Докажите утверждение $(1)$. 

#### Решение
todo
