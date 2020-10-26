# Cách cài đặt MySQL Slave Replication
**Bước 1**: Cấu hình Master Database
```
vi /etc/mysql/my.cnf

[mysqld]
...
server-id=1
log-bin=master
binlog-format=row
binlog-do-db=replica_db
```
Trong đó :
server_id là tùy chọn được sử dụng trong replication cho phép master server và slave server có thể nhận dạng lẫn nhau. Server_id Với mỗi server là khác nhau, nhận giá trị từ 1 đến 4294967295(mariadb >=10.2.2) và 0 đến 4294967295(mariadb =<10.2.1)
log-bin hay log-basename là tên cơ sở nhật ký nhị phân để tạo tên tệp nhật ký nhị phân. binlog-format là định dạng dữ liệu được lưu trong file bin log.
binlog-do-db là tùy chọn để nhận biết cơ sở dữ liệu nào sẽ được replication. Nếu muốn replication nhiều CSDL, bạn phải viết lại tùy chọn binlog-do-db nhiều lần. Hiện tại không có option cho phép chọn toàn bộ CSDL để replica mà bạn phải ghi tất cả CSDL muốn replica ra theo option này.

Sau khi chỉnh sửa xong chúng ta cần lưu lại file cấu hình và khởi động lại mysql
```
sudo service mysql restart
```
Bước tiếp theo chúng ta cần phải mở MySQL mà cấp quyền cho các slave, bạn có thể đặt tên, mật khẩu cho các slave tùy ý:
```
GRANT REPLICATION SLAVE ON *.* TO 'slave_user'@'%' IDENTIFIED BY 'password';
Sau đó kiểm tra bằng cách:
FLUSH PRIVILEGES;
```
Lựa chọn cơ sở dữ liệu cần replicate.
```
USE newdatabase;
```
Sau đó, quan trọng nhất là bạn cần phải khóa cơ sở dữ liệu mà sau này các slave để ở chế độ chỉ đọc
```
FLUSH TABLES WITH READ LOCK;
```
Để kiểm tra cơ sở dữ liệu chúng ta gõ câu lệnh sau:
```
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| master.000001    |      107 | newdatabase  |                  |
+------------------+----------+--------------+------------------+
1 row in set (0.00 sec)
```
Position ở đây có nghĩa là vị trí mà bắt đầu các slave sao lưu dữ liệu. Căn cứ vào cơ sở dữ liệu đã bị khóa, chúng ta export cơ sở dữ liệu ra 1 file sql để tiện dùng cho bước 2.
```
mysqldump -u root -p --opt newdatabase > newdatabase.sql
```
Bây giờ quay trở lại cửa sổ ban đầu và mở khóa cơ sở dữ liệu của bạn:
```
UNLOCK TABLES;
QUIT;
```
Về cơ bản cấu hình máy chủ đã tạm ổn

**Bước 2**: Cấu hình cơ sở dữ liệu slave

Đăng nhập vào server slave, mở mysql là tạo cơ sở dữ liệu với tên giống hệt cơ sở dữ liệu master:
```
CREATE DATABASE newdatabase;
EXIT;
```
Copy newdatabase từ master sang slave rồi Import.
```
mysql -u root -p newdatabase < /path/to/newdatabase.sql
```
Chúng ta cần cấu hình những con slave giống hệt như cách mà chúng ta cấu hình con master. Tuy nhiên cũng cần chỉnh sửa một số thông số cho phù hợp như server-id:
```
[mysqld]
...
server-id               = 2
binlog_do_db            = newdatabase
```
Khởi động lại mysql của con slave:
```
sudo service mysql restart
```
Bước tiếp theo chúng ta cần phải cấp quyền và cho phép nhân bản ở bên trong MySQL shell. Bật lại MySQL shell và thay thế các thông tin như sau:
```
CHANGE MASTER TO MASTER_HOST='12.34.56.789',MASTER_USER='slave_user', MASTER_PASSWORD='password', MASTER_LOG_FILE='master.000001', MASTER_LOG_POS= 107;
```
Nội dung command trên được hiểu như sau:
Chỉ định các máy chủ hiện tại như là slave của server master
Cung cấp thông tin đăng nhập chuẩn cho các máy chủ
Cuối cùng chỉ định cho các máy slave biết rằng cần phải sao lưu từ file log nào và đăng nhập từ vị trí mà đã định nghĩa trong position nào.

Sau đó chúng ta active server slave:
```
START SLAVE;

// Kiểm tra bằng cách:
SHOW SLAVE STATUS\G;

// Nếu có vấn đề trong kết nối bạn có thể thử start slave bằng cách:
SET GLOBAL sql_slave_skip_counter = N;
SLAVE START;
```
Thử N từ 1.



source: https://viblo.asia/p/gioi-thieu-ve-mysql-replication-master-slave-bxjvZYwNkJZ
