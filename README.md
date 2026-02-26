# kubeadm_node_prep

Внутренний Ansible playbook для стандартизированной подготовки Ubuntu 22.04 нод под Kubernetes (kubeadm + containerd) без ручной возни и “зоопарка” команд.

Важно: это именно подготовка ОС. Дальше вы отдельно выполняете `kubeadm init` (control-plane) и `kubeadm join` (workers).

## Зачем это нужно

Подготовка ноды под kubeadm почти всегда одинаковая, но каждый раз всплывают грабли:
- swap забыли выключить
- sysctl не применили
- containerd не настроили под systemd cgroups
- repo Kubernetes не добавился / DNS тупит / APT висит
- версии пакетов пляшут

Этот playbook делает один понятный, воспроизводимый “bootstrap” шаг.

Подходит для:
- lab / homelab кластеров
- bare-metal
- облачных VM
- быстрого развертывания тестовых окружений

## Как это работает

Playbook делает следующее:
- форсирует IPv4 для APT (если IPv6 нестабилен)
- ставит временный DNS фикс (если резолв сломан и pkgs.k8s.io висит)
- ставит базовые пакеты
- отключает swap (сейчас + правит fstab)
- загружает модули ядра: overlay, br_netfilter (и включает автозагрузку)
- применяет sysctl для Kubernetes networking
- (опционально) выключает UFW
- ставит containerd, генерирует config.toml и включает SystemdCgroup
- добавляет репозиторий Kubernetes через pkgs.k8s.io
  - ключ скачивается как ASCII `.asc` (без `gpg --dearmor`)
- устанавливает kubelet/kubeadm/kubectl и ставит hold
- включает chrony и rsyslog
- ставит cri-tools и настраивает crictl под containerd

После выполнения:
- control-plane: `kubeadm init`
- workers: `kubeadm join ...`

## Требования

- Ansible 2.14+
- Ubuntu 22.04 на целевых хостах
- SSH доступ + sudo
- интернет на нодах (APT + pkgs.k8s.io)

## Быстрый старт

1) Создайте inventory (реальный inventory не коммитьте):

[k8s]
control-1 ansible_host=10.0.0.10
worker-1  ansible_host=10.0.0.11
worker-2  ansible_host=10.0.0.12

2) Запустите playbook:

ansible-playbook -i inventory.ini playbooks/prepare-kubeadm-ubuntu2204.yml

3) Дальше kubeadm:

control-plane:
kubeadm init

workers:
kubeadm join ...
## Настройки

Основные параметры, которые обычно меняют:
- k8s_minor: minor-ветка Kubernetes репозитория (например v1.31)
- disable_firewall: отключать ли UFW
- dns_servers: DNS сервера для быстрого фикса резолва

Рекомендуется выносить их в group_vars/all.yml, а в репе держать только пример (all.example.yml).

## Что лежит в репозитории

- playbooks/prepare-kubeadm-ubuntu2204.yml — основной playbook
- inventory.example.ini — пример inventory (без ваших IP/hostname)
- group_vars/all.example.yml — пример переменных
- README.md — этот файл

## Практика безопасности

- не коммитьте реальные IP/hostnames (inventory)
- не храните приватные DNS/прокси/зеркала в публичной репе
- любые токены/пароли — только через Ansible Vault или секреты CI/CD
- если секрет уже попал в историю git — считайте его скомпрометированным и меняйте его
