# Практика 9. Сетевой уровень

## Wireshark: ICMP
В лабораторной работе предлагается исследовать ряд аспектов протокола ICMP:
- ICMP-сообщения, генерируемые программой Ping
- ICMP-сообщения, генерируемые программой Traceroute
- Формат и содержимое ICMP-сообщения

### 1. Ping (4 балла)
Программа Ping на исходном хосте посылает пакет на целевой IP-адрес; если хост с этим адресом
активен, то программа Ping на нем откликается, отсылая ответный пакет хосту, инициировавшему
связь. Оба этих пакета Ping передаются по протоколу ICMP.

Выберите какой-либо хост, расположенный на другом континенте (например, в Америке или
Азии). Захватите с помощью Wireshark ICMP пакеты от утилиты ping.
Для этого из командной строки запустите команду (аргумент `-n 10` означает, что должно быть
отослано 10 ping-сообщений): `ping –n 10 host_name`

Для анализа пакетов в Wireshark введите строку icmp в области фильтрации вывода.

#### Вопросы
1. Каков IP-адрес вашего хоста? Каков IP-адрес хоста назначения?
   - <!-- todo -->
   - <!-- todo -->
2. Почему ICMP-пакет не обладает номерами исходного и конечного портов?
   - <!-- todo -->
3. Рассмотрите один из ping-запросов, отправленных вашим хостом. Каковы ICMP-тип и кодовый
   номер этого пакета? Какие еще поля есть в этом ICMP-пакете? Сколько байт приходится на поля 
   контрольной суммы, порядкового номера и идентификатора?
   - <!-- todo -->
   - <!-- todo -->
   - <!-- todo -->
4. Рассмотрите соответствующий ping-пакет, полученный в ответ на предыдущий. 
   Каковы ICMP-тип и кодовый номер этого пакета? Какие еще поля есть в этом ICMP-пакете? 
   Сколько байт приходится на поля контрольной суммы, порядкового номера и идентификатора?
   - <!-- todo -->
   - <!-- todo -->
   - <!-- todo -->

### 2. Traceroute (4 балла)
Программа Traceroute может применяться для определения пути, по которому пакет попал с
исходного на конечный хост.

Traceroute отсылает первый пакет со значением TTL = 1, второй – с TTL = 2 и т.д. Каждый
маршрутизатор понижает TTL-значение пакета, когда пакет проходит через этот маршрутизатор.
Когда на маршрутизатор приходит пакет со значением TTL = 1, этот маршрутизатор отправляет
обратно к источнику ICMP-пакет, свидетельствующий об ошибке.

Задача – захватить ICMP пакеты, инициированные программой traceroute, в сниффере Wireshark.
В ОС Windows вы можете запустить: `tracert host_name`

Выберите хост, который **расположен на другом континенте**.

#### Вопросы
1. Рассмотрите ICMP-пакет с эхо-запросом на вашем скриншоте. Отличается ли он от ICMP-пакетов
   с ping-запросами из Задания 1 (Ping)? Если да – то как?
   - <!-- todo -->
2. Рассмотрите на вашем скриншоте ICMP-пакет с сообщением об ошибке. В нем больше полей,
   чем в ICMP-пакете с эхо-запросом. Какая информация содержится в этих дополнительных полях?
   - <!-- todo -->
3. Рассмотрите три последних ICMP-пакета, полученных исходным хостом. Чем эти пакеты
   отличаются от ICMP-пакетов, сообщающих об ошибках? Чем объясняются такие отличия?
   - <!-- todo -->
   - <!-- todo -->
4. Есть ли такой канал, задержка в котором существенно превышает среднее значение? Можете
   ли вы, опираясь на имена маршрутизаторов, определить местоположение двух маршрутизаторов,
   расположенных на обоих концах этого канала?
   - <!-- todo -->
   - <!-- todo -->

## Программирование.

### 1. IP-адрес и маска сети (1 балл)
Напишите консольное приложение, которое выведет IP-адрес вашего компьютера и маску сети на консоль.

#### Демонстрация работы

PowerShell
```
Get-NetIPAddress -AddressFamily IPv4 | Where-Object { $_.AddressState -eq "Preferred" } | 
    Select-Object InterfaceAlias, IPAddress, PrefixLength | 
    Format-Table -AutoSize
```


### 2. Доступные порты (2 балла)
Выведите все доступные (свободные) порты в указанном диапазоне для заданного IP-адреса. 
IP-адрес и диапазон портов должны передаваться в виде входных параметров.

#### Демонстрация работы

```
param(
    [string]$ipAddress,
    [int]$startPort,
    [int]$endPort
)

for ($port = $startPort; $port -le $endPort; $port++) {
    $tcpClient = New-Object System.Net.Sockets.TcpClient
    try {
        $tcpClient.Connect($ipAddress, $port)
    }
    catch {
        # Если соединение не установлено, значит порт свободен
        "Port $port is available"
    }
    finally {
        $tcpClient.Close()
    }
}
```

```
.\CheckPorts.ps1 -ipAddress "192.168.1.1" -startPort 80 -endPort 100
```


### 3. Широковещательная рассылка для подсчета копий приложения (6 баллов)
Разработать приложение, подсчитывающее количество копий себя, запущенных в локальной сети.
Приложение должно использовать набор сообщений, чтобы информировать другие приложения
о своем состоянии. После запуска приложение должно рассылать широковещательное сообщение
о том, что оно было запущено. Получив сообщение о запуске другого приложения, оно должно
сообщать этому приложению о том, что оно работает. Перед завершением работы приложение
должно информировать все известные приложения о том, что оно завершает работу. На экран
должен выводиться список IP адресов компьютеров (с указанием портов), на которых приложение
запущено.

Приложение считает другое приложение запущенным, если в течение промежутка времени,
равного нескольким интервалам между рассылками широковещательных сообщений, от него
пришло сообщение.

**Такое приложение может быть использовано, например, при наличии ограничения на
количество лицензионных копий программ.*

Пример GUI:

<img src="images/copies.png" width=200 />

#### Демонстрация работы

```
package org.example;

import org.java_websocket.WebSocket;
import org.java_websocket.client.WebSocketClient;
import org.java_websocket.handshake.ClientHandshake;
import org.java_websocket.handshake.ServerHandshake;
import org.java_websocket.server.WebSocketServer;

import javax.swing.*;
import java.awt.*;
import java.net.InetSocketAddress;
import java.net.URI;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

public class WebSocketApp {

    private static final int PORT = 8080;
    private static Map<String, Long> clients = Collections.synchronizedMap(new HashMap<>());
    private static DefaultListModel<String> listModel = new DefaultListModel<>();

    public static void main(String[] args) throws Exception {
        startUI();
        WebSocketServer server = new Server(new InetSocketAddress(PORT));
        server.start();

        UUID uid = UUID.randomUUID();
        WebSocketClient client = new Client(new URI("ws://localhost:" + PORT), uid.toString());
        client.connect();

        Timer timer = new Timer(2000, e -> {
            if (client.isOpen()) {
                client.send(uid.toString());
            }
        });
        timer.start();
    }

    private static void startUI() {
        JFrame frame = new JFrame("WebSocket Clients");
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        frame.setSize(300, 300);
        JList<String> list = new JList<>(listModel);
        frame.add(new JScrollPane(list), BorderLayout.CENTER);
        frame.setVisible(true);
    }

    private static class Server extends WebSocketServer {
        public Server(InetSocketAddress address) {
            super(address);
        }

        @Override
        public void onOpen(WebSocket conn, ClientHandshake handshake) {
            System.out.println("New connection: " + conn.getRemoteSocketAddress());
        }

        public boolean startSocketServer() {
            try {
                this.start();
                return true;
            } catch (Exception e) {
                return false;
            }
        }

        @Override
        public void onClose(org.java_websocket.WebSocket conn, int code, String reason, boolean remote) {
            System.out.println("Closed connection: " + conn.getRemoteSocketAddress());
        }

        @Override
        public void onMessage(org.java_websocket.WebSocket conn, String message) {
            clients.put(message, System.currentTimeMillis());
            broadcast(message);
            if (!listModel.contains(message)) {
                listModel.addElement(message);
            }
            System.out.println("Message from client: " + message);
        }

        @Override
        public void onError(org.java_websocket.WebSocket conn, Exception ex) {
            ex.printStackTrace();
        }

        @Override
        public void onStart() {
            System.out.println("Server started successfully");
        }
    }

    private static class Client extends WebSocketClient {
        private final String uid;

        public Client(URI serverUri, String uid) {
            super(serverUri);
            this.uid = uid;
        }

        @Override
        public void onOpen(ServerHandshake handshake) {
            System.out.println("Opened connection for client: " + uid);
        }

        @Override
        public void onMessage(String message) {
            if (!listModel.contains(message)) {
                listModel.addElement(message);
            }
            System.out.println("Server broadcast: " + message);
        }

        @Override
        public void onClose(int code, String reason, boolean remote) {
            listModel.removeElement(uid);
            System.out.println("Closed connection for client: " + uid);
        }

        @Override
        public void onError(Exception ex) {
            ex.printStackTrace();
        }
    }
}
```


## Задачи. Работа протокола TCP

### Задача 1. Докажите формулы (3 балла)
Пусть за период времени, в который изменяется скорость соединения с $\frac{W}{2 \cdot RTT}$
до $\frac{W}{RTT}$, только один пакет был потерян (очень близко к концу периода).
1. Докажите, что частота потери $L$ (доля потерянных пакетов) равна
   $$L = \dfrac{1}{\frac{3}{8} W^2 + \frac{3}{4} W}$$
2. Используйте выше полученный результат, чтобы доказать, что, если частота потерь равна
   $L$, то средняя скорость приблизительно равна
   $$\approx \dfrac{1.22 \cdot MSS}{RTT \cdot \sqrt{L}}$$

#### Решение
todo

### Задача 2. Найдите функциональную зависимость (3 балла)
Рассмотрим модификацию алгоритма управления перегрузкой протокола TCP. Вместо
аддитивного увеличения, мы можем использовать мультипликативное увеличение. 
TCP-отправитель увеличивает размер своего окна в небольшую положительную 
константу $a$ ($a > 1$), как только получает верный ACK-пакет.
1. Найдите функциональную зависимость между частотой потерь $L$ и максимальным
размером окна перегрузки $W$.
2. Докажите, что для этого измененного протокола TCP, независимо от средней пропускной
способности, TCP-соединение всегда требуется одинаковое количество времени для
увеличения размера окна перегрузки с $\frac{W}{2}$ до $W$.

#### Решение
todo
