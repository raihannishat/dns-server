# Ubuntu 24.04 DNS Master-Slave সার্ভার কনফিগারেশন (BIND9)

এই ডকুমেন্টে একটি Master ও দুইটি Slave DNS সার্ভার কনফিগারেশন step-by-step দেখানো হয়েছে Ubuntu 24.04.2 LTS ভার্সনে। সার্ভারগুলো VirtualBox-এ VM হিসেবে রান করে।

## সার্ভার ডিটেইলস

| VM Name | IP Address     | Hostname       | ভূমিকা     |
| ------- | -------------- | -------------- | ---------- |
| vm-1    | 192.168.56.115 | ns1.raihan.dns | Master DNS |
| vm-2    | 192.168.56.116 | ns2.raihan.dns | Slave DNS  |
| vm-3    | 192.168.56.117 | ns3.raihan.dns | Slave DNS  |

---

## Step 1: Netplan Configuration (Master + Slaves)

```yaml
# /etc/netplan/50-cloud-init.yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.56.115/24
```

- Slave VM-গুলোর জন্য IP অনুযায়ী `addresses` পরিবর্তন করুন।
- Command:

```bash
sudo netplan apply
```

---

## Step 2: BIND9 ইনস্টল করুন (সব VM-এ)

```bash
sudo apt update
sudo apt install bind9 bind9utils bind9-doc dnsutils -y
```

---

## Step 3: Master DNS Configuration (ns1)

### 3.1 `named.conf.options`:

```conf
options {
    directory "/var/cache/bind";

    allow-transfer { 192.168.56.116; 192.168.56.117; };
    allow-query { any; };
    recursion no;

    dnssec-validation auto;
    listen-on { any; };
    allow-recursion { none; };
};
```

### 3.2 জোন ফাইল ডিক্লারেশন (named.conf.local):

```conf
zone "raihan.dns" IN {
    type master;
    file "/etc/bind/zones/db.raihan.dns";
    allow-transfer { 192.168.56.116; 192.168.56.117; };
};

zone "56.168.192.in-addr.arpa" IN {
    type master;
    file "/etc/bind/zones/db.192";
    allow-transfer { 192.168.56.116; 192.168.56.117; };
};
```

### 3.3 জোন ফাইল তৈরি করুন:

```bash
sudo mkdir -p /etc/bind/zones
sudo cp /etc/bind/db.local /etc/bind/zones/db.raihan.dns
sudo cp /etc/bind/db.127 /etc/bind/zones/db.192
```

### 3.4 জোন ফাইল এডিট (Forward & Reverse)

**db.raihan.dns** (A Records):

```dns
@   IN  SOA ns1.raihan.dns. admin.raihan.dns. (
            1 ; Serial
            604800 ; Refresh
            86400 ; Retry
            2419200 ; Expire
            604800 ) ; Negative Cache TTL
;
@   IN  NS      ns1.raihan.dns.
@   IN  NS      ns2.raihan.dns.
@   IN  NS      ns3.raihan.dns.

ns1 IN  A       192.168.56.115
ns2 IN  A       192.168.56.116
ns3 IN  A       192.168.56.117
www IN  A       192.168.56.115
```

**db.192** (PTR Records):

```dns
@   IN  SOA ns1.raihan.dns. admin.raihan.dns. (
            1 ; Serial
            604800 ; Refresh
            86400 ; Retry
            2419200 ; Expire
            604800 ) ; Negative Cache TTL
;
@   IN  NS      ns1.raihan.dns.
@   IN  NS      ns2.raihan.dns.
@   IN  NS      ns3.raihan.dns.

115 IN PTR ns1.raihan.dns.
116 IN PTR ns2.raihan.dns.
117 IN PTR ns3.raihan.dns.
```

### 3.5 BIND config check ও রিস্টার্ট

```bash
sudo named-checkconf
sudo named-checkzone raihan.dns /etc/bind/zones/db.raihan.dns
sudo named-checkzone 56.168.192.in-addr.arpa /etc/bind/zones/db.192
sudo systemctl restart bind9
```

---

## Step 4: Slave Configuration (ns2, ns3)

### 4.1 named.conf.local:

```conf
zone "raihan.dns" IN {
    type slave;
    masters { 192.168.56.115; };
    file "/var/cache/bind/db.raihan.dns";
};

zone "56.168.192.in-addr.arpa" IN {
    type slave;
    masters { 192.168.56.115; };
    file "/var/cache/bind/db.192";
};
```

### 4.2 `named.conf.options` (স্লেভ সার্ভারের জন্য):

```conf
options {
    directory "/var/cache/bind";

    allow-query { any; };
    recursion no;

    dnssec-validation auto;
    listen-on { any; };
    allow-recursion { none; };
};
```

### 4.3 BIND রিস্টার্ট করুন:

```bash
sudo systemctl restart bind9
```

---

## Step 5: resolv.conf স্থায়ীভাবে কনফিগার করা (যদি systemd-resolved ব্যবহৃত হয় না)

```bash
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
sudo rm /etc/resolv.conf
sudo nano /etc/resolv.conf
```

ফাইলের মধ্যে লিখুন:

```
nameserver 192.168.56.115
nameserver 192.168.56.116
nameserver 192.168.56.117
```

**অপশনাল:** ফাইল লক করে দিন যাতে আর পরিবর্তিত না হয়:

```bash
sudo chattr +i /etc/resolv.conf
```

---

## Step 6: DNS সার্ভার টেস্ট করুন

### 6.1 A রেকর্ড টেস্ট:

```bash
dig @192.168.56.115 www.raihan.dns
dig @192.168.56.116 www.raihan.dns
dig @192.168.56.117 www.raihan.dns
```

### 6.2 PTR রেকর্ড টেস্ট:

```bash
dig -x 192.168.56.115
```

### 6.3 AXFR (Zone Transfer) টেস্ট:

```bash
dig @192.168.56.116 raihan.dns AXFR
```

---

## সম্পন্ন ✅

আপনার DNS সার্ভার Master-Slave রেপ্লিকেশন সহ সম্পূর্ণভাবে কনফিগারড। এখন যেকোনো ক্লায়েন্ট VM এই DNS সার্ভার ব্যবহার করে নাম রেজলভ করতে পারবে।
