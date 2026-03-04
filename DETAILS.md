## 1) Теги, которые тебе нужны

* `tag:infra` — твой MacBook (и/или будущий CI runner)
* `tag:dev` — учебные VM (Swarm dev)
* `tag:prod` — прод VM (Swarm prod)
* (опционально) `tag:admin` — если хочешь отдельную “админскую” роль

---

## 2) Включи MagicDNS (очень рекомендую)

Тогда ты сможешь делать:

```bash
ssh ubuntu@prod-manager-1
ansible -i inventory.ini prod -m ping
```

вместо запоминания `100.x.x.x`.

---

## 3) ACL политика (простой и безопасный старт)

Идея:
✅ Mac (`tag:infra`) может ходить в dev и prod по SSH/Ansible
✅ dev-ноды могут общаться между собой (для Swarm)
✅ prod-ноды могут общаться между собой (для Swarm)
❌ dev не может ходить в prod

Swarm порты между нодами (важно):

* `2377/tcp` (management)
* `7946/tcp+udp` (gossip)
* `4789/udp` (overlay VXLAN)

### Пример ACL (концепт)

* `tag:infra` → `tag:dev:*` (полный доступ)
* `tag:infra` → `tag:prod:22,2377,7946,4789` (достаточно для управления и диагностики)
* `tag:dev` ↔ `tag:dev:2377,7946,4789` (кластер внутри dev)
* `tag:prod` ↔ `tag:prod:2377,7946,4789` (кластер внутри prod)
* `tag:dev` → `tag:prod:*` запрещено (по умолчанию, если не разрешать)

Если хочешь — могу дать готовый `tailnet policy file` целиком (JSON) под твою схему.

---

## 4) Как это применить на практике

### На MacBook

* поставить Tailscale
* пометить устройство как `tag:infra` (в админке Tailscale)

### На VM

* поставить Tailscale
* пометить VM как `tag:dev` или `tag:prod`

---

## 5) Какой IP использовать для Swarm

Критически важно: **Swarm должен жить поверх Tailscale IP**, иначе при смене VPS/провайдера и NAT могут быть сюрпризы.

На manager’е dev:

```bash
docker swarm init --advertise-addr <tailscale-ip-manager-dev>
```

На prod manager’е:

```bash
docker swarm init --advertise-addr <tailscale-ip-manager-prod>
```

И join делать тоже на `100.x.x.x:2377`.

---

## 6) Как будет выглядеть Ansible inventory

Пример (через MagicDNS):

```ini
[dev]
dev-manager-1
dev-worker-1
dev-worker-2

[prod]
prod-manager-1
prod-worker-1
prod-worker-2

[all:vars]
ansible_user=ubuntu
```

И ты можешь прогонять:

```bash
ansible dev -m ping
ansible prod -m ping
```

---

## 7) Почему это хорошо для твоей “Black Friday rolling migration” идеи

Из твоего плана: добавляем новый VPS, заводим его в Tailscale, джойним в Swarm, мигрируем данные, переключаем DNS.
Если Swarm общается по Tailscale — он вообще “не видит”, что провайдеры разные, и переезды становятся проще. (Это ровно то, что ты описываешь в стратегии.)

---

## Мини-чеклист “правильной” схемы

* [ ] MagicDNS включён
* [ ] Все VM в tailnet и помечены тегами (`dev`/`prod`)
* [ ] MacBook помечен `infra`
* [ ] ACL разрешает Swarm порты только внутри `dev` и внутри `prod`
* [ ] Swarm инициализирован с `--advertise-addr` = Tailscale IP

---
