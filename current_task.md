Создай два Ansible playbook:

1) bootstrap.yml — минимальный bootstrap для нового сервера:
- playbook запускается от дефолтного пользователя (например ubuntu/vlavlamat/skufphp) с become: true
- создаёт пользователя ansible с /bin/bash и home dir
- добавляет ansible в группу sudo
- добавляет sudo rule в /etc/sudoers.d/ansible: "ansible ALL=(ALL) NOPASSWD:ALL"
- добавляет публичный ключ в /home/ansible/.ssh/authorized_keys
  публичный ключ задаётся переменной ansible_public_key
- корректные права: ~/.ssh 700, authorized_keys 600
- использовать стандартные модули user, authorized_key, file, copy/lineinfile
- всё должно быть идемпотентно

2) hardening.yml — базовые настройки безопасности (запускать после bootstrap от пользователя ansible):
- настроить /etc/ssh/sshd_config через lineinfile/blockinfile:
  PasswordAuthentication no
  PermitRootLogin no
  PubkeyAuthentication yes
- НЕ запрещать вход по ключу для дефолтного пользователя
- опционально добавить AllowUsers (через переменные allow_users), например: "ansible" + дефолтный пользователь
- перезапустить sshd безопасно (например, validate конфиг перед рестартом)
- включить ufw и разрешить входящий ssh только с tailscale0 (если интерфейс существует), иначе оставить ssh открытым
- всё идемпотентно

В конце добавь README.md инструкции для этих playbook:
1) как запустить bootstrap.yml с дефолтного пользователя
2) как затем запускать hardening.yml от пользователя ansible
3) как перед применением проверить, что ключевой доступ работает (открыть второй ssh-сеанс).