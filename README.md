# CrowdSec + PostgreSQL HA cu Patroni + Keepalived
## Arhitectura

```
                    VIP: 10.10.100.21
                         |
          +--------------+--------------+
          |                             |
       node1: 10.10.100.171          node2: 10.10.100.172
   [PostgreSQL PRIMARY]    <-->  [PostgreSQL REPLICA]
   [Patroni LEADER]     Raft    [Patroni FOLLOWER]
   [CrowdSec LAPI]               [CrowdSec LAPI]
   [Keepalived MASTER]           [Keepalived BACKUP]
          |                             |
          +---- streaming replication --+
```

### Failover automat (Patroni)
1. Primary cade → Patroni detecteaza pierderea leaderului (TTL 30s)
2. Patroni promoveaza automat replica la primary (`pg_promote`)
3. Keepalived detecteaza ca noul nod are `/leader` HTTP 200 → muta VIP-ul
4. CrowdSec si clientii se reconecteaza via VIP transparent

**Fara interventie manuala necesara.**

---

## Structura proiect

```
.
├── ansible.cfg
├── site.yml                   # Playbook principal
├── check_status.yml           # Verificare stare cluster
├── patroni_ops.yml            # Operatii Patroni (switchover, reinit, restart)
├── inventory/
│   └── hosts.yml
├── group_vars/
│   └── all.yml                # Parametri globali (IP-uri, parole, versiuni)
└── roles/
    ├── common/                # Configurare OS, nftables, /etc/hosts
    ├── postgresql/            # PostgreSQL + Patroni (Raft DCS, failover automat)
    ├── crowdsec/              # CrowdSec cu PostgreSQL backend via VIP
    └── keepalived/            # VIP - urmareste liderul Patroni via REST API
```

---

## Inainte de deploy

### 1. Schimba TOATE parolele in `group_vars/all.yml`
```yaml
postgresql_superuser_password: "P0stgr3sSuper@Pass!"   # Schimba!
postgresql_replication_password: "R3plic@t0rP@ss!"     # Schimba!
postgresql_crowdsec_password:    "CrowdS3cDB@Pass!"    # Schimba!
keepalived_auth_pass:            "Str0ngK33p@liv3d!"   # Schimba!
```

### 2. Verifica interfata de retea
Implicit `eth0`. Modifica `vip_interface` in `group_vars/all.yml` daca e diferit.

### 3. Acces SSH root
```bash
ssh-copy-id root@10.10.100.171
ssh-copy-id root@10.10.100.172
```

### 4. Dependinte Ansible
```bash
pip install ansible
ansible-galaxy collection install community.postgresql community.general
```

---

## Deploy

```bash
# Testare conectivitate
ansible all -m ping

# Deploy complet
ansible-playbook site.yml

# Verificare stare dupa deploy
ansible-playbook check_status.yml
```

---

## Verificari post-deploy

### Stare cluster Patroni
```bash
# Afiseaza tabelul cu noduri, rol si lag
ansible-playbook patroni_ops.yml --tags status

# Sau direct pe orice nod:
patronictl -c /etc/patroni/patroni.yml list
```

Output asteptat:
```
+ Cluster: crowdsec-ha ------+----+-----------+
| Member | Host            | Role    | State   | TL | Lag in MB |
+--------+-----------------+---------+---------+----+-----------+
| node1  | 10.10.100.171:5432 | Leader  | running |  1 |           |
| node2  | 10.10.100.172:5432 | Replica | running |  1 |         0 |
+--------+-----------------+---------+---------+----+-----------+
```

### Test failover automat
```bash
# Opreste Patroni pe primary (node1) - simuleaza crash
systemctl stop patroni   # pe node1

# In ~30s, node2 devine primary automat
# Verifica VIP pe node2:
ip addr show eth0 | grep 10.10.100.21

# Verifica cluster:
patronictl -c /etc/patroni/patroni.yml list   # pe node2
```

### Switchover planificat (fara downtime)
```bash
ansible-playbook patroni_ops.yml --tags switchover
```

### Reinitializare replica dupa incident
```bash
ansible-playbook patroni_ops.yml --tags reinit
```

---

## Porturi folosite

| Port | Protocol | Serviciu                     |
|------|----------|------------------------------|
| 5432 | TCP      | PostgreSQL                   |
| 8008 | TCP      | Patroni REST API             |
| 2380 | TCP      | Patroni Raft DCS             |
| 8080 | TCP      | CrowdSec LAPI                |
| 6060 | TCP      | CrowdSec Metrics (localhost) |
| 112  | IP       | Keepalived VRRP              |

