# Домашнее задание к занятию «Ansible.Часть 2» Дойков А.М.

### Оформление домашнего задания

1. Домашнее задание выполните в [Google Docs](https://docs.google.com/) и отправьте на проверку ссылку на ваш документ в личном кабинете.  
1. В названии файла укажите номер лекции и фамилию студента. Пример названия:  Ansible. Часть 2 — Александр Александров.
1. Перед отправкой проверьте, что доступ для просмотра открыт всем, у кого есть ссылка. Если нужно прикрепить дополнительные ссылки, добавьте их в свой Google Docs.

Вы можете прислать решение в виде ссылки на ваш репозийторий в GitHub, для этого воспользуйтесь [шаблоном для домашнего задания](https://github.com/netology-code/sys-pattern-homework).

---

### Задание 1

**Выполните действия, приложите файлы с плейбуками и вывод выполнения.**

Напишите три плейбука. При написании рекомендуем использовать текстовый редактор с подсветкой синтаксиса YAML.

Плейбуки должны:

1. Скачать какой-либо архив, создать папку для распаковки и распаковать скаченный архив. Например, можете использовать [официальный сайт](https://kafka.apache.org/downloads) и зеркало Apache Kafka. При этом можно скачать как исходный код, так и бинарные файлы, запакованные в архив — в нашем задании не принципиально.

playbook

```YAML
# install_kafka.yml
---
- name: Download and extract Apache Kafka
  hosts: all
  become: yes
  vars:
    kafka_version: "3.6.1"
    kafka_scala_version: "2.13"
    install_dir: "/opt/kafka"
    download_url: "https://downloads.apache.org/kafka/{{ kafka_version }}/kafka_{{ kafka_scala_version }}-{{ kafka_version }}.tgz"

  tasks:
    - name: Create installation directory
      ansible.builtin.file:
        path: "{{ install_dir }}"
        state: directory
        owner: root
        group: root
        mode: '0755'

    - name: Download Kafka archive
      ansible.builtin.get_url:
        url: "{{ download_url }}"
        dest: "/tmp/kafka.tgz"
        timeout: 60

    - name: Extract Kafka archive
      ansible.builtin.unarchive:
        src: "/tmp/kafka.tgz"
        dest: "{{ install_dir }}"
        remote_src: yes
        extra_opts: ["--strip-components=1"]
        creates: "{{ install_dir }}/bin"

    - name: Set ownership of Kafka directory
      ansible.builtin.file:
        path: "{{ install_dir }}"
        owner: root
        group: root
        recurse: yes

    - name: Clean up temporary archive
      ansible.builtin.file:
        path: "/tmp/kafka.tgz"
        state: absent

    - name: Display installation info
      ansible.builtin.debug:
        msg: "Kafka {{ kafka_version }} successfully installed to {{ install_dir }}"
```

output log

```

PLAY [Download and extract Apache Kafka] ***************************************

TASK [Gathering Facts] *********************************************************
ok: [Ex2]
ok: [10.129.0.33]
ok: [Ex1]
ok: [10.129.0.25]

TASK [Create installation directory] *******************************************
ok: [10.129.0.33]
ok: [Ex2]
ok: [10.129.0.25]
ok: [Ex1]

TASK [Download Kafka archive] **************************************************
changed: [10.129.0.33]
changed: [10.129.0.25]
ok: [Ex1]
ok: [Ex2]

TASK [Extract Kafka archive] ***************************************************
changed: [Ex1]
changed: [10.129.0.25]
changed: [10.129.0.33]
changed: [Ex2]

TASK [Set ownership of Kafka directory] ****************************************
ok: [10.129.0.25]
ok: [10.129.0.33]
ok: [Ex2]
ok: [Ex1]

TASK [Clean up temporary archive] **********************************************
ok: [Ex1]
changed: [10.129.0.25]
changed: [10.129.0.33]
ok: [Ex2]

TASK [Display installation info] ***********************************************
ok: [10.129.0.25] => {
    "msg": "Kafka 4.1.0 successfully installed to /opt/kafka"
}
ok: [10.129.0.33] => {
    "msg": "Kafka 4.1.0 successfully installed to /opt/kafka"
}
ok: [Ex1] => {
    "msg": "Kafka 4.1.0 successfully installed to /opt/kafka"
}
ok: [Ex2] => {
    "msg": "Kafka 4.1.0 successfully installed to /opt/kafka"
}

PLAY RECAP *********************************************************************
10.129.0.25                : ok=7    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
10.129.0.33                : ok=7    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
Ex1                        : ok=7    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
Ex2                        : ok=7    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```

2. Установить пакет tuned из стандартного репозитория вашей ОС. Запустить его, как демон — конфигурационный файл systemd появится автоматически при установке. Добавить tuned в автозагрузку.

playbook
```YAML
# setup_tuned.yml
---
- name: Install and configure tuned
  hosts: all
  become: yes

  tasks:
    - name: Update package cache (for apt-based systems)
      ansible.builtin.apt:
        update_cache: yes
      when: ansible_pkg_mgr == 'apt'

    - name: Update package cache (for yum-based systems)
      ansible.builtin.yum:
        update_cache: yes
      when: ansible_pkg_mgr == 'yum'

    - name: Update package cache (for dnf-based systems)
      ansible.builtin.dnf:
        update_cache: yes
      when: ansible_pkg_mgr == 'dnf'

    - name: Install tuned package
      ansible.builtin.package:
        name: tuned
        state: present

    - name: Start tuned service
      ansible.builtin.systemd:
        name: tuned
        state: started
        enabled: yes
        daemon_reload: yes

    - name: Enable and start tuned-profile service (for RHEL/CentOS 8+)
      ansible.builtin.systemd:
        name: tuned-profile
        state: started
        enabled: yes
      when: ansible_os_family == "RedHat" and ansible_distribution_major_version | int >= 8

    - name: Verify tuned service status
      ansible.builtin.systemd:
        name: tuned
        state: started
        enabled: yes

    - name: Display tuned status
      ansible.builtin.command: tuned-adm active
      register: tuned_status
      changed_when: false

    - name: Show tuned status
      ansible.builtin.debug:
        msg: "{{ tuned_status.stdout }}"
```

output log

```
PLAY [Install and configure tuned] *********************************************

TASK [Gathering Facts] *********************************************************
ok: [10.129.0.25]
ok: [10.129.0.33]
ok: [Ex1]
ok: [Ex2]

TASK [Update package cache (for apt-based systems)] ****************************
changed: [Ex2]
changed: [10.129.0.33]
changed: [Ex1]
changed: [10.129.0.25]

TASK [Update package cache (for yum-based systems)] ****************************
skipping: [10.129.0.25]
skipping: [10.129.0.33]
skipping: [Ex1]
skipping: [Ex2]

TASK [Update package cache (for dnf-based systems)] ****************************
skipping: [10.129.0.25]
skipping: [10.129.0.33]
skipping: [Ex1]
skipping: [Ex2]

TASK [Install tuned package] ***************************************************
fatal: [10.129.0.33]: FAILED! => {"cache_update_time": 1758892758, "cache_updated": false, "changed": false, "msg": "'/usr/bin/apt-get -y -o \"Dpkg::Options::=--force-confdef\" -o \"Dpkg::Options::=--force-confold\"       install 'tuned=2.21.0-1ubuntu2'' failed: E: Could not get lock /var/lib/dpkg/lock-frontend. It is held by process 5350 (apt-get)\nE: Unable to acquire the dpkg frontend lock (/var/lib/dpkg/lock-frontend), is another process using it?\n", "rc": 100, "stderr": "E: Could not get lock /var/lib/dpkg/lock-frontend. It is held by process 5350 (apt-get)\nE: Unable to acquire the dpkg frontend lock (/var/lib/dpkg/lock-frontend), is another process using it?\n", "stderr_lines": ["E: Could not get lock /var/lib/dpkg/lock-frontend. It is held by process 5350 (apt-get)", "E: Unable to acquire the dpkg frontend lock (/var/lib/dpkg/lock-frontend), is another process using it?"], "stdout": "", "stdout_lines": []}
fatal: [Ex1]: FAILED! => {"cache_update_time": 1758892780, "cache_updated": false, "changed": false, "msg": "'/usr/bin/apt-get -y -o \"Dpkg::Options::=--force-confdef\" -o \"Dpkg::Options::=--force-confold\"       install 'tuned=2.21.0-1ubuntu2'' failed: E: Could not get lock /var/lib/dpkg/lock-frontend. It is held by process 6985 (apt-get)\nE: Unable to acquire the dpkg frontend lock (/var/lib/dpkg/lock-frontend), is another process using it?\n", "rc": 100, "stderr": "E: Could not get lock /var/lib/dpkg/lock-frontend. It is held by process 6985 (apt-get)\nE: Unable to acquire the dpkg frontend lock (/var/lib/dpkg/lock-frontend), is another process using it?\n", "stderr_lines": ["E: Could not get lock /var/lib/dpkg/lock-frontend. It is held by process 6985 (apt-get)", "E: Unable to acquire the dpkg frontend lock (/var/lib/dpkg/lock-frontend), is another process using it?"], "stdout": "", "stdout_lines": []}
changed: [Ex2]
changed: [10.129.0.25]

TASK [Start tuned service] *****************************************************
ok: [Ex2]
ok: [10.129.0.25]

TASK [Enable and start tuned-profile service (for RHEL/CentOS 8+)] *************
skipping: [Ex2]
skipping: [10.129.0.25]

TASK [Verify tuned service status] *********************************************
ok: [Ex2]
ok: [10.129.0.25]

TASK [Display tuned status] ****************************************************
ok: [Ex2]
ok: [10.129.0.25]

TASK [Show tuned status] *******************************************************
ok: [10.129.0.25] => {
    "msg": "Current active profile: virtual-guest"
}
ok: [Ex2] => {
    "msg": "Current active profile: virtual-guest"
}

PLAY RECAP *********************************************************************
10.129.0.25                : ok=7    changed=2    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
10.129.0.33                : ok=2    changed=1    unreachable=0    failed=1    skipped=2    rescued=0    ignored=0   
Ex1                        : ok=2    changed=1    unreachable=0    failed=1    skipped=2    rescued=0    ignored=0   
Ex2                        : ok=7    changed=2    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0  
```

3. Изменить приветствие системы (motd) при входе на любое другое. Пожалуйста, в этом задании используйте переменную для задания приветствия. Переменную можно задавать любым удобным способом.

playbook
```YAML
# configure_motd_with_vars.yml
---
- name: Configure MOTD using external variables
  hosts: all
  become: yes
  vars_files:
    - vars/motd_vars.yml

  tasks:
    - name: Create custom MOTD with variables
      ansible.builtin.template:
        src: motd.j2
        dest: /etc/motd
        owner: root
        group: root
        mode: '0644'

    - name: Reload SSH service to apply new MOTD
      ansible.builtin.systemd:
        name: ssh
        state: reloaded

    - name: Verify MOTD is readable
      ansible.builtin.command: cat /etc/motd
      register: motd_check

    - name: Display current MOTD
      ansible.builtin.debug:
        msg: "Текущий MOTD:\n{{ motd_check.stdout }}"
```

output log

```
PLAY [Configure MOTD using external variables] *********************************

TASK [Gathering Facts] *********************************************************
ok: [Ex1]
ok: [Ex2]

TASK [Create custom MOTD with variables] ***************************************
changed: [Ex2]
changed: [Ex1]

PLAY RECAP *********************************************************************
Ex1                        : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
Ex2                        : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

variables

```YAML
# vars/motd_vars.yml
motd_message: "🚀 Добро пожаловать на оптимизированный сервер!"
company_name: "Моя Компания"
support_contact: "support@company.com"
```

template

```YAML
##################################################
# {{ motd_message }}
#
# Сервер: {{ ansible_hostname }}
# ОС: {{ ansible_distribution }} {{ ansible_distribution_version }}
# Компания: {{ company_name }}
# Контакт поддержки: {{ support_contact }}
# Дата обновления: {{ ansible_date_time.iso8601 }}
##################################################
```

screenshot
[Скриншот-1](https://github.com/Morfey29/sdvps-homeworks-7-01.part_2/edit/main/README.MD#:~:text=img1.png)

### Задание 2

**Выполните действия, приложите файлы с модифицированным плейбуком и вывод выполнения.**

Модифицируйте плейбук из пункта 3, задания 1. В качестве приветствия он должен установить IP-адрес и hostname управляемого хоста, пожелание хорошего дня системному администратору.

modified template
```YAML
##################################################
|
          ╔═════════════════════════════════════════════════════════════════════════════════════════════════════╗
          ║                    ДОБРО ПОЖАЛОВАТЬ                                                                 ║
          ║  на сервер {{ ansible_hostname | upper }}{{ ' ' * (25 - ansible_hostname|length) }}               ║
          ╚═════════════════════════════════════════════════════════════════════════════════════════════════════╝

          📊 ИНФОРМАЦИЯ О СИСТЕМЕ:
          ────────────────────────────────────────────────────────────────
          🖥️  Хост: {{ ansible_hostname }}
          🌐 IP-адрес: {{ ansible_default_ipv4.address }}
          🐧 ОС: {{ ansible_distribution }} {{ ansible_distribution_version }}
          ⚙️  Архитектура: {{ ansible_architecture }}
          💾 ОЗУ: {{ (ansible_memtotal_mb / 1024) | round(1) }} GB
          🔄 Время работы: {{ (ansible_uptime_seconds // 3600) | int }}ч {{ ((ansible_uptime_seconds % 3600) // 60) | int }}м
          📅 Дата: {{ ansible_date_time.date }}
          🕐 Время: {{ ansible_date_time.time }}

          ════════════════════════════════════════════════════════════════
          Будьте внимательны при выполнении операций на сервере!
          "Хорошего дня, системный администратор! 🚀"
          ════════════════════════════════════════════════════════════════
##################################################
```

screenshot
[Скриншот-2](https://github.com/Morfey29/sdvps-homeworks-7-01.part_2/edit/main/README.MD#:~:text=img2.png)


### Задание 3

**Выполните действия, приложите архив с ролью и вывод выполнения.**

Ознакомьтесь со статьёй [«Ansible - это вам не bash»](https://habr.com/ru/post/494738/), сделайте соответствующие выводы и не используйте модули **shell** или **command** при выполнении задания.

Создайте плейбук, который будет включать в себя одну, созданную вами роль. Роль должна:

1. Установить веб-сервер Apache на управляемые хосты.
2. Сконфигурировать файл index.html c выводом характеристик каждого компьютера как веб-страницу по умолчанию для Apache. Необходимо включить CPU, RAM, величину первого HDD, IP-адрес.
Используйте [Ansible facts](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html) и [jinja2-template](https://linuxways.net/centos/how-to-use-the-jinja2-template-in-ansible/). Необходимо реализовать handler: перезапуск Apache только в случае изменения файла конфигурации Apache.
4. Открыть порт 80, если необходимо, запустить сервер и добавить его в автозагрузку.
5. Сделать проверку доступности веб-сайта (ответ 200, модуль uri).

В качестве решения:
- предоставьте плейбук, использующий роль;

playbook
```YAML
# playbook.yml
---
- name: Deploy Apache web server with system info
  hosts: serv
  become: yes
  roles:
    - role: apache_server
```

- разместите архив созданной роли у себя на Google диске и приложите ссылку на роль в своём решении;
```
https://drive.google.com/file/d/1zmdyXENwMGpnGYN6irKCC_nyjlkCEBxW/view?usp=sharing
```

- предоставьте скриншоты выполнения плейбука;

output log

```
ansible-playbook -i ./apache_server/hosts.ini ./apache_server/playbook.yml | tee "web_serv_deploy_log$(date +%Y%m%d_%H%M%S).log"

PLAY [Deploy Apache web server with system info] *******************************

TASK [Gathering Facts] *********************************************************
ok: [Ex1]
ok: [Ex2]

TASK [apache_server : Gather system facts] *************************************
ok: [Ex1]
ok: [Ex2]

TASK [apache_server : Install Apache web server] *******************************
ok: [Ex1]
ok: [Ex2]

TASK [apache_server : Ensure Apache is running and enabled] ********************
ok: [Ex1]
ok: [Ex2]

TASK [apache_server : Allow HTTP traffic on port 80 via iptables] **************
ok: [Ex1]
ok: [Ex2]

TASK [apache_server : Save iptables rules] *************************************
changed: [Ex1]
changed: [Ex2]

TASK [apache_server : Create custom index.html from template] ******************
changed: [Ex1]
changed: [Ex2]

TASK [apache_server : Ensure proper permissions on web root] *******************
changed: [Ex1]
changed: [Ex2]

TASK [apache_server : Wait for Apache to start] ********************************
ok: [Ex1]
ok: [Ex2]

TASK [apache_server : Test website availability] *******************************
ok: [Ex1]
ok: [Ex2]

TASK [apache_server : Display website test result] *****************************
ok: [Ex1] => {
    "msg": "Веб-сайт успешно доступен. Статус: 200"
}
ok: [Ex2] => {
    "msg": "Веб-сайт успешно доступен. Статус: 200"
}

PLAY RECAP *********************************************************************
Ex1                        : ok=11   changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
Ex2                        : ok=11   changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

screenshot
[Скриншот-3](https://github.com/Morfey29/sdvps-homeworks-7-01.part_2/edit/main/README.MD#:~:text=img3.png)


- предоставьте скриншот браузера, отображающего сконфигурированный index.html в качестве сайта.

screenshot
![Скриншот-4](https://github.com/Morfey29/sdvps-homeworks-7-01.part_2/edit/main/README.MD#:~:text=img4.png)
