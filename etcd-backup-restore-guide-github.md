# etcdctl ডাউনলোড, Backup এবং Restore এর সম্পূর্ণ ধাপসমূহ

## ধাপ ১: etcdctl ডাউনলোড এবং ইন্সটলেশন

### etcdctl ডাউনলোড করুন

``` bash
# প্রথমে etcd version check করুন

ETCD_VERSION=$(sudo docker ps | grep etcd | grep -oP 'etcd:\K[0-9.]+' | head -1)

# যদি version না পাওয়া যায়, তাহলে default version use করুন

ETCD_VERSION=${ETCD_VERSION:-"3.5.0"}

# etcdctl ডাউনলোড করুন

wget https://github.com/etcd-io/etcd/releases/download/v${ETCD_VERSION}/etcd-v${ETCD_VERSION}-linux-amd64.tar.gz

# ফাইলটি extract করুন

tar -xvf etcd-v${ETCD_VERSION}-linux-amd64.tar.gz

# etcdctl কে system path-এ move করুন

sudo mv etcd-v${ETCD_VERSION}-linux-amd64/etcdctl /usr/local/bin/

# Permission set করুন

sudo chmod +x /usr/local/bin/etcdctl

# Version check করুন

etcdctl version
```

**বাংলা ব্যাখ্যা**: - প্রথমে আমরা etcd-এর version check করি - তারপর
GitHub থেকে matching version-এর etcdctl ডাউনলোড করি - ফাইলটি extract করে
etcdctl কে system path-এ রাখি যাতে anywhere access করতে পারি

## ধাপ ২: Environment Variables সেটআপ

``` bash
# Environment variables set করুন

export ETCDCTL_API=3
export ETCD_CERT="/etc/kubernetes/pki/etcd/server.crt"
export ETCD_KEY="/etc/kubernetes/pki/etcd/server.key"
export ETCD_CACERT="/etc/kubernetes/pki/etcd/ca.crt"
export ETCD_ENDPOINTS="https://127.0.0.1:2379"

# Permanent করার জন্য bashrc-তেও add করুন

echo 'export ETCDCTL_API=3' >> ~/.bashrc
echo 'export ETCD_CERT="/etc/kubernetes/pki/etcd/server.crt"' >> ~/.bashrc
echo 'export ETCD_KEY="/etc/kubernetes/pki/etcd/server.key"' >> ~/.bashrc
echo 'export ETCD_CACERT="/etc/kubernetes/pki/etcd/ca.crt"' >> ~/.bashrc
echo 'export ETCD_ENDPOINTS="https://127.0.0.1:2379"' >> ~/.bashrc

# Environment reload করুন

source ~/.bashrc
```

**বাংলা ব্যাখ্যা**: - `ETCDCTL_API=3` - etcd version 3 use করবে -
Certificate paths গুলো set করি যাতে etcdctl securely connect করতে পারে -
Endpoint set করি যেখানে etcd service running আছে

## ধাপ ৩: Connection Verify করুন

``` bash
# etcd connection check করুন

etcdctl --endpoints=$ETCD_ENDPOINTS   --cert=$ETCD_CERT   --key=$ETCD_KEY   --cacert=$ETCD_CACERT   endpoint health

# Output expected: https://127.0.0.1:2379 is healthy

# etcd members list দেখুন

etcdctl --endpoints=$ETCD_ENDPOINTS   --cert=$ETCD_CERT   --key=$ETCD_KEY   --cacert=$ETCD_CACERT   member list
```

**বাংলা ব্যাখ্যা**: - এই command দিয়ে check করি যে etcd-এর সাথে properly
connect হতে পারছি কিনা - যদি "healthy" দেখায় তাহলে connection
successful

## ধাপ ৪: Backup তৈরি করুন

``` bash
# Backup directory create করুন

sudo mkdir -p /opt/etcd-backups
cd /opt/etcd-backups

# Snapshot backup নিন

etcdctl --endpoints=$ETCD_ENDPOINTS   --cert=$ETCD_CERT   --key=$ETCD_KEY   --cacert=$ETCD_CACERT   snapshot save etcd-snapshot-$(date +%Y-%m-%d-%H-%M-%S).db

# Backup verify করুন

etcdctl --write-out=table snapshot status etcd-snapshot-*.db
```

**বাংলা ব্যাখ্যা**: - প্রথমে backup রাখার জন্য directory create করি -
তারপর etcdctl দিয়ে snapshot তৈরি করি - সবশেষে verify করি যে backup
properly created হয়েছে

## ধাপ ৫: Automated Backup Script তৈরি করুন

``` bash
# Backup script create করুন

sudo nano /usr/local/bin/etcd-backup.sh
```

Script content:

``` bash
#!/bin/bash

# Configuration

BACKUP_DIR="/opt/etcd-backups"
RETENTION_DAYS=7
TIMESTAMP=$(date +%Y-%m-%d-%H-%M-%S)

# Environment variables

export ETCDCTL_API=3
export ETCD_CERT="/etc/kubernetes/pki/etcd/server.crt"
export ETCD_KEY="/etc/kubernetes/pki/etcd/server.key"
export ETCD_CACERT="/etc/kubernetes/pki/etcd/ca.crt"
export ETCD_ENDPOINTS="https://127.0.0.1:2379"

# Create backup directory

mkdir -p $BACKUP_DIR

# Take snapshot

echo "$(date) - Taking etcd snapshot..."
etcdctl --endpoints=$ETCD_ENDPOINTS   --cert=$ETCD_CERT   --key=$ETCD_KEY   --cacert=$ETCD_CACERT   snapshot save $BACKUP_DIR/etcd-snapshot-$TIMESTAMP.db

# Verify snapshot

echo "$(date) - Verifying snapshot..."
etcdctl --write-out=table snapshot status $BACKUP_DIR/etcd-snapshot-$TIMESTAMP.db

# Clean up old backups

echo "$(date) - Cleaning up backups older than $RETENTION_DAYS days..."
find $BACKUP_DIR -name "etcd-snapshot-*.db" -mtime +$RETENTION_DAYS -delete

echo "$(date) - Backup completed: $BACKUP_DIR/etcd-snapshot-$TIMESTAMP.db"
```

**বাংলা ব্যাখ্যা**: - একটি automated script তৈরি করি যাতে regularly
backup নিতে পারি - Retention policy set করি (৭ দিনের পুরানো backup
automatically delete হবে) - প্রতিবার backup নেওয়ার সময় timestamp সহ
filename create হয়

## ধাপ ৬: Script Executable করুন

``` bash
# Script কে executable করুন

sudo chmod +x /usr/local/bin/etcd-backup.sh

# Test run করুন

sudo /usr/local/bin/etcd-backup.sh
```

## ধাপ ৭: CronJob সেটআপ করুন

``` bash
# Crontab edit করুন

sudo crontab -e

# নিচের line add করুন (রাত ২টায় everyday backup নেবে)

0 2 * * * /usr/local/bin/etcd-backup.sh >> /var/log/etcd-backup.log 2>&1

# Cron service restart করুন

sudo systemctl restart cron
```

**বাংলা ব্যাখ্যা**: - CronJob set করি যাতে automatically প্রতিদিন রাত
২টায় backup হয় - সব logs `/var/log/etcd-backup.log`-এ save হয়

## ধাপ ৮: Restore করার প্রস্তুতি

``` bash
# প্রথমে current cluster state check করুন

kubectl get pods --all-namespaces
kubectl get nodes

# Latest backup file identify করুন

BACKUP_FILE=$(ls -t /opt/etcd-backups/etcd-snapshot-*.db | head -1)
echo "Using backup file: $BACKUP_FILE"

# Backup file verify করুন

etcdctl snapshot status $BACKUP_FILE
```

## ধাপ ৯: Cluster Services বন্ধ করুন

``` bash
# kube-apiserver বন্ধ করুন

sudo systemctl stop kube-apiserver

# Wait করুন

sleep 30

# etcd service বন্ধ করুন

sudo systemctl stop etcd

# Verify services stopped

sudo systemctl status kube-apiserver
sudo systemctl status etcd
```

**বাংলা ব্যাখ্যা**: - Restore করার সময় cluster services বন্ধ করতে হয় -
না হলে data corruption হতে পারে

## ধাপ ১০: Current Data Backup নিন

``` bash
# Current etcd data backup নিন (safety purpose)

sudo tar -czf /opt/etcd-backups/etcd-data-before-restore-$(date +%Y-%m-%d).tar.gz -C /var/lib/etcd .

# Current data directory rename করুন

sudo mv /var/lib/etcd /var/lib/etcd-old-backup
```

## ধাপ ১১: Restore করুন

``` bash
# Snapshot থেকে restore করুন

sudo etcdctl snapshot restore $BACKUP_FILE   --data-dir /var/lib/etcd-new

# Restored data কে proper location-এ move করুন

sudo mv /var/lib/etcd-new /var/lib/etcd

# Permission set করুন

sudo chown -R etcd:etcd /var/lib/etcd
# OR kubeadm clusters-এর জন্য

sudo chown -R 1001:1001 /var/lib/etcd
```

**বাংলা ব্যাখ্যা**: - Backup file থেকে new data directory-তে restore
করি - তারপর সেটাকে proper location-এ move করি - Correct permissions set
করি

## ধাপ ১২: Services Start করুন

``` bash
# প্রথমে etcd start করুন

sudo systemctl start etcd

# Wait করুন

sleep 10

# etcd health check করুন

etcdctl --endpoints=$ETCD_ENDPOINTS   --cert=$ETCD_CERT   --key=$ETCD_KEY   --cacert=$ETCD_CACERT   endpoint health

# kube-apiserver start করুন

sudo systemctl start kube-apiserver

# Wait করুন

sleep 30
```

## ধাপ ১৩: Verify Restoration

``` bash
# Cluster status check করুন

kubectl cluster-info

# Nodes check করুন

kubectl get nodes

# All pods check করুন

kubectl get pods --all-namespaces

# আপনার deleted resources check করুন

kubectl get all --all-namespaces | grep "your-resource-name"

# Test deployment create করুন

kubectl create deployment test-restore --image=nginx

# Verify test deployment

kubectl get deployments
kubectl get pods

# Test deployment delete করুন

kubectl delete deployment test-restore
```

## ধাপ ১৪: Cleanup (যদি প্রয়োজন হয়)

``` bash
# পুরানো backup data delete করুন (যদি restore successful হয়)

sudo rm -rf /var/lib/etcd-old-backup

# Backup logs check করুন

tail -f /var/log/etcd-backup.log
```

## Important Bengali Tips:

1.  **বেকাপ নিয়মিত নিন** - কমপক্ষে দৈনিক
2.  **বেকাপ ভেরিফাই করুন** - শুধু বেকাপ নিলেই হবে না, verifyও করতে হবে
3.  **প্রোডাকশনে টেস্ট করুন** - restore process production-এ apply করার
    আগে test environment-এ test করুন
4.  **ডকুমেন্টেশন রাখুন** - কখন কোন বেকাপ নিলেন, কীভাবে restore করবেন সব
    লিখে রাখুন
5.  **মোনিটরিং সেটআপ করুন** - cronjob fail হলে notification পেতে alert
    set করুন




    ঠিক আছে। তুমি চাইছো **Automated Backup Script + CronJob সেটআপ ধাপ বাদ দিয়ে** README.md সংস্করণ।
আমি তোমার জন্য ফাইলটি সেইভাবে সাজিয়ে দিলাম, শুধু **manual backup এবং restore steps** থাকবে।

---

````markdown
# 🛠️ etcdctl Download, Backup, and Restore Guide (Manual Backup Only)

এই guide-এ আমরা শেখব কিভাবে **etcdctl ডাউনলোড, manual backup, restore এবং verify** করতে হয় Kubernetes cluster-এর জন্য।  

---

## ধাপ ১: etcdctl ডাউনলোড এবং ইন্সটলেশন

```bash
# প্রথমে etcd version check করুন
ETCD_VERSION=$(sudo docker ps | grep etcd | grep -oP 'etcd:\K[0-9.]+' | head -1)

# যদি version না পাওয়া যায়, তাহলে default version use করুন
ETCD_VERSION=${ETCD_VERSION:-"3.5.0"}

# etcdctl ডাউনলোড করুন
wget https://github.com/etcd-io/etcd/releases/download/v${ETCD_VERSION}/etcd-v${ETCD_VERSION}-linux-amd64.tar.gz

# ফাইলটি extract করুন
tar -xvf etcd-v${ETCD_VERSION}-linux-amd64.tar.gz

# etcdctl কে system path-এ move করুন
sudo mv etcd-v${ETCD_VERSION}-linux-amd64/etcdctl /usr/local/bin/
sudo chmod +x /usr/local/bin/etcdctl

# Version check করুন
etcdctl version
````

---

## ধাপ ২: Environment Variables সেটআপ

```bash
export ETCDCTL_API=3
export ETCD_CERT="/etc/kubernetes/pki/etcd/server.crt"
export ETCD_KEY="/etc/kubernetes/pki/etcd/server.key"
export ETCD_CACERT="/etc/kubernetes/pki/etcd/ca.crt"
export ETCD_ENDPOINTS="https://127.0.0.1:2379"
```

---

## ধাপ ৩: Connection Verify করুন

```bash
# etcd connection check করুন
etcdctl --endpoints=$ETCD_ENDPOINTS \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY \
  --cacert=$ETCD_CACERT \
  endpoint health

# etcd members list দেখুন
etcdctl --endpoints=$ETCD_ENDPOINTS \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY \
  --cacert=$ETCD_CACERT \
  member list
```

---

## ধাপ ৪: Manual Backup তৈরি করুন

```bash
# Backup directory create করুন
sudo mkdir -p /opt/etcd-backups
cd /opt/etcd-backups

# Snapshot backup নিন
etcdctl --endpoints=$ETCD_ENDPOINTS \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY \
  --cacert=$ETCD_CACERT \
  snapshot save etcd-snapshot-$(date +%Y-%m-%d-%H-%M-%S).db

# Backup verify করুন
etcdctl --write-out=table snapshot status etcd-snapshot-*.db
```

---

## ধাপ ৫: Restore করার প্রস্তুতি

```bash
# Cluster state check করুন
kubectl get pods --all-namespaces
kubectl get nodes

# Latest backup file identify করুন
BACKUP_FILE=$(ls -t /opt/etcd-backups/etcd-snapshot-*.db | head -1)
echo "Using backup file: $BACKUP_FILE"

# Backup verify করুন
etcdctl snapshot status $BACKUP_FILE
```

---

## ধাপ ৬: Cluster Services বন্ধ করুন

```bash
sudo systemctl stop kube-apiserver
sleep 30
sudo systemctl stop etcd
sudo systemctl status kube-apiserver
sudo systemctl status etcd
```

---

## ধাপ ৭: Current Data Backup নিন

```bash
sudo tar -czf /opt/etcd-backups/etcd-data-before-restore-$(date +%Y-%m-%d).tar.gz -C /var/lib/etcd .
sudo mv /var/lib/etcd /var/lib/etcd-old-backup
```

---

## ধাপ ৮: Restore করুন

```bash
sudo etcdctl snapshot restore $BACKUP_FILE --data-dir /var/lib/etcd-new
sudo mv /var/lib/etcd-new /var/lib/etcd
sudo chown -R etcd:etcd /var/lib/etcd
# অথবা kubeadm cluster-এর জন্য:
sudo chown -R 1001:1001 /var/lib/etcd
```

---

## ধাপ ৯: Services Start করুন

```bash
sudo systemctl start etcd
sleep 10
etcdctl --endpoints=$ETCD_ENDPOINTS \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY \
  --cacert=$ETCD_CACERT \
  endpoint health

sudo systemctl start kube-apiserver
sleep 30
```

---

## ধাপ ১০: Verify Restoration

```bash
kubectl cluster-info
kubectl get nodes
kubectl get pods --all-namespaces

# Test deployment
kubectl create deployment test-restore --image=nginx
kubectl get deployments
kubectl get pods
kubectl delete deployment test-restore
```

---

## ধাপ ১১: Cleanup

```bash
sudo rm -rf /var/lib/etcd-old-backup
```

---

## ✅ Important Tips

1. **নিয়মিত ব্যাকআপ নিন** (manual বা automated)
2. **ব্যাকআপ verify করুন**
3. **প্রোডাকশন আগে টেস্ট করুন**
4. **ডকুমেন্টেশন রাখুন**
5. **Restore process safe environment-এ practice করুন**

---

🎉 এই guide অনুসরণ করে তুমি সহজেই **manual etcd backup এবং restore** করতে পারবে।

```

---

যদি চাও, আমি এই Markdown ফাইল **`manual-etcd-backup-restore.md` নামে download link** হিসেবে তৈরি করে দিতে পারি যাতে তুমি সরাসরি সেভ করতে পারো।  

চাও কি আমি সেটা বানাই?
```

