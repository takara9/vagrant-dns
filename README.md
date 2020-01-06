# vagrant-dns

CentOS8でDNSサーバーを構築するVagrant + Ansible playbookです。

## 起動方法

~~~


~~~


DNSの設定ファイルは、playbook/bind/vars/main.yml です。

~~~
.
├── README.md
├── Vagrantfile
├── ansible.cfg
└── playbook
    ├── bind
    │   ├── defaults
    │   │   └── main.yml
    │   ├── handlers
    │   │   └── main.yml
    │   ├── hosts
    │   ├── tasks
    │   │   └── main.yml
    │   ├── templates
    │   │   ├── db.forward.j2
    │   │   ├── db.reverse.j2
    │   │   ├── named.conf.j2
    │   │   ├── named.conf.local.j2
    │   │   └── resolv.j2
    │   └── vars
    │       └── main.yml
    └── install_bind.yml
~~~


## 設定ファイルの使用法

このファイルを編集することで、正引きと逆引きのzoneファイルを自動生成します。


~~~file:playbook/bind/vars/main.yml
---
hostname: server1

domain: takara9.org
reverse_domain: 1.20.172.in-addr.arpa.

dns_record:
 - name: w1
   type: A
   ipaddress: 172.20.1.100
 - name: w2
   type: A
   ipaddress: 172.20.1.101
~~~

* hostname: 値をセットすると、Vagrantfileで指定したホスト名を上書
* domain: 実在しないドメイン名を設定
* reverse_domain: 172.20.1.X であれば、1.20.172.in-addr.arpaとして記述
* name: ホスト名
* type: A はホストネームであることを表すので固定
* ipaddress: IPアドレスをベタ書きする

上記の設定ファイルから、以下のように/etc/named以下にファイルを生成する

~~~
[root@server1 vagrant]# tree /etc/named
/etc/named
|-- named.conf.local
`-- zones
    |-- db.1.20.172.in-addr.arpa.
    `-- db.takara9.org

1 directory, 3 files
~~~

## 動作確認方法

`systemctl status named.service` を実行することで、正常に稼働しているか確認できる。

~~~
[root@server1 vagrant]# systemctl status named.service
● named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2020-01-06 13:23:50 UTC; 21min ago
  Process: 6231 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/S>
  Process: 6228 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin>
 Main PID: 6233 (named)
    Tasks: 4 (limit: 11525)
   Memory: 57.2M
   CGroup: /system.slice/named.service
           └─6233 /usr/sbin/named -u named -c /etc/named.conf

Jan 06 13:23:50 server1.takara9.org named[6233]: zone 1.20.172.in-addr.arpa/IN: loaded serial 7
Jan 06 13:23:50 server1.takara9.org named[6233]: all zones loaded
Jan 06 13:23:50 server1.takara9.org named[6233]: running
Jan 06 13:23:50 server1.takara9.org systemd[1]: Started Berkeley Internet Name Domain (DNS).
Jan 06 13:33:15 server1.takara9.org named[6233]: resolver priming query complete
~~~

## ドメインの正引き

独自のドメインと、インターネット上のドメインの両方をひく事ができる

~~~
[root@server1 vagrant]# nslookup
> w1.takara9.org
Server:		127.0.0.1
Address:	127.0.0.1#53

Name:	w1.takara9.org
Address: 172.20.1.100
> w2.takara9.org
Server:		127.0.0.1
Address:	127.0.0.1#53

Name:	w2.takara9.org
Address: 172.20.1.101
> www.google.co.jp
Server:		127.0.0.1
Address:	127.0.0.1#53

Non-authoritative answer:
Name:	www.google.co.jp
Address: 172.217.161.35
Name:	www.google.co.jp
Address: 2404:6800:4004:808::2003
~~~




