# Ansible: первое приложение на GitHub

Основной репозиторий: https://github.com/gondrr/SPO (ветки `main`, `dev`).
Автоматизация выполняется через GitHub Actions.

## Структура и этапы

- `inventory/` — настоящий Git submodule https://github.com/gondrr/inventory.
- `group_vars/k8s.yml` — настройки конфигурации для группы серверов.
- `playbook.yml` — SSH ping и отдельный локальный рендер конфигураций.
- `requirements.yml` — роль https://github.com/gondrr/first-ansible-role,
  тег `v1.0.1`, и коллекция `ansible.utils` для проверки IP.
- `.github/workflows/ansible.yml` — workflow CI/CD для обоих этапов.

Этап 1 реализован первым play: `ansible.builtin.ping` проверяет SSH и Python
на 10.160.161.184 и 10.160.161.185 (это не ICMP ping).
Этап 2 использует отдельные репозитории, роль, Jinja2 и артефакты.

## Локальный запуск (Linux / WSL)

```bash
git clone --recurse-submodules https://github.com/gondrr/SPO.git
cd SPO
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
ansible-galaxy role install -r requirements.yml -p .ansible/roles
ansible-galaxy collection install -r requirements.yml -p .ansible/collections
ansible-playbook playbook.yml --syntax-check
ansible-playbook playbook.yml --tags render -e ansible_python_interpreter="$(which python)"
ANSIBLE_REMOTE_USER=your_user ansible-playbook playbook.yml --tags ping --private-key ~/.ssh/id_ed25519
```

На контроллере Ansible нужен Linux; обычный Windows PowerShell не поддерживается.
Добавьте `.venv/` в локальные исключения, если используете локальный venv.
Для существующего клона: `git submodule update --init --recursive`.

## GitHub Actions

На push в `main`/`dev`, PR и ручном запуске две независимые prepare jobs
скачивают роли и коллекции. Через upload/download-artifact зависимости
переходят в render job. Она проверяет синтаксис playbook, создаёт локальные
конфиги, проверяет их через `nginx -t` и публикует артефакт `nginx-configs`:

```text
k8s-01/nginx.conf
k8s-02/nginx.conf
```

Настройки портов, server_name, логов и корня сайта берутся из group_vars;
на целевые машины конфиги не устанавливаются.

## Включение SSH-проверки

Адреса 10.x доступны из частной сети. В Settings → Actions → Runners
подключите Linux runner с Python 3.12 и меткой `ansible-network`, имеющий
SSH-доступ к обоим серверам и доступ к GitHub/PyPI.

В Settings → Secrets and variables → Actions задайте:

| Тип | Имя | Значение |
| --- | --- | --- |
| Variable | `ANSIBLE_REMOTE_USER` | SSH-пользователь серверов |
| Secret | `ANSIBLE_SSH_PRIVATE_KEY` | Приватный SSH-ключ без passphrase для CI |
| Secret | `ANSIBLE_KNOWN_HOSTS` | Проверенные SSH host keys обоих IP в формате known_hosts |
| Variable | `ANSIBLE_ENABLED` | `true`, когда runner и доступ настроены |

Проверяйте fingerprints host keys через доверенный канал администратора.
Публичный ключ должен быть в authorized_keys пользователя на обоих серверах,
а `/usr/bin/python3` должен быть установлен. Root и sudo для ping не нужны.
До включения `ANSIBLE_ENABLED` job ping явно пропускается; render работает.
На PR ping не запускается. После включения каждый push проверяет серверы.

## Обновление зависимостей

Инвентарь: измените отдельный репозиторий, затем выполните
`git submodule update --remote inventory` и закоммитьте новый указатель.
Роль: создайте новый тег в репозитории роли и обновите `version` в requirements.

Документация: [Galaxy](https://docs.ansible.com/projects/ansible/latest/cli/ansible-galaxy.html),
[self-hosted runners](https://docs.github.com/en/actions/reference/runners/self-hosted-runners).
