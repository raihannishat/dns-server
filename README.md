
# DNS সার্ভার পূর্ণ কনফিগারেশন ও ডকুমেন্টেশন

---

## ✅ ভূমিকা: DNS কি, কেন দরকার?

### DNS (Domain Name System) কী?

DNS হলো একটি সিস্টেম যা ডোমেইন নাম (যেমন: `www.google.com`) কে IP ঠিকানায় (যেমন: `142.250.190.68`) রূপান্তর করে।

### কেন দরকার?

- ব্যবহারকারীরা সহজে নাম দিয়ে ওয়েবসাইট অ্যাক্সেস করতে পারে।
- সিস্টেম ও নেটওয়ার্কে নামভিত্তিক কমিউনিকেশন সম্ভব হয়।
- DNS ছাড়া আপনাকে সব জায়গায় IP মনে রাখতে হতো।

### কোথায় দরকার?

- স্থানীয় LAN নেটওয়ার্কে
- ইন্টারনাল সার্ভিস রেজল্যুশনের জন্য
- ইন্টারনেটে ওয়েবসাইট হোস্টিংয়ের সময়

### কাদের জন্য দরকার?

- সিস্টেম অ্যাডমিন
- নেটওয়ার্ক ইঞ্জিনিয়ার
- ওয়েব অ্যাপ্লিকেশন ও সার্ভার ডেভেলপার
---

### আমার সার্ভার গুলো vm এ রান করবে এবং আমার প্রাইমারি OS হলো windows 11, যেখানে ১ টা মাস্টার সার্ভার এবং ২ টা স্লেভ সার্ভার থাকবে। আমি vm এর ভিতরে Ubuntu 24.04.2 LTS ব্যবহার করেছি। 
---

## ✅ সার্ভার প্ল্যান

| VM   | Hostname       | IP             | ভূমিকা     |
| ---- | -------------- | -------------- | ---------- |
| vm-1 | ns1.raihan.dns | 192.168.56.115 | Master DNS |
| vm-2 | ns2.raihan.dns | 192.168.56.116 | Slave DNS  |
| vm-3 | ns3.raihan.dns | 192.168.56.117 | Slave DNS  |

---

## ✅ Hostname এবং /etc/hosts সেটআপ

প্রতিটি VM-এ নিচের কমান্ড রান করুন:

```bash
sudo hostnamectl set-hostname ns1.raihan.dns   # Master এর জন্য
sudo hostnamectl set-hostname ns2.raihan.dns   # Slave 1 এর জন্য
sudo hostnamectl set-hostname ns3.raihan.dns   # Slave 2 এর জন্য
```

`/etc/hosts` ফাইল এডিট করুন (সব সার্ভারে):

```
192.168.56.115   ns1.raihan.dns ns1
192.168.56.116   ns2.raihan.dns ns2
192.168.56.117   ns3.raihan.dns ns3
```

---

## ✅ Netplan নেটওয়ার্ক কনফিগারেশন

### Master (ns1) - `/etc/netplan/50-cloud-init.yaml`

```yaml
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

### Slave 1 (ns2) - `/etc/netplan/50-cloud-init.yaml`

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.56.116/24
```

### Slave 2 (ns3) - `/etc/netplan/50-cloud-init.yaml`

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.56.117/24
```

নেটওয়ার্ক প্রয়োগ করুন:

```bash
sudo netplan try
sudo netplan apply
```

---

## ✅ DNS সার্ভার সফটওয়্যার ইনস্টলেশন

সব সার্ভারে রান করুন:

```bash
sudo apt update
sudo apt install bind9 bind9utils bind9-doc dnsutils -y
```

---

## ✅ Master DNS কনফিগারেশন (ns1)

### ১. `/etc/bind/named.conf.options`

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

### ২. `/etc/bind/named.conf.local`

```conf
zone "raihan.dns" {
    type master;
    file "/etc/bind/zones/db.raihan.dns";
    allow-transfer { 192.168.56.116; 192.168.56.117; };
};

zone "56.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192";
    allow-transfer { 192.168.56.116; 192.168.56.117; };
};
```

### ৩. Zone ফাইল তৈরি করুন

`zones` ফোল্ডার তৈরি করুন:

```bash
sudo mkdir -p /etc/bind/zones
```

#### Forward Zone: `/etc/bind/zones/db.raihan.dns`

```dns
$TTL 604800
@       IN      SOA     ns1.raihan.dns. root.raihan.dns. (
                            2         ; Serial
                       604800         ; Refresh
                        86400         ; Retry
                      2419200         ; Expire
                       604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.raihan.dns.
@       IN      NS      ns2.raihan.dns.
@       IN      NS      ns3.raihan.dns.

ns1     IN      A       192.168.56.115
ns2     IN      A       192.168.56.116
ns3     IN      A       192.168.56.117

www     IN      A       192.168.56.115
```

#### Reverse Zone: `/etc/bind/zones/db.192`

```dns
$TTL 604800
@       IN      SOA     ns1.raihan.dns. root.raihan.dns. (
                            1         ; Serial
                       604800         ; Refresh
                        86400         ; Retry
                      2419200         ; Expire
                       604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.raihan.dns.
@       IN      NS      ns2.raihan.dns.
@       IN      NS      ns3.raihan.dns.

115     IN      PTR     ns1.raihan.dns.
116     IN      PTR     ns2.raihan.dns.
117     IN      PTR     ns3.raihan.dns.
```

---

## ✅ Slave DNS কনফিগারেশন (ns2 এবং ns3)

### ১. `/etc/bind/named.conf.options`

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

### ২. `/etc/bind/named.conf.local`

```conf
zone "raihan.dns" {
    type slave;
    masters { 192.168.56.115; };
    file "/var/cache/bind/db.raihan.dns";
};

zone "56.168.192.in-addr.arpa" {
    type slave;
    masters { 192.168.56.115; };
    file "/var/cache/bind/db.192";
};
```

---

## ✅ সার্ভিস রিস্টার্ট এবং কনফিগারেশন যাচাই

**Master ও Slave উভয়ে:**

```bash
sudo named-checkconf
sudo named-checkzone raihan.dns /etc/bind/zones/db.raihan.dns      # Master এ রান করবেন
sudo named-checkzone 56.168.192.in-addr.arpa /etc/bind/zones/db.192 # Master এ রান করবেন
sudo systemctl restart bind9
sudo systemctl status bind9
```

DNS সার্ভার সঠিকভাবে কাজ করার জন্য, ফায়ারওয়ালে TCP এবং UDP পোর্ট ৫৩ অবশ্যই খোলা থাকতে হবে (সব সার্ভারের জন্য):

```bash
sudo firewall-cmd --add-port=53/tcp --permanent
sudo firewall-cmd --add-port=53/udp --permanent
sudo firewall-cmd --reload

sudo iptables -L -n | grep 53
sudo iptables -A INPUT -p tcp --dport 53 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 53 -j ACCEPT
sudo iptables-save

sudo firewall-cmd --list-all
```

---

## ✅ DNS টেস্টিং

### Forward Lookup

```bash
dig @192.168.56.115 www.raihan.dns
dig @192.168.56.116 www.raihan.dns
dig @192.168.56.117 www.raihan.dns
```

### Reverse Lookup

```bash
dig -x 192.168.56.115
dig -x 192.168.56.116
dig -x 192.168.56.117
```

### Zone Transfer (Slave Sync Test)

```bash
dig @192.168.56.116 raihan.dns AXFR
dig @192.168.56.117 raihan.dns AXFR
```

---

## ✅ Optional: `resolvectl` দিয়ে DNS কনফিগারেশন (যদি systemd-resolved ব্যবহার হয়)

```bash
sudo resolvectl dns enp0s8 192.168.56.115 192.168.56.116 192.168.56.117
sudo resolvectl domain enp0s8 ~raihan.dns
resolvectl status
```

---

## ✅ Master vs Slave DNS পার্থক্য

| বিষয়        | Master DNS                  | Slave DNS                  |
|-------------|-----------------------------|----------------------------|
| Zone Type   | Master (Authoritative)       | Slave (Read-only copy)      |
| Zone File   | নিজে zone ফাইল ধরে রাখে       | Master থেকে zone টেনে নেয়   |
| Update      | DNS রেকর্ড আপডেটের জায়গা     | শুধুমাত্র রিড-অনলি          |
| Zone Transfer| Slave সার্ভারকে অনুমতি দেয়   | Master থেকে zone টেনে নেয়   |

---

## ✅ টিপস ও বেস্ট প্র্যাকটিস

- Zone ফাইল পরিবর্তনের পর Serial নম্বর বাড়ানো আবশ্যক।
- `allow-transfer` এ Slave সার্ভারের সঠিক IP দিন।
- কনফিগারেশন চেক করতে `named-checkconf` এবং `named-checkzone` ব্যবহার করুন।
- ফায়ারওয়ালে TCP ও UDP 53 পোর্ট খোলা আছে কিনা নিশ্চিত করুন।
- Master ও Slave উভয়ের জন্য `recursion no` রাখা ভালো।

---

# শেষ কথা

এই ডকুমেন্ট অনুসরণ করে আপনি প্রোডাকশন-রেডি Master-Slave DNS ইনফ্রাস্ট্রাকচার তৈরি করতে পারবেন। প্রয়োজনে DHCP, লোড ব্যালেন্সার, অথবা এক্সটার্নাল DNS ইন্টিগ্রেশন যুক্ত করা যাবে।

---
