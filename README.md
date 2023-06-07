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
Chuẩn bị 2 máy với Postgre cùng phiên bản
### Trên máy Master
B1: Sửa cấu hình trong file postgresql.conf bằng lệnh: `sudo nano /etc/postgresql/15/main/postgresql.conf`
B2: Tìm trong thư mục dòng `listen_addresses` và sửa từ `localhost` về thành `*` rồi lưu và thoát.
B3: Đăng nhập vào postgre bằng lệnh `sudo -u postgres psql` và thêm dùng dùng để replication: `CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'admin@123';`
B4: Sau khi thêm người dùng thì ta chỉnh sửa file pg_hba.conf bằng lệnh sudo nano /etc/postgresql/15/main/
