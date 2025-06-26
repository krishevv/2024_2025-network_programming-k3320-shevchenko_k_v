University: [ITMO University](https://itmo.ru/ru/)  
Faculty: [FICT](https://fict.itmo.ru)  
Course: [Network programming](https://github.com/itmo-ict-faculty/network-programming)  
Year: 2024/2025    
Group: K3320   
Author: Shevchenko Kristina   
Lab: Lab4    
Date of create: 26.06.2025    
Date of finished:  26.06.2025  

## Лабораторная работа №4 "Базовая 'коммутация' и туннелирование используя язык программирования P4"

***Описание***
В данной лабораторной работе будет осуществлено знакомство на практике с языком программирования P4, разработанный компанией Barefoot (ныне Intel) для организации процесса обработки сетевого трафика на скорости чипа. Barefoot разработал несколько FPGA чипов для обработки трафика которые были встроенны в некоторые модели коммутаторов Arista и Brocade.

***Цель работы***
Изучить синтаксис языка программирования P4 и выполнить 2 задания обучающих задания от Open network foundation для ознакомления на практике с P4.

# Ход работы

## Создание виртуальной машины 
1. Установила [vagrant](https://vagrantup.com) 


2. Склонировала репозиторий 
```
git clone https://github.com/p4lang/tutorials
cd tutorials/vm/
```
3. И запустила виртуальную машину `vagrant up`
![image](https://github.com/user-attachments/assets/cff4843e-703e-4cd0-88bf-21c47aca5772)


## Базовая переадресация

1. Перейдём в директорию с заданием и выполним `make run` для сборки `basic.p4`
   
2. После запуска Mininet проверим, что пинг не идет. 

![image](https://github.com/user-attachments/assets/0933d7fa-119f-4414-9b2d-cf11979fcd3e)

![image](https://github.com/user-attachments/assets/6da93821-1cc2-4503-ba3d-1ac86b0a6e14)


3. `basic.p4` внесены следующие изменения:
   
- В раздел парсер добавлен парсинг Ethernet и IPv4 заголовков. В зависимости от значения поля `etherType` определяется дальнейшее состояние.
```c
/*************************************************************************
*********************** P A R S E R  ***********************************
*************************************************************************/

parser MyParser(packet_in packet,
                out headers hdr,
                inout metadata meta,
                inout standard_metadata_t standard_metadata) {

    state start {
        transition parse_ethernet;
    }

    state parse_ethernet {
        packet.extract(hdr.ethernet);
        transition select(hdr.ethernet.etherType) {
            TYPE_IPV4 : parse_ipv4;
            default : accept;
        }
    }

    state parse_ipv4 {
        packet.extract(hdr.ipv4);
        transition accept;
    }
}
```

- В раздел Ingress добавлены:

  - действие `ipv4_forward` для настройки MAC-адресов и порта выхода,

  - таблица `ipv4_lpm`, реализующая маршрутизацию по IP-адресу назначения.

  
```c
/*************************************************************************
**************  I N G R E S S   P R O C E S S I N G   *******************
*************************************************************************/

control MyIngress(inout headers hdr,
                  inout metadata meta,
                  inout standard_metadata_t standard_metadata) {
    action drop() {
        mark_to_drop(standard_metadata);
    }

    action ipv4_forward(macAddr_t dstAddr, egressSpec_t port) {
        standard_metadata.egress_spec = port;
        hdr.ethernet.srcAddr = hdr.ethernet.dstAddr;
        hdr.ethernet.dstAddr = dstAddr;
        hdr.ipv4.ttl = hdr.ipv4.ttl - 1;
    }

    table ipv4_lpm {
        key = {
            hdr.ipv4.dstAddr: lpm;
        }
        actions = {
            ipv4_forward;
            drop;
            NoAction;
        }
        size = 1024;
        default_action = NoAction();
    }

    apply {
        if (hdr.ipv4.isValid()) {
            ipv4_lpm.apply();
        }
    }
}
```

- В разделе Deparsers Добавлена отправка обратно Ethernet и IPv4 заголовков.

```c
/*************************************************************************
***********************  D E P A R S E R  *******************************
*************************************************************************/

control MyDeparser(packet_out packet, in headers hdr) {
    apply {
        packet.emit(hdr.ethernet);
        packet.emit(hdr.ipv4);
    }
}
```
4. Сеть перезапускается. Хосты теперь могут пинговать друг друга.

![image](https://github.com/user-attachments/assets/92544ca0-edce-4705-b3e9-cfe2a7f8f34f)


## Базовое туннелирование

1. В basic_tunnel.p4 внесены следущие изменения:


- Добавлены структуры для собственного туннельного заголовка.

- Константа `bit<16> TYPE_MYTUNNEL = 0x1212` и проверим заголовок `header myTunnel_t `, который включает id протокола и коммутатора приема.

```c
const bit<16> TYPE_MYTUNNEL = 0x1212;
const bit<16> TYPE_IPV4 = 0x800;

/*************************************************************************
*********************** H E A D E R S  ***********************************
*************************************************************************/

typedef bit<9>  egressSpec_t;
typedef bit<48> macAddr_t;
typedef bit<32> ip4Addr_t;

header ethernet_t {
    macAddr_t dstAddr;
    macAddr_t srcAddr;
    bit<16>   etherType;
}

header myTunnel_t {
    bit<16> proto_id;
    bit<16> dst_id;
}

header ipv4_t {
    bit<4>    version;
    bit<4>    ihl;
    bit<8>    diffserv;
    bit<16>   totalLen;
    bit<16>   identification;
    bit<3>    flags;
    bit<13>   fragOffset;
    bit<8>    ttl;
    bit<8>    protocol;
    bit<16>   hdrChecksum;
    ip4Addr_t srcAddr;
    ip4Addr_t dstAddr;
}

struct metadata {
    /* empty */
}

struct headers {
    ethernet_t   ethernet;
    myTunnel_t   myTunnel;
    ipv4_t       ipv4;
}
```

- В Parser добавлено состояние `parse_myTunnel`. Оно вызывается, если `etherType` соответствует нашему туннелю. При необходимости происходит переход к парсингу IPv4.

```c
/*************************************************************************
*********************** P A R S E R  ***********************************
*************************************************************************/

parser MyParser(packet_in packet,
                out headers hdr,
                inout metadata meta,
                inout standard_metadata_t standard_metadata) {

    state start {
        transition parse_ethernet;
    }

    state parse_ethernet {
        packet.extract(hdr.ethernet);
        transition select(hdr.ethernet.etherType) {
            TYPE_IPV4 : parse_ipv4;
            TYPE_MYTUNNEL : parse_myTunnel;
            default : accept;
        }
    }

    state parse_ipv4 {
        packet.extract(hdr.ipv4);
        transition accept;
    }
    
    state parse_myTunnel {
        packet.extract(hdr.myTunnel);
        transition select(hdr.myTunnel.proto_id) {
            TYPE_IPV4 : parse_ipv4;
            default : accept;    
        }
    }

}
```

- В Ingress добавлена логика обработки туннелированных пакетов:

  - при отсутствии туннеля используется обычная IPv4 маршрутизация;

  - при наличии туннеля — специальная таблица маршрутизации `myTunnel_exact`.
```c
/*************************************************************************
**************  I N G R E S S   P R O C E S S I N G   *******************
*************************************************************************/

control MyIngress(inout headers hdr,
                  inout metadata meta,
                  inout standard_metadata_t standard_metadata) {
    action drop() {
        mark_to_drop(standard_metadata);
    }

    action ipv4_forward(macAddr_t dstAddr, egressSpec_t port) {
        standard_metadata.egress_spec = port;
        hdr.ethernet.srcAddr = hdr.ethernet.dstAddr;
        hdr.ethernet.dstAddr = dstAddr;
        hdr.ipv4.ttl = hdr.ipv4.ttl - 1;
    }


    table ipv4_lpm {
        key = {
            hdr.ipv4.dstAddr: lpm;
        }
        actions = {
            ipv4_forward;
            drop;
            NoAction;
        }
        size = 1024;
        default_action = drop();
    }

    action myTunnel_forward(egressSpec_t port) {
        standard_metadata.egress_spec = port;
    }

    table myTunnel_exact {
        key = {
            hdr.myTunnel.dst_id: exact;
        }
        actions = {
            myTunnel_forward;
            drop;
            NoAction;
        }
        size = 1024;
        default_action = NoAction();
    }

    apply {
        if (hdr.ipv4.isValid() && !hdr.myTunnel.isValid()) {
            ipv4_lpm.apply();
        }
        if (hdr.myTunnel.isValid()) {
            myTunnel_exact.apply();
        }
    }
}
```

- В раздел Deparsers добавлен вывод заголовков в следующем порядке: Ethernet -> MyTunnel -> IPv4.

```c
/*************************************************************************
***********************  D E P A R S E R  *******************************
*************************************************************************/

control MyDeparser(packet_out packet, in headers hdr) {
    apply {
        packet.emit(hdr.ethernet);
        packet.emit(hdr.myTunnel);
        packet.emit(hdr.ipv4);
    }
}
```

2. Выполним вход в Mininet и откроем два терминала. Проверим корректность работы без туннелирования `./send.py 10.0.2.2 "Hello"`. Доставка сообщения выполнена успешно.

![image](https://github.com/user-attachments/assets/cdeecbef-2986-4da7-ab19-27b51f5e9361)


3. Отправка с туннелированием `./send.py 10.0.2.2 "Hello" --dst_id 2`.  Пакет доставлен через туннель

![image](https://github.com/user-attachments/assets/331aceba-dd4d-41f9-a233-6f02955c3f57)


4. Проверка маршрутизации по неправильному IP: `./send.py 10.0.2.3 "Helloooo" --dst_id 2`.
    Пакет всё равно доставлен, так как используется `myTunnel.dst_id ` вместо IP
![image](https://github.com/user-attachments/assets/d2b2a6da-5456-4489-b93b-ddc2f775be84)


### Вывод
В лабораторной работе реализованы базовая коммутация и туннелирование на языке P4. Это позволило понять структуру программы на P4 и научиться обрабатывать заголовки пакетов в разных режимах.
