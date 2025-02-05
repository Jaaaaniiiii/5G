**Pre-requisite**
-
|  Server  |  CPU  | RAM  | Function |      IP       |      OS      |
|:--------:|:-----:|:----:|:--------:|:-------------:|:------------:|
| Server-1 | 4 CPU | 4 GB | Open5GS  | 192.168.22.12 | Ubuntu 22.04 |
| Server-2 | 4 CPU | 2 GB | EURANSIM | 192.168.22.13 | Ubuntu 22.04 |

**Server-1**
-
**1. Install MongoDB**
- Add mongoDB package
```bash
sudo apt update
sudo apt install gnupg curl
curl -fsSL https://pgp.mongodb.com/server-6.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-6.0.gpg --dearmor
echo &#34;deb [arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-6.0.gpg] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/6.0 multiverse&#34; | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
```

- Install mongoDB
```bash
sudo apt update
sudo apt install -y mongodb-org
sudo systemctl start mongod
sudo systemctl enable mongod
```

**2. Install Open5GS**
```bash
sudo add-apt-repository ppa:open5gs/latest
sudo apt update
sudo apt install open5gs
```

**3. Install WebUI Open5GS**
- Add nodejs package
```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
NODE_MAJOR=20
echo &#34;deb [arch=amd64 signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main&#34; | sudo tee /etc/apt/sources.list.d/nodesource.list
```

- Install nodejs
```bash
sudo apt update
sudo apt install nodejs -y
```

- Install WebUI via bash script
```bash
curl -fsSL https://open5gs.org/open5gs/assets/webui/install | sudo -E bash -
```

**4. Configure Open5GS**
- Edit amf config file
```bash
sudo nano /etc/open5gs/amf.yaml
```
```yaml
amf:
  sbi:
    server:
      - address: 127.0.0.5
        port: 7777
    client:
#      nrf:
#        - uri: http://127.0.0.10:7777
      scp:
        - uri: http://127.0.0.200:7777
  ngap:
    server:
      - address: 192.168.22.12
```

- Restart amf service and see the log
```bash
sudo systemctl restart open5gs-amfd
sudo tail -f /var/log/open5gs/amf.log
```

- Edit upf config file
```bash
sudo nano /etc/open5gs/upf.yaml
```
```yaml
upf:
  pfcp:
    server:
      - address: 127.0.0.7
    client:
#      smf:     #  UPF PFCP Client try to associate SMF PFCP Server
#        - address: 127.0.0.4
  gtpu:
    server:
      - address: 192.168.22.12
```

- Restart upf service and see the log
```bash
sudo systemctl restart open5gs-upfd
sudo tail -f /var/log/open5gs/upf.log
```

**5. NAT Port Forwarding**
- Add nat rule
```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o ens33 -j MASQUERADE
sudo systemctl stop ufw
sudo iptables -I FORWARD 1 -j ACCEPT
```

- Save rule to file config
```bash
sudo apt install iptables-persistent -y
sudo iptables-save &gt; /etc/iptables/rules.v4
```

**6. Register Subscriber Information**
![image](https://hackmd.io/_uploads/r1CPhZ0lkl.png)
Access to http://localhost:9999 and login with this credential:
```
username : admin
password : 1423
```

![image](https://hackmd.io/_uploads/BJQr3WCeyg.png)
Go to subscriber menu -&gt; click + button -&gt; fill IMSI, subs key, usim type, operator key -&gt; click save button.

**Server-2**
-
**1. Install UERANSIM**
- Install dependency
```bash
sudo apt update &amp;&amp; sudo apt upgrade -y
sudo apt install make g++ libsctp-dev lksctp-tools iproute2
sudo snap install cmake --classic
```

- Clone UERANSIM project and install it
```bash
git clone https://github.com/aligungr/UERANSIM.git
cd UERANSIM
make
```

**2. Setup gNB**
- Edit gNB config file
```bash
nano config/open5gs-gnb.yaml
```
```yaml
linkIp: 192.168.22.13
ngapIp: 192.168.22.13
gtpIp: 192.168.22.13

amfConfigs:
  - address: 192.168.22.12
    port: 38412
```

- Start gNB with open5gs-gnb.yaml config file
```bash
./build/nr-gnb -c config/open5gs-gnb.yaml
```

**3. Setup UE**
- Edit gNB config file
```bash
nano config/open5gs-ue.yaml
```
```yaml
supi: &#39;imsi-999700000000001&#39;
mcc: &#39;999&#39;
mnc: &#39;70&#39;

key: &#39;465B5CE8B199B49FAA5F0A2EE238A6BC&#39;
op: &#39;E8ED289DEBA952E4283B54E88E6183CA&#39;
opType: &#39;OPC&#39;
amf: &#39;8000&#39;
imei: &#39;356938035643803&#39;
imeiSv: &#39;4370816125816151&#39;

gnbSearchList:
  - 192.168.22.13
```

- Start gNB with open5gs-ue.yaml config file
```bash
sudo ./build/nr-ue -c config/open5gs-ue.yaml
```

**4. Test 5G Network**
- Ping via uesimtun0 interface
```bash
ping -I uesimtun0 google.com
```

- Curl command bind direcly to uesimtun0
```bash
sudo apt install curl -y
curl --interface uesimtun0 -X GET &#34;https://httpbin.org/get&#34;
