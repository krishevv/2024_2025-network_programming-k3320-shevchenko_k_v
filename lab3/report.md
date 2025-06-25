University: [ITMO University](https://itmo.ru/ru/)  
Faculty: [FICT](https://fict.itmo.ru)  
Course: [Network programming](https://github.com/itmo-ict-faculty/network-programming)  
Year: 2024/2025  
Group: K3320     
Author: Shevchenko Kristina    
Lab: Lab3    
Date of create: 25.06.2025    
Date of finished: 25.06.2025  


## Лабораторная работа №3 "Развертывание Netbox, сеть связи как источник правды в системе технического учета Netbox"

***Описание:*** в данной лабораторной работе вы ознакомитесь с интеграцией Ansible и Netbox и изучите методы сбора информации с помощью данной интеграции.

***Цель работы:*** c помощью Ansible и Netbox собрать всю возможную информацию об устройствах и сохранить их в отдельном файле.

### Ход работы
#### Установка Netbox

1. Создадим новую ВМ в GCP и установим Netbox с помощью докер. При установке использовала [эту статью](https://github.com/netbox-community/netbox-docker/wiki/Getting-Started).



2. Спустя некоторое время контейнеры запущены и "здоровы", контейнер слушает порт 8000
  ![image](https://github.com/user-attachments/assets/b0b8824e-db31-49ec-8ce6-6c0fb9e9dc8d)


4. Необходимо открыть порт в GCP для доступа из браузера, создала новое правило фаервола
VPC Network → Firewall Rules.

```
Name	allow-netbox
Direction	Ingress
Targets	All instances in the network
Source IP ranges	0.0.0.0/0
Protocols and ports	TCP: 8000
```
![image](https://github.com/user-attachments/assets/553ebf20-6665-463d-bf2d-4110725c5423)




После установки мы получаем доступ к приложению в веб-браузере.

![image](https://github.com/user-attachments/assets/f8729c2a-3221-4d91-916d-524ecc4121de)


Создаем суперпользователя пользователя netbox ```docker compose exec netbox /opt/netbox/netbox/manage.py createsuperuser ``` и заходим.

#### Основная часть

1. Добавим наши СНR в раздел "Устройства"

![image](https://github.com/user-attachments/assets/65f915b0-cbb1-42f2-abf3-6d954ffb8eba)


2. Установим ansible-модули для Netbox:

![image](https://github.com/user-attachments/assets/67782ee4-604f-4119-add5-4d222f6d43e3)


3. Создадим файл netbox_conf_galaxy.yml (токен генерируется в приложении netbox)
```
plugin: netbox.netbox.nb_inventory
api_endpoint: http://127.0.0.1:8000
token: 5092c2f60cded6678ed87c10dc1d9f811d5342ed
validate_certs: True
config_context: False
interfaces: True
```
4. Сохраним вывод скрипта в файл командой `ansible-inventory -v --list -y -i netbox_conf_galaxy.yml > netbox_inventory.yml`

5. Перенесём переменные для подключения к роутерам:
```
  vars:
    ansible_connection: ansible.netcommon.network_cli
    ansible_network_os: community.routeros.routeros
    ansible_user: admin
    ansible_ssh_pass: 12345
```

6. Напишем плейбук для изменения имени устройства и добавления IP.
```
- name: Setup Routers
  hosts: ungrouped
  tasks:
    - name: "Change names of chr"
      community.routeros.command:
        commands:
          - /system identity set name="{{ interfaces[0].device.name }}"

    - name: "Change IP-address"
      community.routeros.command:
        commands:
          - /ip address add address="{{ interfaces[0].ip_addresses[0].address }}" interface="{{ interfaces[0].display }}"
```

7. Запустим playbook:

```
sh_kri2003@network-prog:~$ ansible-playbook -i inventory ansible-playbook.yml

PLAY [Setup Routers] ***************************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************************
ok: [chr2]
ok: [chr1]

TASK [Change names of chr] *********************************************************************************************************************************
changed: [chr2]
changed: [chr1]

TASK [Change IP-address] ***********************************************************************************************************************************
changed: [chr2]
changed: [chr1]

PLAY RECAP *************************************************************************************************************************************************
chr1                       : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
chr2                       : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```


8. Напишем сценарий, позволяющий собрать серийный номер устройства и вносящий серийный номер в Netbox.
```
sh_kri2003@network-prog:~$ cat serial_num.yml
---
- name: Collect and update serial number in NetBox
  hosts: chr_routers
  gather_facts: no
  vars:
    netbox_url: "http://34.135.118.73:8000/api/"
    netbox_api_token: "5092c2f60cded6678ed87c10dc1d9f811d5342ed"

  tasks:
    - name: Gather serial number from the device
      community.routeros.command:
        commands:
          - /system license print
      register: serial_output

    - name: Verify that serial number was gathered
      fail:
        msg: "Could not find serial number"
      when: serial_number is not defined or serial_number == ""

    - name: Fetch device ID from NetBox
      uri:
        url: "{{ netbox_url }}/api/dcim/devices/?name={{ inventory_hostname }}"
        method: GET
        headers:
          Authorization: "Token {{ netbox_api_token }}"
        validate_certs: no
        return_content: yes
      register: netbox_device_data
      failed_when: "'results' not in netbox_device_data.json or netbox_device_data.json.results | length == 0"

    - name: Extract device ID from NetBox response
      set_fact:
        device_id: "{{ netbox_device_data.json.results[0].id }}"

    - name: Update serial number in NetBox
      uri:
        url: "{{ netbox_url }}/api/dcim/devices/{{ device_id }}/"
        method: PATCH
        headers:
          Authorization: "Token {{ netbox_api_token }}"
          Content-Type: "application/json"
        body: "{{ {'serial': serial_number} | to_json }}"
        status_code: 200
        validate_certs: no
      register: update_response

    - name: Check if serial number update was successful
      debug:
        msg: "Serial number for {{ inventory_hostname }} updated to {{ serial_number }} in NetBox"
      when: update_response.status == 200
```
9. Появились серийные номера.

![image](https://github.com/user-attachments/assets/a4050df2-10e9-461a-a83c-b1047f096887)


10. Схема связи в draw.io.

![image](https://github.com/user-attachments/assets/55595b40-0c87-47f9-9b1c-09172a8b5a98)


Вывод: в ходе выполнения данной лабораторной работе мы ознакомитесь с интеграцией Ansible и Netbox и изучили методы сбора информации с помощью данной интеграции.
