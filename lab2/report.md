University: [ITMO University](https://itmo.ru/ru/)  
Faculty: [FICT](https://fict.itmo.ru)  
Course: [Network programming](https://github.com/itmo-ict-faculty/network-programming)  
Year: 2024/2025  
Group: K3320     
Author: Shevchenko Kristina    
Lab: Lab2    
Date of create: 24.06.2025    
Date of finished: 24.06.2025    

## Лабораторная работа №2 "Развертывание дополнительного CHR, первый сценарий Ansible"

***Описание:*** В данной лабораторной работе будет осуществлено практическое знакомство с системой управления конфигурацией Ansible, использующейся для автоматизации настройки и развертывания программного обеспечения.

***Цель работы:*** с помощью Ansible настроить несколько сетевых устройств и собрать информацию о них. Правильно собрать файл Inventory.</p>

### Ход работы
#### Развертывание дополнительного CHR
1. Создадим ВМ с помощью Oracle VirtualBox с образом, который был скачан с сайта mikrotik.com.

*Скачала образ диска другой версии, иначе не получалось.*

![image](https://github.com/user-attachments/assets/7460468d-d509-4948-929d-4518247dc4a1)


2. Запустим обе машины и убедимся, что они имеют разные ip-адреса.

![image](https://github.com/user-attachments/assets/dd230185-b516-4cb3-909e-14337a37ce5c)

![image](https://github.com/user-attachments/assets/46b9ef57-a201-490d-a618-996f5bc046a3)




3. Создадим WireGuard Client на втором CHR и проверим доступ до сервера 10.0.0.1 и между клиентами 10.0.0.2 10.0.0.3 Пропингуем сервер с клиентов, чтобы протестировать связь.
   
CHR1 



![image](https://github.com/user-attachments/assets/15135d0a-b800-41cf-a610-2a1f7a394563)



CHR2




![image](https://github.com/user-attachments/assets/72db41f4-72fc-4a9b-b827-c6b2d2647f7e)



   


### Настройка Ansible

Используя Ansible настроим на 2-х CHR:
* логин/пароль;
* NTP Client;
* OSPF с указанием Router ID;
* сбор данных по OSPF топологии и полный конфиг устройства.

1. Перед настройкой проверим версию Ansible и что коллекция для roterous установлена.
  
![image](https://github.com/user-attachments/assets/54192050-de75-4ca7-8f02-a49dd6c43ad4)


2. Ansible использует SSH для подключения к удаленным хостам. Чтобы избежать ошибок, создадим публичный ключ на сервере и импортируем его на клиентов.

![image](https://github.com/user-attachments/assets/5b86336e-85ee-4cb5-a1c0-4bbe574504df)


3. Далее создадим inventory/hosts — это файл инвентаря (inventory file), который используется в Ansible для управления списком хостов.

```
[chr_routers]
chr1 ansible_host=10.0.0.2 ansible_user=admin
chr2 ansible_host=10.0.0.3 ansible_user=admin

[chr_routers:vars]
ansible_connection=ansible.netcommon.network_cli
ansible_network_os=community.routeros.routeros
ansible_ssh_private_key_file=/home/sh_kri2003/.ssh/id_rsa
```

4. И конфигурационныйй файл. Где прописано, что где лежит.

```
[defaults]
inventory = ./inventory/hosts
host_key_checking = False
ansible_remote_tmp = /tmp
collections_paths = /home/sh_kri2003/.ansible/collections:/usr/share/ansible/collections
```
5. Плейбук выглядит следующим образом:

```
- name: Configure CHR routers with user credentials, NTP client, and OSPF
  hosts: chr_routers
  gather_facts: no
  vars:
    router_ospf_ip: "{{ '10.255.255.1/32' if ansible_host == '10.0.0.2' else '10.255.255.2/32' }}"
  tasks:
    - name: Set up user credentials
      community.routeros.command:
        commands:
          - /user add name=kri group=full password=111
      register: user_config

    - name: Enable NTP client and configure server
      community.routeros.command:
        commands:
          - /system ntp client set enabled=yes servers=0.ru.pool.ntp.org
      register: ntp_client_config

    - name: Configure OSPF
      community.routeros.command:
        commands:
          - /routing ospf instance add name=default
          - /interface bridge add name=loopback
          - /ip address add address={{ router_ospf_ip }} interface=loopback
          - /routing ospf instance set 0 router-id={{ router_ospf_ip }}
          - /routing ospf area add instance=default name=backbone
          - /routing ospf interface-template add area=backbone interfaces=ether1 type=ptp

    - name: Export router configuration
      community.routeros.command:
        commands:
          - /export
      register: router_config

    - name: Display router configuration
      debug:
        var: router_config.stdout_lines
```
6. Запустим его и получим следующий результат:

```
sh_kri2003@network-prog:~$ ansible-playbook chr_conf.yml

PLAY [Configure CHR routers with user credentials, NTP client, and OSPF] ***********************************************************************************

TASK [Set up user credentials] *****************************************************************************************************************************
changed: [chr2]
changed: [chr1]

TASK [Enable NTP client and configure server] **************************************************************************************************************
changed: [chr2]
changed: [chr1]

TASK [Configure OSPF] **************************************************************************************************************************************
changed: [chr2]
changed: [chr1]

TASK [Export router configuration] *************************************************************************************************************************
changed: [chr2]
changed: [chr1]

TASK [Display router configuration] ************************************************************************************************************************
ok: [chr1] => {
"router_config.stdout_lines": [
    [
        "# 2025-06-24 20:15:13 by RouterOS 7.18.2",
        "# software id = ",
        "#",
        "/interface bridge",
        "add name=loopback",
        "/interface wireguard",
        "add listen-port=61055 mtu=1420 name=wg0",
        "/routing ospf instance",
        "add disabled=no name=default",
        "/routing ospf area",
        "add disabled=no instance=default name=backbone",
        "/interface wireguard peers",
        "add allowed-address=10.0.0.0/24 endpoint-address=35.228.196.92  endpoint-port=\\",
        "    61055 interface=wg0 name=peer1 persistent-keepalive=25s public-key=\\",
        "    \"HoU536wWstLLPc1MwzOpX77TVSRuNQnEcNDXG6qIvik=\"",
        "/ip address",
        "add address=10.0.0.2/24 interface=wg0 network=10.0.0.0",
        "add address=10.255.255.1 interface=loopback network=10.255.255.1",
        "/ip dhcp-client",
        "add interface=ether1",
        "/ip firewall nat",
        "add action=masquerade chain=srcnat",
        "/ip ssh",
        "set always-allow-password-login=yes forwarding-enabled=both host-key-type=\\",
        "    ed25519",
        "/routing ospf interface-template",
        "add area=backbone disabled=no interfaces=ether1 type=ptp",
        "/system note",
        "set show-at-login=no",
        "/system ntp client",
        "set enabled=yes",
        "/system ntp client servers",
        "add address=0.ru.pool.ntp.org"
    ]
]
}
ok: [chr2] => {
"router_config.stdout_lines": [
    [
        "# 2025-06-24 20:15:13 by RouterOS 7.18.1",
        "# software id = ",
        "#",
        "/interface bridge",
        "add name=loopback",
        "/interface wireguard",
        "add listen-port=61055 mtu=1420 name=wg1",
        "/routing ospf instance",
        "add disabled=no name=default",
        "/routing ospf area",
        "add disabled=no instance=default name=backbone",
        "/interface wireguard peers",
        "add allowed-address=10.0.0.0/24 endpoint-address=35.228.196.92  endpoint-port=\\",
        "    61055 interface=wg1 name=peer2 persistent-keepalive=25s public-key=\\",

        "    \"HoU536wWstLLPc1MwzOpX77TVSRuNQnEcNDXG6qIvik= "",
        "/ip address",
        "add address=10.0.0.3/24 interface=wg1 network=10.0.0.0",
        "add address=10.255.255.2 interface=loopback network=10.255.255.2",
        "/ip dhcp-client",
        "add interface=ether1",
        "/ip firewall nat",
        "add action=masquerade chain=srcnat",
        "/ip ssh",
        "set always-allow-password-login=yes forwarding-enabled=both host-key-type=\\",
        "    ed25519",
        "/routing ospf interface-template",
        "add area=backbone disabled=no interfaces=ether1 type=ptp",
        "/system note",
        "set show-at-login=no",
        "/system ntp client",
        "set enabled=yes",
        "/system ntp client servers",
        "add address=0.ru.pool.ntp.org"
    ]
]
}

PLAY RECAP *************************************************************************************************************************************************
chr1                       : ok=5    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
chr2                       : ok=5    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

Проверим на клиенте, что всё выролнено верно.

![image](https://github.com/user-attachments/assets/236099a3-5fa2-43d7-bb5d-4b1b289e0595)


### Вывод
В ходе лабораторной работы с помощью Ansible были настроены сетевые устройства и собрана информация о них.
