University: [ITMO University](https://itmo.ru/ru/)  
Faculty: [FICT](https://fict.itmo.ru)  
Course: [Network programming](https://github.com/itmo-ict-faculty/network-programming)  
Year: 2024/2025  
Group: K3320    
Author: Shevchenko Kristina  
Lab: Lab1  
Date of create: 24.06.2024  
Date of finished: 24.06.2024 

## Лабораторная работа №1 "Установка CHR и Ansible, настройка VPN"

***Описание:*** данная работа предусматривает обучение развертыванию виртуальных машин (VM) и системы контроля конфигураций Ansible а также организации собственных VPN серверов.

***Цель работы:*** развернуть виртуальную машину на базе платформы GCP с установленной системой контроля конфигураций Ansible и установить CHR в VirtualBox.</p>

### Ход работы
#### Создание ВМ на GCP
1. Создадим ВМ в Google Cloud на образе Ubuntu и проверим конфигурацию и корректность работы ВМ.
![image](https://github.com/user-attachments/assets/ad4f97fd-dd6a-4482-a119-0d8ad8590838)




2. Установим python3 и ansible. Убедимся, что всё установлено корректно.
![image](https://github.com/user-attachments/assets/64b63077-cc26-461a-acd0-aed486126205)

![image](https://github.com/user-attachments/assets/d9745dae-c835-4267-93aa-22cb69c780f6)




#### Установка CHR (RouterOS) на VirtualBox
1. Создадим ВМ с помощью Oracle VirtualBox с образом с сайта mikrotik.com 

![image](https://github.com/user-attachments/assets/1028d089-7bd1-47df-afaf-4431a6ddf24f)


#### Cоздание Wireguard сервера для организации VPN туннеля между сервером и локальным CHR.

1. Сгенерируем приватный и публичный ключи на GCP ВМ.

```bash
sudo apt update
sudo apt install wireguard iptables

wg genkey | tee private.key | wg pubkey > public.key
sudo mv private.key /etc/wireguard/private.key
sudo mv public.key /etc/wireguard/public.key
```
![image](https://github.com/user-attachments/assets/aeaef1dc-d87c-4a34-9bb8-9ed51cb3e04c)

2. Для настройки сервера создадим конфигурационный файл в `/etc/wireguard/wg0.conf` и добавим необходимые записи.

![image](https://github.com/user-attachments/assets/0d516d93-7a3a-48c7-b1a4-128c6f6db9ba)
Что делают PostUp и PostDown?
Это команды, которые выполняются после поднятия (PostUp) или перед опусканием (PostDown) интерфейса wg0.


`iptables -A FORWARD -i wg0 -j ACCEPT ` — разрешает пересылку трафика, входящего с `wg0`.

`iptables -t nat -A POSTROUTING -o ens4 -j MASQUERADE` — выполняет маскарадинг для трафика, который выходит через `ens4` (интерфейс внешнего подключения).

Соответственно, `-D` в `PostDown` удаляет эти правила при выключении интерфейса.

3. Посмотрим статус Wireguard, убедимся, что всё работает.
   ![Снимок экрана 2025-06-24 155313](https://github.com/user-attachments/assets/56fb3db9-a164-4765-a0df-7fbc766a3b45)



4. Сгенерируем ключи на CHR (и обменяемся ключами с GCP).
![image](https://github.com/user-attachments/assets/71f585d7-aad3-47ad-a3c7-2378a3cab738)


5. Настроим VPN туннель между VPN сервером на Ubuntu и VPN клиентом на RouterOS (CHR).

![image](https://github.com/user-attachments/assets/9947e6b7-12cd-495f-9097-11a9412b485c)

![image](https://github.com/user-attachments/assets/25f9461b-510f-42b3-a594-4752710898ec)


### Проверка доступности

Проверим доступ сервера (GCP) к клиенту (CHR).

![Снимок экрана 2025-06-24 150205](https://github.com/user-attachments/assets/561d4947-b2c0-4525-a932-1a4ad1b2d398)


Поверим доступ клиента к серверу.

![Снимок экрана 2025-06-24 150118](https://github.com/user-attachments/assets/729d0a99-c949-49b6-ae50-b268f1d14caf)




Все пинги прошли успешно! Цель работы достигнута.

### Вывод
В ходе лабораторной работы была развернута виртуальная машина на базе платформы GCP с установленной системой контроля конфигураций Ansible, а также CHR в VirtualBox. Между ними был настроен WireGuard туннель.
