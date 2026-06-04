# Ansible Ubuntu Update

Автоматизация обновления пакетов Ubuntu-серверов с помощью Ansible.

Репозиторий содержит playbook, который:
- обновляет индекс пакетов (`apt update`);
- устанавливает все доступные обновления (`apt upgrade` / `dist-upgrade`);
- выполняет очистку неиспользуемых пакетов;
- проверяет необходимость перезагрузки;
- при необходимости инициирует автоматический reboot.

## Структура репозитория

```text
.
├── inventory.ini        # Инвентори с перечнем хостов
├── update-ubuntu.yml    # Основной playbook обновления
└── README.md            # Описание проекта
```

При желании можно вынести логику в роль `roles/update_ubuntu`, но базовый вариант умышленно оставлен максимально простым для быстрого внедрения.

## Требования

- Ansible (рекомендуется актуальная стабильная версия);
- SSH-доступ к целевым Ubuntu-хостам;
- Права `sudo` на удалённых серверах (без интерактивного ввода пароля либо с `--ask-become-pass`).

Ansible можно установить через системный пакетный менеджер или PPA `ppa:ansible/ansible` для получения более свежей версии, если это необходимо в окружении [web:7].

## Инвентори

Пример `inventory.ini`:

```ini
[ubuntu_servers]
server1 ansible_host=192.0.2.10
server2 ansible_host=192.0.2.11

[ubuntu_servers:vars]
ansible_user=ubuntu
ansible_become=true
```

Группу и переменные можно адаптировать под свою инфраструктуру (другой пользователь, свой SSH-порт, ключи и т.п.).

## Playbook

Пример `update-ubuntu.yml`:

```yaml
***
- name: Update Ubuntu packages
  hosts: all
  become: true
  gather_facts: true

  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Upgrade all packages
      ansible.builtin.apt:
        upgrade: dist

    - name: Autoremove unused packages
      ansible.builtin.apt:
        autoremove: true
        autoclean: true

    - name: Check if reboot is required
      ansible.builtin.stat:
        path: /var/run/reboot-required
      register: reboot_required

    - name: Reboot host if required
      ansible.builtin.reboot:
        msg: "Reboot initiated by Ansible after package upgrade"
        connect_timeout: 5
        reboot_timeout: 600
        pre_reboot_delay: 5
      when: reboot_required.stat.exists
```

Такой подход соответствует типовым практикам обновления Ubuntu через модуль `apt` и использование файла `reboot-required` для принятия решения о перезагрузке [web:6][web:17].

### Более «консервативный» режим

Если нежелательно использовать полный `dist-upgrade`, можно заменить задачу обновления пакетов на:

```yaml
    - name: Upgrade packages safely
      ansible.builtin.apt:
        upgrade: yes
```

Это ближе к «безопасному» обновлению без агрессивного изменения зависимостей [web:17].

## Запуск

Обновление всех хостов из инвентори:

```bash
ansible-playbook -i inventory.ini update-ubuntu.yml
```

Обновление отдельного хоста или группы:

```bash
ansible-playbook -i inventory.ini update-ubuntu.yml --limit server1
ansible-playbook -i inventory.ini update-ubuntu.yml --limit ubuntu_servers
```

Если требуется ввод пароля для `sudo`:

```bash
ansible-playbook -i inventory.ini update-ubuntu.yml --ask-become-pass
```

## Рекомендуемый workflow

- Сначала протестировать playbook на одном тестовом сервере.
- Затем применить его на небольшой группе боевых хостов.
- После валидации — расширить на остальные сервера.
- Регулярно запускать playbook (например, по cron/CI) для поддержания актуальности пакетов, аналогично ручному `apt update && apt upgrade`.

## Кастомизация

Возможные направления доработки:

- вынести задачи в отдельную роль `roles/update_ubuntu`;
- добавить хэндлеры/уведомления (в Slack/Telegram) по результатам обновления;
- интегрировать с CI/CD (GitHub Actions, GitLab CI и др.);
- добавить теги (`tags: update`, `tags: reboot`) для более гибкого запуска;
- разделить playbook на «dry-run» и «production» варианты.

Pull Request’ы и улучшения приветствуются.
