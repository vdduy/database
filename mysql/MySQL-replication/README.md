# Cách cài đặt MySQL Slave Replication
**Bước 1**: Cấu hình Master Database
```
sudo nano /etc/mysql/my.cnf
```
Bước đầu tiên phải tìm đến phần trông như sau để binding server master localhost chẳng hạn:
```
bind-address            = 127.0.0.1
```
Thay thế địa chỉ IP local thành địa chỉ của server. Ví dụ:
```
bind-address            = 12.34.56.789
```
Thay đổi tiếp theo đề cập đến các server-id, nằm trong phần [mysqlId]. Bạn có thể chọn bất kì số nào, ví dụ đơn giản nhất có thể đặt là 1.
```
server-id               = 1
```
Tiếp theo đặt đường dẫn file log cho mysql, tất cả các sự kiện của slave được lưu trữ trong đường dẫn này. Tìm đến dòng log_bin:
```
log_bin                 = /var/log/mysql/mysql-bin.log
```
Cuối cùng chúng ta cần phải chỉ định cơ sở dữ liệu sẽ được nhân bản trên các máy slave. Bạn có thể chỉ định thay vì một mà là nhiều các slave bằng cách lặp lại dòng này cho tất cả các cơ sở dữ liệu mà bạn cần:
```
binlog_do_db            = newdatabase
```
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
Một điều quan trọng nữa, bạn cần phải mở 1 tab mới và lựa chọn cơ sở dữ liệu của bạn.
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
| mysql-bin.000001 |      107 | newdatabase  |                  |
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
Import cơ sở dữ liệu mà đã export ở bước 1.
```
mysql -u root -p newdatabase < /path/to/newdatabase.sql
```
Chúng ta cần cấu hình những con slave giống hệt như cách mà chúng ta cấu hình con master. Tuy nhiên cũng cần chỉnh sửa một số thông số cho phù hợp như server-id:
```
server-id               = 2

relay-log               = /var/log/mysql/mysql-relay-bin.log

log_bin                 = /var/log/mysql/mysql-bin.log

binlog_do_db            = newdatabase
```
Khởi động lại mysql của con slave:
```
sudo service mysql restart
```
Bước tiếp theo chúng ta cần phải cấp quyền và cho phép nhân bản ở bên trong MySQL shell. Bật lại MySQL shell và thay thế các thông tin như sau:
```
CHANGE MASTER TO MASTER_HOST='12.34.56.789',MASTER_USER='slave_user', MASTER_PASSWORD='password', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=  107;
```
Nội dung command trên được hiểu như sau:
Chỉ định các máy chủ hiện tại như là slave của server master
Cung cấp thông tin đăng nhập chuẩn cho các máy chủ
Cuối cùng chỉ định cho các máy slave biết rằng cần phải sao lưu từ file log nào và đăng nhập từ vị trí mà đã định nghĩa trong position nào.

Sau đó chúng ta active server slave:
```
START SLAVE;

// Kiểm tra bằng cách:
SHOW SLAVE STATUS\G

// Nếu có vấn đề trong kết nối bạn có thể thử start slave bằng cách:
SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1; SLAVE START;
```
Trên đây là tất cả những gì tôi tìm hiểu được về MySQL Replication. Nó còn nhiều các góc cạnh khác nhau, đây chỉ là một trong những khía cạnh tôi tìm hiểu một cách khái quát.



source: https://viblo.asia/p/gioi-thieu-ve-mysql-replication-master-slave-bxjvZYwNkJZ
