# DNS সার্ভার পূর্ণ কনফিগারেশন ও ডকুমেন্টেশন

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

## ✅ সার্ভার প্ল্যান

| VM   | Hostname       | IP             | ভূমিকা     |
| ---- | -------------- | -------------- | ---------- |
| vm-1 | ns1-raihan-dns | 192.168.56.115 | Master DNS |
| vm-2 | ns2-raihan-dns | 192.168.56.116 | Slave DNS  |
| vm-3 | ns3-raihan-dns | 192.168.56.117 | Slave DNS  |

---

## ✅ Netplan কনফিগারেশন

### Master: `/etc/netplan/50-cloud-init.yaml`

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

### Slave (ns2): `/etc/netplan/50-cloud-init.yaml`

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

### Slave (ns3): `/etc/netplan/50-cloud-init.yaml`

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

### Netplan অ্যাপ্লাই:

```bash
sudo netplan apply
```

---

## ✅ DNS সার্ভার সফটওয়্যার ইনস্টল

```bash
sudo apt update
sudo apt install bind9 bind9utils bind9-doc dnsutils
```

---

## ✅ Master DNS কনফিগারেশন ফাইলসমূহ

### `/etc/bind/named.conf.options`

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

**উদ্দেশ্য ব্যাখ্যা:**

- `allow-transfer`: Slave সার্ভারকে zone transfer অনুমতি
- `recursion no`: Master authoritative server, নিজের DNS resolve করবে না

### `/etc/bind/named.conf.local`

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

### জোন ফাইল: `/etc/bind/zones/db.raihan.dns`

```dns
$TTL    604800
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

### রিভার্স জোন: `/etc/bind/zones/db.192`

```dns
$TTL    604800
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

## ✅ Slave DNS: ns2 এবং ns3

### `/etc/bind/named.conf.local`

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

### `/etc/bind/named.conf.options`

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

---

## ✅ resolvectl ব্যবহার করে DNS কনফিগার (যদি resolv.conf ম্যানুয়ালি না করা হয়)

```bash
sudo resolvectl dns enp0s8 192.168.56.115 192.168.56.116 192.168.56.117
sudo resolvectl domain enp0s8 ~raihan.dns
resolvectl status
```

---

## ✅ কনফিগারেশন টেস্ট ও সার্ভার রিস্টার্ট

```bash
sudo named-checkconf
sudo named-checkzone raihan.dns /etc/bind/zones/db.raihan.dns
sudo named-checkzone 56.168.192.in-addr.arpa /etc/bind/zones/db.192
sudo systemctl restart bind9
```

---

## ✅ Master vs Slave পার্থক্য

| দিক    | Master                  | Slave                      |
| ------ | ----------------------- | -------------------------- |
| Type   | master                  | slave                      |
| File   | নিজে zone ফাইল ধরে রাখে | master থেকে ট্রান্সফার করে |
| Update | নিজে authoritative      | শুধু read-only copy রাখে   |

---

## ✅ টেস্টিং (dig দিয়ে)

```bash
dig @192.168.56.115 www.raihan.dns
dig @192.168.56.116 www.raihan.dns
dig -x 192.168.56.115
```

## ✅ zone transfer টেস্ট:

```bash
dig @192.168.56.116 raihan.dns AXFR
```

---

## ✅ শেষ কথা:

এই ডকুমেন্ট অনুসরণ করে আপনি একটি প্রোডাকশন-রেডি Master-Slave DNS ইনফ্রাস্ট্রাকচার তৈরি করেছেন, সম্পূর্ণ standard ও secure উপায়ে। আপনি চাইলে পরবর্তী ধাপে DHCP, Load Balancing, অথবা External DNS integration করতে পারেন।
