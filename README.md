# master slave on all database
## 1. Percona Database
### Trên máy master:
- Đầu tiên chúng ta chạy lệnh `sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf` để thiết lập cấu hình mysql
- Sau đó ta tìm đến dòng bind-address=127.0.0.1 rồi thay đổi thông số về 0.0.0.0
- Tiếp đến ta tìm tiếp 2 dòng là server-id=1 và log_bin=/var/log/mysql/mysql-bin.log và bỏ # ở trước đi
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
- Đầu tiên chúng ta chạy lệnh `sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf` để thiết lập cấu hình mysql
- Sau đó ta tìm đến dòng bind-address=127.0.0.1 rồi thay đổi thông số về 0.0.0.0
- Tiếp đến ta tìm tiếp 2 dòng là server-id=2 và log_bin=/var/log/mysql/mysql-bin.log và bỏ # ở trước đi
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
cp -R /var/lib/postgresql/14/main/ /var/lib/postgresql/15/main_old/
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
