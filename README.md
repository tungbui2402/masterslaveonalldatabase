# master slave on all database
## 1. Percona Database
### Trên máy master:
- Đầu tiên chúng ta chạy lệnh `sudo nano /etc/mysql/percona-server.conf.d/mysqld.cnf` để thiết lập cấu hình mysql
- Thêm dòng dưới đây vào cuối file:
```
# Replication
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
```
- Sau khi thiết lập xong thì chúng ta lưu lại rồi sử dụng lệnh `sudo systemctl restart mysql` để khởi động lại mysql
- Tiếp theo vào mysql bằng mysql -u root -p
- Tạo và cấp quyền cho tài khoản vừa tạo:
```
CREATE USER 'repl'@'%' IDENTIFIED BY 'repl_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
GRANT ALL PRIVILEGES ON *.* TO 'repl'@'%';
```
- Sau khi cấp quyền xong thì chạy lệnh show master status\G để lấy thông tin file và pos:
```
************************** 1. row ***************************
 File: master_log_file           
 Position: master_log_position       
 Binlog_Do_DB:
 Binlog_Ignore_DB:
 Executed_Gtid_Set:
 1 row in set (0.01 sec)
 ```
### Trên máy slave:
- Đầu tiên chúng ta chạy lệnh `sudo nano /etc/mysql/percona-server.conf.d/mysqld.cnf` để thiết lập cấu hình mysql
- Thêm dòng dưới đây vào cuối file:
```
# Replication
server-id = 2
```
- Sau khi thiết lập xong thì chúng ta lưu lại rồi sử dụng lệnh `sudo systemctl restart mysql` để khởi động lại mysql
- Tiếp theo vào `mysql bằng mysql -u root -p`
- Dùng lệnh `stop slave;` để dừng slave
- Sau đó cấu hình các câu lệnh:
```
CHANGE MASTER TO MASTER_HOST='%', MASTER_USER='repl', MASTER_PASSWORD='repl_password', MASTER_LOG_FILE='master_log_file', MASTER_LOG_POS=master_log_position;
```
- Sau khi cấu hình lệnh xong ta dùng lệnh `START SLAVE;` và kiểm tra trạng thái của slave `SHOW SLAVE STATUS\G` 
- Kiểm tra các trường "Slave_IO_Running" và "Slave_SQL_Running" để đảm bảo replication đang chạy một cách bình thường.
- Nếu các trường "Slave_IO_Running" và "Slave_SQL_Running" đều hiển thị giá trị "Yes", thì replication giữa master và slave đã được thiết lập thành công.
## 2. Postgre Database
Chuẩn bị 2 máy với Postgre cùng phiên bản, ở đây mình dùng phiên bản 15.3.
### Trên máy Master
B1: Sửa cấu hình trong file postgresql.conf bằng lệnh: `sudo nano /etc/postgresql/15/main/postgresql.conf`.

B2: Tìm trong thư mục dòng `listen_addresses` và sửa từ `localhost` về thành `*` rồi lưu và thoát.

B3: Đăng nhập vào postgre bằng lệnh `sudo -u postgres psql` và thêm dùng dùng để replication: `CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'admin@123';`.

B4: Sau khi thêm người dùng thì ta chỉnh sửa file pg_hba.conf bằng lệnh `sudo nano /etc/postgresql/15/main/pg_hba.conf`.

B5: Thêm vào cuối dòng sau: `host    replication     replicator      ipslave/24      md5`.

B6: Lưu và khởi động lại postgres `sudo systemctl restart postgresql`.

B7 (Sau bước 4 ở máy slave): Đăng nhập vào postgres và có thể thấy vị trí sao chép có tên là slotslave1 khi mở chế độ xem pg_replication_slots như sau:
`SELECT * FROM pg_replication_slots;`
Nếu trên máy master hiện slaveslot1 thì là thành công.

B8: Kiểm tra xem đã kết nối chưa bằng lệnh: `SELECT * FROM pg_stat_replication;`
Nếu hiện là streaming là thành công.

### Trên máy Slave
B1: Dừng postgres bằng lệnh `sudo systemctl stop postgresql`.

B2: Tạo backup:
```
su - postgres
cp -R /var/lib/postgresql/15/main/ /var/lib/postgresql/15/main_old/
```
B3: Sau đó ta xóa folder main cũ đi: `rm -rf /var/lib/postgresql/15/main/`.

B4: Bây giờ, sử dụng basebackup để có được một bản sao lưu cơ bản với quyền sở hữu chính xác bằng cách sử dụng postgres (hoặc bất kỳ người dùng nào có quyền chính xác):
`pg_basebackup -h ip master -D /var/lib/postgresql/15/main/ -U replicator -P -v -R -X stream -C -S slaveslot1`.
Sau đó nhập mật khẩu vào để quá trình bắt đầu.
Chú ý rằng file standby.signal được tạo ra và cài đặt kết nối được thêm vào postgresql.auto.conf:`ls -ltrh /var/lib/postgresql/15/main/`

B5: Chạy lại postgres bằng lệnh `systemctl start postgresql`.

B6: Dùng lệnh `SELECT * FROM pg_stat_wal_receiver;` để kiểm tra trạng thái ở chế độ chờ bằng lệnh bên dưới.

### Test
Tạo 1 database ở máy master bằng lệnh `create database dbname;`, nếu ở bên máy slave khi dùng lệnh `\l`; mà có database đó thì là thành công.

## 3. Mongo database
### Để hoàn thành hướng dẫn này, chúng ta sẽ cần:
- Hai máy chủ, mỗi máy chạy Ubuntu 20.04.
- MongoDB được cài đặt trên mỗi máy chủ Ubuntu của chúng ta.
### Các bước tiến hành:
B1: Cấu hình phân giải DNS:

Trước tiên, chúng ta sẽ cần thiết lập phân giải DNS trên mỗi máy chủ để chúng có thể giao tiếp với nhau bằng tên máy chủ.
Tiến hành sửa file /etc/hosts trên mỗi node bằng lệnh `sudo nano /etc/hosts` và thêm các dòng sau:
```
ipmaster mongodb0.replset.member
ipslave mongodb1.replset.member
```
Lưu và đóng file.
Chúng ta có thể tiến hành bước tiếp theo.

B2: Cấu hình node master:

Bước này thực hiện bằng cách chỉnh sửa file cấu hình của MongoDB /etc/mongod.conf của node master.
Tìm phần network interfaces trong file /etc/mongod.conf: `sudo nano /etc/mongod.conf`
Thêm dấu , vào dòng bindIp bằng tên máy chủ như sau:
```
# network interfaces
net:
  port: 27017
  bindIp: 127.0.0.1,mongodb0.replset.member
```
Tiếp theo, tìm dòng #replication: ở cuối file. Bỏ ghi chú dòng này bằng cách bỏ dấu thăng (#). Sau đó, thêm một replSetName, theo sau là tên mà MongoDB sẽ sử dụng để xác định tập hợp bản sao:
```
replication:
  replSetName: "" # Chúng ta có thể cung cấp bất kỳ tên nào chúng ta muốn ở đây. Lưu ý: Có hai khoảng trắng trước replSetName và tên được đặt trong dấu ngoặc kép (").
```
B2: Cấu hình node slave:

Bước này thực hiện bằng cách chỉnh sửa file cấu hình của MongoDB /etc/mongod.conf của node slave.
Tìm phần network interfaces trong file /etc/mongod.conf: `sudo nano /etc/mongod.conf`
Thêm dấu , vào dòng bindIp bằng tên máy chủ như sau:
```
# network interfaces
net:
  port: 27017
  bindIp: 127.0.0.1,mongodb1.replset.member
```
Tiếp theo, tìm dòng #replication: ở cuối file. Bỏ ghi chú dòng này bằng cách bỏ dấu thăng (#). Sau đó, thêm một replSetName, theo sau là tên mà MongoDB sẽ sử dụng để xác định tập hợp bản sao:
```
replication:
  replSetName: "" # Để giống bên master.
```
B4: Khởi động lại dịch vụ MongoDB trên tất cả máy
```
sudo systemctl restart mongod
```
B5: Thiết lập Replica Set:

Tiếp theo, chúng ta sẽ cấu hình replicate trên node master và các node slave.
Trên node master, kết nối với Mongo bằng người dùng quản trị:
```
mongo
```
Tiếp theo, khởi tạo tập hợp bản sao bằng lệnh sau:
```
rs.initiate()
```
Kết quả trả về như sau:
```
{
    "info2" : "no configuration specified. Using a default configuration for the set",
    "me" : "mongodb0.replset.member:27017",
    "ok" : 1
}
```
Tiếp theo, thêm node Slave 1 làm thành viên bằng lệnh sau:
```
replica0:SECONDARY> rs.add("mongodb1.replset.member")
```
Chúng ta cũng có thể kiểm tra trạng thái của tất cả các node bằng lệnh sau:
```
rs.status()
```
Kết quả ra như sau:
```
"set" : "rs0",
        "date" : ISODate("2023-06-08T09:37:56.239Z"),
        "myState" : 1,
        "term" : NumberLong(1),
        "syncSourceHost" : "",
        "syncSourceId" : -1,
        "heartbeatIntervalMillis" : NumberLong(2000),
        "majorityVoteCount" : 2,
        "writeMajorityCount" : 2,
        "votingMembersCount" : 2,
        "writableVotingMembersCount" : 2,
        "optimes" : {
                "lastCommittedOpTime" : {
                        "ts" : Timestamp(1686217074, 1),
                        "t" : NumberLong(1)
                },
                "lastCommittedWallTime" : ISODate("2023-06-08T09:37:54.600Z"),
                "readConcernMajorityOpTime" : {
                        "ts" : Timestamp(1686217074, 1),
                        "t" : NumberLong(1)
                },
                "appliedOpTime" : {
                        "ts" : Timestamp(1686217074, 1),
                        "t" : NumberLong(1)
                },
                "durableOpTime" : {
                        "ts" : Timestamp(1686217074, 1),
                        "t" : NumberLong(1)
                },
                "lastAppliedWallTime" : ISODate("2023-06-08T09:37:54.600Z"),
                "lastDurableWallTime" : ISODate("2023-06-08T09:37:54.600Z")
        },
        "lastStableRecoveryTimestamp" : Timestamp(1686217034, 1),
        "electionCandidateMetrics" : {
                "lastElectionReason" : "electionTimeout",
                "lastElectionDate" : ISODate("2023-06-08T09:37:14.565Z"),
                "electionTerm" : NumberLong(1),
                "lastCommittedOpTimeAtElection" : {
                        "ts" : Timestamp(1686217034, 1),
                        "t" : NumberLong(-1)
                },
                "lastSeenOpTimeAtElection" : {
                        "ts" : Timestamp(1686217034, 1),
                        "t" : NumberLong(-1)
                },
                "numVotesNeeded" : 1,
                "priorityAtElection" : 1,
                "electionTimeoutMillis" : NumberLong(10000),
                "newTermStartDate" : ISODate("2023-06-08T09:37:14.581Z"),
                "wMajorityWriteAvailabilityDate" : ISODate("2023-06-08T09:37:14.602Z")
        },
        "members" : [
                {
                        "_id" : 0,
                        "name" : "mongodb0.replset.member:27017",
                        "health" : 1,
                        "state" : 1,
                        "stateStr" : "PRIMARY",
                        "uptime" : 87,
                        "optime" : {
                                "ts" : Timestamp(1686217074, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDate" : ISODate("2023-06-08T09:37:54Z"),
                        "lastAppliedWallTime" : ISODate("2023-06-08T09:37:54.600Z"),
                        "lastDurableWallTime" : ISODate("2023-06-08T09:37:54.600Z"),
                        "syncSourceHost" : "",
                        "syncSourceId" : -1,
                        "infoMessage" : "",
                        "electionTime" : Timestamp(1686217034, 2),
                        "electionDate" : ISODate("2023-06-08T09:37:14Z"),
                        "configVersion" : 3,
                        "configTerm" : 1,
                        "self" : true,
                        "lastHeartbeatMessage" : ""
                },
                {
                        "_id" : 1,
                        "name" : "mongodb1.replset.member:27017",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 32,
                        "optime" : {
                                "ts" : Timestamp(1686217074, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDurable" : {
                                "ts" : Timestamp(1686217074, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDate" : ISODate("2023-06-08T09:37:54Z"),
                        "optimeDurableDate" : ISODate("2023-06-08T09:37:54Z"),
                        "lastAppliedWallTime" : ISODate("2023-06-08T09:37:54.600Z"),
                        "lastDurableWallTime" : ISODate("2023-06-08T09:37:54.600Z"),
                        "lastHeartbeat" : ISODate("2023-06-08T09:37:55.340Z"),
                        "lastHeartbeatRecv" : ISODate("2023-06-08T09:37:54.347Z"),
                        "pingMs" : NumberLong(0),
                        "lastHeartbeatMessage" : "",
                        "syncSourceHost" : "mongodb0.replset.member:27017",
                        "syncSourceId" : 0,
                        "infoMessage" : "",
                        "configVersion" : 3,
                        "configTerm" : 1
                }
        ],
        "ok" : 1,
        "$clusterTime" : {
                "clusterTime" : Timestamp(1686217074, 1),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        },
        "operationTime" : Timestamp(1686217074, 1)
}
```
### Thử nghiệm Replica Set
Tạo cơ sở dữ liệu và thêm một số giá trị:
```
use test
switched to db test
for (var i = 0; i <= 10; i++) db.exampleTest.insert( { x : i } )
```
Đầu ra:
```
WriteResult({ "nInserted" : 1 })
```
Kiểm tra cơ sở dữ liệu của bạn bằng lệnh sau: ` show dbs `
Đầu ra: 
```
test  0.000GB
admin      0.000GB
config     0.000GB
local      0.000GB
```
Trên máy slave chạy lệnh: `db.getMongo().setSecondaryOk()`
Tiếp theo, thay đổi cơ sở dữ liệu thành test: ` use test `
Kết quả:
```
switched to db test
```
Tiếp theo, chạy lệnh sau để hiển thị tất cả các tài liệu:
Nếu bản sao đang hoạt động, chúng ta sẽ thấy danh sách các tài liệu mẫu mà chúng ta đã tạo trên node master.
