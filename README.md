# Lab — Namespaces na prática (Debian)
debootstrap + unshare + chroot

## Objetivo
Construir um “mini‑container” sem Docker para entender:
- isolamento por **namespaces** (UTS/PID/NET/MOUNT/IPC/USER)

---

## Pré‑requisitos
- Debian em VM com Internet e `sudo` funcionando
- Espaço livre: ~1–2 GB
- Rootfs do lab: **/debian**

---

## 1) Instalar dependências
```bash
sudo apt update
sudo apt install -y debootstrap util-linux iproute2 procps
```

Valide:
```bash
sudo debootstrap --help | head
unshare --help | head
ip a | head
ps aux | head
```

---

## 2) Criar rootfs com debootstrap
```bash
sudo mkdir -p /debian
sudo chmod 0755 /debian
sudo debootstrap stable /debian http://deb.debian.org/debian
```

Checagens:
```bash
ls /debian | head
test -f /debian/bin/bash && echo "OK: bash existe"
```

---

## 3) Entrar no mini‑container com root mapeado
### Comando padrão (tente sem sudo primeiro)
```bash
unshare --user --map-root-user --mount --uts --ipc --pid --fork chroot /debian bash
```

Se der `Operation not permitted`, use fallback:
```bash
sudo unshare --user --map-root-user --mount --uts --ipc --pid --fork chroot /debian bash
```

Checagem dentro do ambiente:
```bash
whoami
id
```

Você deve ver `root`/`uid=0` **dentro**, mas isso não te transforma em root **no host**.

> Observação: `--map-root-user` depende de `--user` (user namespace).

---

## 4) Montar /proc e /sys (essencial)
**Dentro do ambiente:**
```bash
mount -t proc proc /proc
mount -t sysfs sys /sys
ps aux | head
cat /proc/uptime
```

Por que isso importa?
- `/proc` é pseudo‑filesystem do kernel; `ps/top/uptime/free` leem dele.
- Sem montar, esses comandos falham ou mostram saída incompleta.

---

## 5) Provas de isolamento

### 5.1 UTS namespace (hostname)
**Dentro:**
```bash
hostname
hostname mini-ns
hostname
```
**No host (outro terminal):**
```bash
hostname
```
Dentro muda; no host não muda.

### 5.2 PID namespace (processos)
**Dentro:**
```bash
echo $$
ps -p 1 -o pid,comm,args
ps -ef | head
```
**No host (achar o processo):**
```bash
ps -ef | grep "chroot /debian bash" | grep -v grep
```

### 5.3 Rede: demonstração do `--net` (sem configuração)
A ideia aqui é comparar a saída do `ip a` em dois cenários.

1) Saia do ambiente:
```bash
exit
```

2) Entre **SEM** `--net` e observe interfaces:
```bash
unshare --user --map-root-user --mount --uts --ipc --pid --fork chroot /debian bash
ip a
exit
```

3) Entre **COM** `--net` e observe o isolamento:
```bash
unshare --user --map-root-user --mount --uts --ipc --pid --net --fork chroot /debian bash
ip a
exit
```

Esperado:
- **Sem `--net`**: você enxerga as interfaces do host (mesmo net namespace).
- **Com `--net`**: você enxerga um estado isolado (geralmente só `lo`).

---

## Entrega do aluno (evidências)
Envie prints com comando + output, numerados **E1…E5**:
- **E1**: `whoami` e `id` dentro do ambiente (root mapeado)
- **E2**: `hostname` dentro e `hostname` no host
- **E3**: `ps -p 1 -o pid,comm,args` dentro
- **E4**: `/proc` montado: `cat /proc/uptime` + `ps aux | head` dentro
- **E5**: `ip a` sem `--net` e `ip a` com `--net` (comparação)

---

## Encerramento e limpeza
No host:
```bash
sudo umount -l /debian/proc 2>/dev/null || true
sudo umount -l /debian/sys 2>/dev/null || true
sudo rm -rf /debian
```

---

## Troubleshooting rápido
- `Operation not permitted` ao criar user namespace → use o fallback com `sudo`.
- `ps/top` falhando dentro → confirme `mount -t proc proc /proc`.
- Algo travou → `exit` e desmonte `/debian/{proc,sys}` no host.

