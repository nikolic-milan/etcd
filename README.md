# etcd
An etcd demo, developed for the course Database Systems for Big Data.
# Setup Raspberry Pi
Since didn't have an extra display to connect to my device I used the Lite version of the Raspberry Pi OS 32-bit.
Instalation was done through the Raspberry Pi Imager. On both a did additional configuration to enable SSH, so that I can connect to them remotely. And on one I enabled Wi-Fi and configured it to automatically connect od bootup. The other one was connected via ethernet cable.
I used an app called Zenmap to discover all devices connected to my network. The comman used for this is:  
```
npam -sn 192.168.2.0/24
```
Finnaly in terminal:  
```
ssh username@<ip-address-of-device>
```
# Install etcd on a Raspberry Pi
Since etcd is not supported on Raspberry Pi we will use an unoficial build from https://github.com/robertojrojas/kubernetes-the-hard-way-raspberry-pi/blob/master/docs/03-etcd.md.
Per the instruction from that github page:
```
wget https://raw.githubusercontent.com/robertojrojas/kubernetes-the-hard-way-raspberry-pi/master/etcd/etcd-3.1.5-arm.tar.gz   
tar -xvf etcd-3.1.5-arm.tar.gz 
sudo mv etcd-3.1.5-arm/etcd* /usr/bin/
rm -rf etcd-3.1.5-arm*
sudo mkdir -p /var/lib/etcd
```

# Install etcd on macOS
Since I wanted at least 3 nodes uin my cluster and I only have 2 Raspberry Pis I needed to install etcd on my Macbook Air M1.
Luckaly that is a simpler proces, just use Homebrew:  
```
brew install etcd
```

# Setup a cluster
etcd has multiple ways of setting up a cluster:
1. Static
2. etcd Discovery
3. DNS Discovery
Since I know the exact IP address of all my devices I will setup the cluster manualy.  

| Name         | IP address |
|--------------|:-----:|
| rasPi1 |  192.168.2.9 |
| rasPi2      |  192.168.2.11 |
| macOS      |  192.168.2.6 |

## On Raspberry Pi
To configure the raspbery pi and start a node we need to do some configuration. We will use the static method since we know our ip addresses.
```
cat > etcd.service <<"EOF"
[Unit] 
Description=etcd  
Documentation=https://github.com/coreos  

[Service]  
Environment=ETCD_UNSUPPORTED_ARCH=arm  
Type=notify  
ExecStart=/usr/bin/etcd --name ETCD_NAME \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --initial-advertise-peer-urls http://INTERNAL_IP:2380 \  
  --listen-peer-urls http://INTERNAL_IP:2380 \  
  --listen-client-urls http://INTERNAL_IP:2379,http://127.0.0.1:2379 \  
  --advertise-client-urls http://INTERNAL_IP:2379 \  
  --initial-cluster-token etcd-cluster-1 \  
  --initial-cluster rasPi1=http://192.168.2.9:2380,rasPi2=http://192.168.2.11:2380,macOS=http://192.168.2.6:2380 \  
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
```
ETCD_NAME=<name> (in my case rasPi1/rasPi2)
INTERNAL_IP=<ip-address-of-device>
```
```
sed -i s/INTERNAL_IP/${INTERNAL_IP}/g etcd.service
sed -i s/ETCD_NAME/${ETCD_NAME}/g etcd.service
sudo mv etcd.service /etc/systemd/system/
```
```
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```
## On macOS
It is a similar process on macOS, but since etcd is normaly supported, it's only one command
```
etcd --name macOS --initial-advertise-peer-urls http://192.168.2.6:2380 \
  --listen-peer-urls http://192.168.2.6:2380 \
  --listen-client-urls http://192.168.2.6:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://192.168.2.6:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster rasPi1=http://192.168.2.9:2380,rasPi2=http://192.168.2.11:2380,macOS=http://192.168.2.6:2380 \
  --initial-cluster-state new
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
```
