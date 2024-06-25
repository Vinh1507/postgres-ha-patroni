# High-availability (HA) PostgreSQL Cluster with Patroni

Patroni dùng để tạo giải pháp tính HA PostgreSQL sử dụng sự kết hợp của PostgreSQL streaming replication và consensus-based configuration store (kho cấu hình dựa trên sự đồng thuận) như etcd, ZooKeeper, or Consul

Cấu hình mặc định thì PostgreSQL không có tính năng dành cho high availability

![alt text](./images/image-16.png)

Trong kiến trúc này, sẽ luôn có 1 `leader`, và những node còn lại sẽ là `replicas`. Khi leader down, một trong những replicas sẽ được chọn trở thành leader mới với sự giúp đỡ từ `etcd`

## Components

### 1. PostgreSQL
PostgreSQL is the most widely used database management system in the world

### 2. Patroni

Cung cấp một mẫu để định cấu hình cụm PostgreSQL có tính sẵn sàng cao

Patroni cũng có thể xử lý các cấu hình để replication, sao lưu và khôi phục cơ sở dữ liệu.

Cài đặt của Patroni được quản lý thông qua việc sử dụng tệp YAML, tệp này có thể được lưu trữ ở bất kỳ vị trí nào. Thường lưu các cấu hình trong tệp /etc/patroni.yml.

### 3. ETCD
Etcd lưu trữ trạng thái của cụm PostgreSQL. Khi tìm thấy bất kỳ thay đổi nào về trạng thái của bất kỳ nút PostgreSQL nào, Patroni sẽ cập nhật thay đổi trạng thái trong kho lưu trữ key-value ETCD. ETCD sử dụng thông tin này để chọn primary và duy trì hoạt động của cụm.

### 4. HAProxy

HAProxy là phần mềm mã nguồn mở miễn phí, cung cấp bộ cân bằng tải và máy chủ proxy có tính khả dụng cao cho các ứng dụng dựa trên TCP và HTTP nhằm phân tán yêu cầu trên nhiều máy chủ. 

HAProxy theo dõi các thay đổi trong node master và node slave và kết nối với nút master thích hợp khi client yêu cầu kết nối.


## How it works?
Một <b>Patroni</b> bot được install trong mỗi node Postgresql trong cụm. 

Quá trình chọn ra leader liên quan đến việc tạo cố gắng set expired key trong Etcd. Database primary được chọn là node set được Etcd key đầu tiên. 

Sau khi xác định được có node sở hưu khóa, Patroni bot sẽ cầu hình instance postgresql đó hoạt động như database `primary`.

Việc lựa chọn leader cũng được hiển thị cho các node còn lại, và Patroni của các bot còn lại sẽ cấu hình instances thành `replica`

<b>HAProxy</b> giám sát các thay đổi trong node master/slave và kết nối với nút master thích hợp khi máy client cầu kết nối. HAProxy xác định nút nào là nút master bằng cách gọi API REST của Patroni. API REST của Patroni được định cấu hình để chạy trên cổng 8008 trong mỗi nút cơ sở dữ liệu.

Role: Master

![alt text](./images/image-17.png)

Role: Replica

![alt text](./images/image-18.png)


Kiến trúc này được thiết kế để cung cấp một cụm cơ sở dữ liệu mạnh mẽ, tự quản lý, trong đó tính sẵn sàng cao là mối quan tâm hàng đầu. Bằng cách sử dụng đồng thời Patroni, etcd và HAProxy, quá trình thiết lập sẽ tự động chuyển đổi dự phòng, cân bằng tải hiệu quả và sao chép nhất quán, từ đó mang lại một môi trường cơ sở dữ liệu linh hoạt và hiệu suất cao.

Quá trình hồi phục failover không phải luôn được thực hiện ngay llaapj tức, còn tùy thuộc vào trạng thái của cụm và tùy vào lý do xảy ra failover. Có thể có 1 khoảng thời gian ngắn unavailablity khi một leader mới đang được chọn vầ cụm đang được config lại

# Setup Postgres HA Patroni
## Architecture
![alt text](./images/image-15.png)

Machine: node1                   IP: 192.168.144.133                 Role: Postgresql, Patroni

Machine: node2                   IP: 192.168.144.135                Role: Postgresql, Patroni

Machine: node3                   IP: 192.168.144.136                 Role: Postgresql, Patroni

Machine: etcdnode              IP: 192.168.144.129           Role: etcd

Machine: haproxynode       IP: 192.168.144.132    Role: HA Proxy

## Step 1 –  Setup node1, node2, node3:

### Node1
```
sudo apt update

sudo hostnamectl set-hostname nodeN # chú ý thay N = 1,2,3

sudo apt install net-tools

sudo apt install postgresql postgresql-server-dev-12

sudo systemctl stop postgresql

sudo ln -s /usr/lib/postgresql/12/bin/* /usr/sbin/

sudo apt -y install python python3-pip

sudo apt install python3-testresources   

sudo pip3 install --upgrade setuptools 
 
sudo pip3 install psycopg2

sudo pip3 install patroni

sudo pip3 install python-etcd
```

## Step 2 –  Setup etcdnode:

```
sudo apt update

sudo hostnamectl set-hostname etcdnode

sudo apt install net-tools

sudo apt -y install etcd 
```

## Step 3 – Setup haproxynode:
```
sudo apt update

sudo hostnamectl set-hostname haproxynode

sudo apt install net-tools

sudo apt -y install haproxy
```

## Step 4 – Configure etcd on the etcdnode: 
sudo vi /etc/default/etcd   

ETCD_LISTEN_PEER_URLS="http://192.168.144.129:2380"
ETCD_LISTEN_CLIENT_URLS="http://localhost:2379,http://192.168.144.129:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.144.129:2380"
ETCD_INITIAL_CLUSTER="default=http://192.168.144.129:2380,"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.144.129:2379"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"

![alt text](./images/image.png)

sudo systemctl restart etcd 

sudo systemctl status etcd

curl http://192.168.144.129:2380/members
![alt text](./images/image-1.png)


## Step 5 – Configure Patroni on the node1, on the node2 and on the node3:
```
sudo vi /etc/patroni.yml
```

### Node 1
```
scope: postgres
namespace: /db/
name: node1

restapi:
    listen: 192.168.144.133:8008
    connect_address: 192.168.144.133:8008

etcd:
    host: 192.168.144.129:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:

  initdb:
  - encoding: UTF8
  - data-checksums

  pg_hba:
  - host replication replicator 127.0.0.1/32 md5
  - host replication replicator 192.168.144.133/0 md5
  - host replication replicator 192.168.144.135/0 md5
  - host replication replicator 192.168.144.136/0 md5
  - host all all 0.0.0.0/0 md5

  users:
    admin:
      password: admin
      options:
        - createrole
        - createdb

postgresql:
  listen: 192.168.144.133:5432
  connect_address: 192.168.144.133:5432
  data_dir: /data/patroni
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: Vinh1507
    superuser:
      username: postgres
      password: Vinh1507
  parameters:
      unix_socket_directories: '.'

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

### Node 2

```
scope: postgres
namespace: /db/
name: node2

restapi:
    listen: 192.168.144.135:8008
    connect_address: 192.168.144.135:8008

etcd:
    host: 192.168.144.129:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:

  initdb:
  - encoding: UTF8
  - data-checksums

  pg_hba:
  - host replication replicator 127.0.0.1/32 md5
  - host replication replicator 192.168.144.133/0 md5
  - host replication replicator 192.168.144.135/0 md5
  - host replication replicator 192.168.144.136/0 md5
  - host all all 0.0.0.0/0 md5

  users:
    admin:
      password: admin
      options:
        - createrole
        - createdb

postgresql:
  listen: 192.168.144.135:5432
  connect_address: 192.168.144.135:5432
  data_dir: /data/patroni
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: Vinh1507
    superuser:
      username: postgres
      password: Vinh1507
  parameters:
      unix_socket_directories: '.'

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```


### Node 3

```
scope: postgres
namespace: /db/
name: node3

restapi:
    listen: 192.168.144.136:8008
    connect_address: 192.168.144.136:8008

etcd:
    host: 192.168.144.129:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:

  initdb:
  - encoding: UTF8
  - data-checksums

  pg_hba:
  - host replication replicator 127.0.0.1/32 md5
  - host replication replicator 192.168.144.133/0 md5
  - host replication replicator 192.168.144.135/0 md5
  - host replication replicator 192.168.144.136/0 md5
  - host all all 0.0.0.0/0 md5

  users:
    admin:
      password: admin
      options:
        - createrole
        - createdb

postgresql:
  listen: 192.168.144.136:5432
  connect_address: 192.168.144.136:5432
  data_dir: /data/patroni
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: Vinh1507
    superuser:
      username: postgres
      password: Vinh1507
  parameters:
      unix_socket_directories: '.'

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

```
sudo mkdir -p /data/patroni
```
```
sudo chown postgres:postgres /data/patroni
```
```
sudo chmod 700 /data/patroni 
```
```
sudo vi /etc/systemd/system/patroni.service
```
```
[Unit]
Description=High availability PostgreSQL Cluster
After=syslog.target network.target

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni.yml
KillMode=process
TimeoutSec=30
Restart=no

[Install]
WantedBy=multi-user.targ
```

## Step 6 – Start Patroni service on the node1, on the node2 and on the node3:
```
sudo systemctl start patroni

sudo systemctl status patroni
```

## Step 7 – Configuring HA Proxy on the node haproxynode: 

```
sudo vi /etc/haproxy/haproxy.cfg

Replace its context with this:

global

        maxconn 100
        log     127.0.0.1 local2

defaults
        log global
        mode tcp
        retries 2
        timeout client 30m
        timeout connect 4s
        timeout server 30m
        timeout check 5s

frontend stats
   bind *:8404
   option http-use-htx
   http-request use-service prometheus-exporter if { path /metrics }
   stats enable
   stats uri /stats
   stats refresh 10s


listen postgres
    bind *:5000
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 192.168.144.133:5432 maxconn 100 check port 8008
    server node2 192.168.144.135:5432 maxconn 100 check port 8008
    server node3 192.168.144.136:5432 maxconn 100 check port 8008
```
```
sudo systemctl restart haproxy

sudo systemctl status haproxy
```
![alt text](./images/image-4.png)

Truy cập http://192.168.144.132:8404/stats
![alt text](./images/image-3.png)


Chạy trên 1 node bất kỳ
```
patronictl -c /etc/patroni.yml list
```
![alt text](./images/image-5.png)


### Kết nối tới Database thông qua Ha proxy

Bị lỗi: Error connecting to server : received invalid response to SSL negotiation

[Stackoverflow](https://stackoverflow.com/questions/28313848/error-connecting-to-server-received-invalid-response-to-ssl-negotiation)

Chú ý phải sửa lại /etc/haproxy/haproxy.cfg để mở kết nối dạng TCP
```
listen postgres
    bind *:5000
    mode tcp
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 192.168.144.133:5432 maxconn 100 check port 8008
    server node2 192.168.144.135:5432 maxconn 100 check port 8008
    server node3 192.168.144.136:5432 maxconn 100 check port 8008
```

pgAdmin4

![alt text](./images/image-6.png)

Hoặc sử dụng psql:
```
psql -h 192.168.144.132 -p 5000 -U postgres
```

## Thử nghiệm replica:

### Kịch bản:
- Truy cập vào 3 database trên 3 node
- Tại node1 (primary) tạo mới 1 table `employees`
- Thêm vào bảng trên 2 bản ghi
- Kiểm tra 2 node còn lại với câu lệnh `\dt`
```
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    position VARCHAR(50),
    salary NUMERIC(15, 2)
);
INSERT INTO employees (name, position, salary) VALUES
('Bui Hoang Vinh', 'Software Engineer', 75000.00),
('Pham Van Toi', 'Database Administrator', 80000.00);
SELECT * FROM employees;
```

```
INSERT INTO employees (name, position, salary) VALUES
('Nguyen Duc Anh', 'Software Engineer', 95000.00);
SELECT * FROM employees;
```
### Kết quả

Khi hoàn thành tạo bảng

![alt text](./images/image-7.png)

Sau khi insert 2 bản ghi

Tất cả các node đã có dữ liệu

![alt text](./images/image-8.png)

Nếu thực hiện insert trên replica sẽ không được phép thực hiện (read-only transaction)

![alt text](./images/image-9.png)
### Nhận xét:
Đã thực hiện replica 


## Thử nghiệm failover

Hiện tại, node1 đang là primary, thực hiện stop patroni trên node1
```
sudo systemctl stop patroni
```

![alt text](./images/image-10.png)

Kết quả: Node3 được trở thành primary thay thế cho Node1

Thực hiện thêm 1 bản ghi khi node1 chưa trở lại

![alt text](./images/image-11.png)
![alt text](./images/image-12.png)

Khi node 1 đang down thì không thể access vào được

![alt text](./images/image-13.png)

Cho node1 quay trở lại
```
sudo systemctl start patroni
```


Node1 vẫn được replica đầy đủ như các node khác

![alt text](./images/image-14.png)


## References:

[High-availability (HA) PostgreSQL Cluster with Patroni - Medium](https://medium.com/@chriskevin_80184/high-availability-ha-postgresql-cluster-with-patroni-1af7a528c6be)

[Setup Patroni - jfrog](https://jfrog.com/community/devops/highly-available-postgresql-cluster-using-patroni-and-haproxy/)
