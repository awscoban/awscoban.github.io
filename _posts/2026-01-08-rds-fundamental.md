---
layout: post
title: "19. Relational Database Service (RDS)"
title2: "Relational Database Service (RDS)"
date: 2026-01-08
permalink: /2026/01/08/rds-fundamental
categories: [RDS, Database]
tags: [RDS, Database]
img: /assets/19_rds/rds.png
summary: "Hẳn bạn đọc đã quen thuộc với việc sử dụng cơ sở dữ liệu SQL trong các ứng dụng truyền thống, như MySQL, PostgreSQL, SQL Server, Oracle, v.v. Đây là thành phần quan trọng bậc nhất trong hầu hết các ứng dụng, nên đương nhiên AWS cũng cung cấp dịch vụ cơ sở dữ liệu trên hạ tầng đám mây, gọi là Relational Database Service (RDS)."
---

Hẳn bạn đọc đã quen thuộc với việc sử dụng cơ sở dữ liệu SQL trong các ứng dụng truyền thống, như MySQL, PostgreSQL, SQL Server, Oracle, v.v. 
Đây là thành phần quan trọng bậc nhất trong hầu hết các ứng dụng, nên đương nhiên AWS cũng cung cấp dịch vụ cơ sở dữ liệu trên hạ tầng đám mây, gọi là **Relational Database Service** (**RDS**).



## Trong bài này:

- [1. SQL và NoSQL](#sql-nosql)
    - [1.1. SQL](#sql)
    - [1.2. NoSQL](#no-sql)
- [2. Relational Database Service (RDS)](#rds)
    - [2.1. So với Cài đặt CSDL trên EC2](#rds-vs-ec2)
    - [2.2. Các Khái niệm Chính trong RDS](#rds-concept)
    - [2.3. Kết nối](#connect-to-rds)
    - [2.4. Sao lưu và Phục hồi](#backup-recovery)
- [Tài liệu tham khảo](#reference)

<a name = "sql-nosql"></a>

## 1. SQL và NoSQL

Trong phần này, cùng nhắc lại một chút kiến thức cơ bản về hai mô hình cơ sở dữ liệu chính hiện nay: SQL và NoSQL.

<a name = "sql"></a>

### 1.1. SQL

Viết tắt của *Structured Query Language* (Ngôn ngữ Truy vấn có Cấu trúc), là ngôn ngữ tiêu chuẩn để quản lý và thao tác với dữ liệu trong các cơ sở dữ liệu quan hệ (*Relational Database*). Do đó, hai thuật ngữ SQL Database và Relational Database thường được hiểu là một.

Cơ sở dữ liệu loại này **có cấu trúc rõ ràng, chặc chẽ**, với dữ liệu được tổ chức thành các bảng (*table*), mỗi bảng gồm các hàng (*row*) và cột (*column*). Mỗi bảng có một **khóa chính** (*primary key*) để xác định duy nhất mỗi hàng trong bảng, và các bảng có thể liên kết với nhau thông qua **khóa ngoại** (*foreign key*). Việc liên kết giữa các bảng này là lý do cho tên gọi *Relational* Database.

Dưới đây là một ví dụ về một cơ sở dữ liệu SQL với 3 bảng:

Bảng `Users`:

| **UserID** | Username  | Email             | CreatedAt           |
|--------|-----------|-------------------|---------------------|
| 1      | cvn   | cvn@awscoban.com | 2026-01-01 10:00:00 |
| 2      | admin   | admin@awscoban.com | 2026-01-02 11:30:00 |

- **UserID**: Khóa chính, xác định duy nhất mỗi người dùng.

Bảng `Products`:

| **ProductID** | ProductName | Price  | Stock |
|-----------|-------------|--------|-------|
| 11         | Laptop      | 1000.0 | 50    |
| 12         | Smartphone  | 500.0  | 100   |

- **ProductID**: Khóa chính, xác định duy nhất mỗi sản phẩm.

Bảng `Orders`:

| **OrderID** | **UserID** | **ProductID** | Quantity | OrderDate           |
|---------|--------|-------------|----------|---------------------|
| 101     | 1      | 11           | 1        | 2026-02-01 14:00:00 |
| 102     | 2      | 12           | 2        | 2026-02-02 15:30:00 |

- **OrderID**: Khóa chính, xác định duy nhất mỗi đơn hàng.
- **UserID**: Khóa ngoại, liên kết với bảng `Users`.
- **ProductID**: Khóa ngoại, liên kết với bảng `Products`.


Các bảng này có thể liên kết với nhau thông qua các khóa chính và khóa ngoại, ví dụ: `UserID` trong bảng `Orders` liên kết với `UserID` trong bảng `Users`. Điều này cho phép truy vấn dữ liệu liên quan giữa các bảng. 

<a name = "no-sql"></a>

### 1.2. NoSQL

Là thuật ngữ chung để chỉ các cơ sở dữ liệu không sử dụng mô hình quan hệ truyền thống. NoSQL thực chất bao gồm nhiều loại cơ sở dữ liệu khác nhau, như **Document Store** (MongoDB, CouchDB), **Key-Value Store** (Redis, DynamoDB), **Column Family Store** (Cassandra, HBase), và **Graph Database** (Neo4j, Amazon Neptune).

Điểm chính để phân biệt với SQL là **NoSQL không có cấu trúc dữ liệu chặt chẽ**. Dữ liệu trong NoSQL có thể được lưu trữ dưới nhiều định dạng khác nhau, như JSON, khóa-giá trị, bảng cột, đồ thị, v.v.. Điều này mang lại sự linh hoạt cao hơn trong việc lưu trữ và truy vấn dữ liệu, phù hợp cho các ứng dụng có dữ liệu phi cấu trúc hoặc bán cấu trúc.

<a name = "rds"></a>

## 2. Relational Database Service (RDS)

<a name = "rds-vs-ec2"></a>

### 2.1. So với Cài đặt CSDL trên EC2

Với cơ sở dữ liệu SQL, nếu muốn triển khai trên AWS, đương nhiên ta có thể cài đặt chúng trên một hoặc nhiều [EC2 Instance](/2025/12/16/ec2-fundamental), cũng giống như khi cài đặt trên máy chủ vật lý ở hạ tầng on-premises. Tuy nhiên, trong hầu hết các trường hợp, nếu không có yêu cầu đặc biệt nào, ta nên sử dụng **RDS** của AWS để quản lý, vì những lý do sau:

- **Hiệu năng cao**: RDS được tối ưu hóa để cung cấp hiệu năng cao cho các cơ sở dữ liệu SQL, với khả năng tự động điều chỉnh tài nguyên dựa trên nhu cầu sử dụng.
- **Quản lý đơn giản**: người dùng chỉ cần sử dụng, không cần lo lắng về việc cài đặt phần mềm, vá lỗi, nâng cấp, v.v.
- **Tự động sao lưu và phục hồi**: RDS hỗ trợ sao lưu tự động, tiết kiệm rất nhiều thời gian và công sức quản lý sao lưu thủ công. Cũng có thể dễ dàng phục hồi cơ sở dữ liệu về trạng thái trước đó, hoặc khi cần [phục hồi sau thảm họa](/2025/11/12/aws_infrastructure/#dr).

Các *yêu cầu đặc biệt* đề cập ở trên khi cần tự cài đặt cơ sở dữ liệu trên EC2 thay vì dùng RDS, bao gồm: cần tùy chỉnh sâu cấu hình cơ sở dữ liệu (tầng hệ điều hành), sử dụng các phiên bản cơ sở dữ liệu không được RDS hỗ trợ, hoặc đơn giản là do yêu cầu cấp trên.


<a name = "rds-concept"></a>

### 2.2. Các Khái niệm Chính trong RDS

- **DB Instance**: là một máy chủ cơ sở dữ liệu ảo trong RDS. Mỗi DB Instance có thể chứa một hoặc nhiều cơ sở dữ liệu, giống như một máy chủ cơ sở dữ liệu truyền thống. 
- **DB Instance Class**: xác định cấu hình phần cứng của DB Instance, bao gồm CPU, RAM, dung lượng lưu trữ, v.v. Tương tự như [EC2 Instance Type](/2025/12/16/ec2-fundamental#instance-type), có 4 loại chính: General Purpose, Memory Optimized, Compute Optimized, và Burstable Performance, với nhu cầu sử dụng phù hợp ứng với tên gọi.
- **Ổ lưu trữ**: RDS sử dụng [EBS](/2025/12/20/ebs) để lưu trữ dữ liệu của DB Instance. Có 3 loại: General Purpose SSD, Provisioned IOPS SSD, hoặc Magnetic (ổ HDD cũ, không khuyến khích sử dụng).
- **DB Engine**: là phần mềm cơ sở dữ liệu được cài đặt trên DB Instance. RDS hỗ trợ nhiều loại DB Engine phổ biến, như MySQL, PostgreSQL, MariaDB, Oracle, IBM Db2. **Mỗi DB Instance chỉ chạy được một loại DB Engine**.

<a name = "connect-to-rds"></a>

### 2.3. Kết nối


<p>
<image src="/assets/19_rds/rds.png" alt="RDS" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>

Mặc định, mỗi RDS DB Instance sẽ nằm trong **một AZ**. Tốt nhất nên tạo DB Instance trong một VPC và subnet, như đã đề cập trong [kiến trúc ứng dụng đa tầng](/2025/11/20/design-vpc#multi-tier-design), sau đó thêm [Security Group](/2025/11/27/sg-nacl#sg) liên quan để cho phép các EC2 Instance trong Security Group được kết nối đến DB Instance. RDS sẽ cung cấp một **endpoint** (địa chỉ IP hoặc DNS) và cổng (port) để kết nối. 

DB Instance cũng có thể kết nối từ mạng doanh nghiệp thông qua Direct Connect hoặc VPN.
Ngoài ra, RDS cũng hỗ trợ **public access**, tức cung cấp địa chỉ IP công khai cho DB Instance để cho phép kết nối từ bên ngoài VPC. Tuy nhiên, người dùng cần cân nhắc kỹ nếu muốn sử dụng tính năng này vì lý do bảo mật.
<!-- TODO: include link to DX, VPN -->

Cấu hình mặc định đơn AZ tất nhiên chỉ có khả năng phục hồi [AZ Resilience](/2025/11/12/aws_infrastructure/#resilience). Để tăng tính khả dụng, có thể cấu hình [Multi-AZ](/2026/01/10/rds-multi-az#multi-az-deployment) cho DB Instance, với một hoặc hai *Standby Instance* ở AZ khác sẵn sàng thay thế nếu DB Instance chính xảy ra vấn đề. Thiết lập này sẽ được thảo luận kỹ hơn trong bài tiếp theo.

<a name = "backup-recovery"></a>

### 2.3. Sao lưu và Phục hồi

RDS cung cấp hai phương thức sao lưu chính:
- **Sao lưu tự động** (**Automated Backup**): RDS tự động sao lưu toàn bộ DB Instance hàng ngày, và lưu giữ trong khoảng thời gian do người dùng chọn (gọi là *retention period*, từ 0 đến 35 ngày). Việc sao lưu không ảnh hưởng đến hiệu năng của DB Instance. Có thể tự động **sao lưu sang Region khác**. Khi xoá DB Instance, có thể chọn giữ lại các bản sao lưu tự động, nhưng chỉ trong thời gian còn lại của retention period. 

- **Snapshot thủ công** (**Manual Snapshot**): người dùng có thể tự tạo snapshot của DB Instance bất cứ lúc nào. Snapshot được tạo theo cơ chế tăng dần, giống như [EBS Snapshot](/2025/12/20/ebs#ebs-snapshot), tức chỉ snapshot đầu tiên là đầy đủ, các snapshot sau lưu các thay đổi so với snapshot trước đó, để tiết kiệm dung lượng lưu trữ. Snapshot thủ công sẽ được giữ cho đến khi người dùng xóa nó, không bị mất khi xoá DB Instance.

Các bản sao lưu theo hai cách này đều được lưu trên S3, nhưng AWS chỉ hiển thị các bản sao lưu khả dụng trên giao diện RDS, không thể truy cập chúng trực tiếp trên S3. 


Người dùng có thể sử dụng các bản sao lưu này để phục hồi DB Instance về bất cứ thời điểm nào nằm trong khoảng retention period (gọi là **point-in-time recovery**). Khi phục hồi, RDS sẽ tạo một DB Instance mới từ bản sao lưu (với endpoint riêng biệt), chạy lại tất cả các giao dịch để phục hồi trạng thái dữ liệu. Quá trình phục hồi có thể mất một khoảng thời gian tuỳ thuộc vào kích thước dữ liệu và số lượng giao dịch cần xử lý.

<a name = "rpo-rto"></a>

Khi thiết kế khả năng phục hồi của hệ thống, cần cân nhắc hai mục tiêu:
- **Recovery Time Objective (RTO)**: thời gian tối đa chấp nhận được để khôi phục hệ thống sau sự cố. Mục tiêu này tập trung vào giảm thời gian gián đoạn dịch vụ.
- **Recovery Point Objective (RPO)**: mức độ mất mát dữ liệu tối đa có thể chấp nhận được, tính bằng thời gian (ví dụ: mất tối đa 5 phút dữ liệu). Nói cách khác, đây là khoảng thời gian giữa lần sao lưu cuối cùng và thời điểm xảy ra sự cố.


<a name = "rds-custom"></a>

## 3. RDS Tuỳ chỉnh

Đây là một tính năng kết hợp giữa chạy DB Instance trên RDS và khả năng tùy chỉnh sâu như khi tự cài đặt CSDL trên EC2.
Với RDS, AWS quản lý phần lớn công việc bảo trì, vận hành, không thể kết nối sâu đến tầng hệ điều hành, thậm chí không thể SSH tới DB Instance. 

Với RDS Tuỳ chỉnh, người dùng có thể can thiệp sâu hơn vào một số khía cạnh, tại thời điểm viết bài chỉ hỗ trợ Oracle và SQL Server. Dưới đây là so sánh mô hình chia sẻ trách nhiệm (cũng là quyền lợi) giữa 3 lựa chọn triển khai CSDL: trên EC2, RDS tiêu chuẩn, và RDS Tuỳ chỉnh. Nguồn: [Tài liệu AWS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-custom.html).

| Tính năng                      | EC2      | RDS Tiêu chuẩn   | RDS Tuỳ chỉnh <br> (Oracle)      | RDS Tuỳ chỉnh <br> (SQL Server)           |
|:--------------------------------|:---------------------------|:--------------------|:----------------------|:-------------------------|
| Tối ưu ứng dụng                | Khách hàng           | Khách hàng         | Khách hàng                 | Khách hàng                          |
| Mở rộng quy mô                 | Khách hàng           | AWS                | Chia sẻ                    | Chia sẻ                             |
| Đảm bảo tính sẵn sàng cao      | Khách hàng           | AWS                | AWS                        | AWS                                 |
| Sao lưu cơ sở dữ liệu          | Khách hàng           | AWS                | Chia sẻ                    | AWS                                 |
| Vá lỗi phần mềm CSDL           | Khách hàng           | AWS                | Chia sẻ                    | Chia sẻ   |
| Cài đặt phần mềm CSDL          | Khách hàng           | AWS                | Chia sẻ                    | Chia sẻ   |
| Vá lỗi hệ điều hành            | Khách hàng           | AWS                | Khách hàng                 | Chia sẻ   |
| Cài đặt hệ điều hành           | Khách hàng           | AWS                | Chia sẻ                    | AWS                                 |
| Bảo trì Host                | AWS                  | AWS                | AWS                        | AWS                                 |
| Phần cứng             | AWS                  | AWS                | AWS                        | AWS                                 |
| Điện, mạng, làm mát            | AWS                  | AWS                | AWS                        | AWS                                 |

<a name = "reference"></a>


## Tài liệu tham khảo

1. [Tài liệu AWS: Giới thiệu về RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html)
2. [Quản lý sao lưu tự động](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ManagingAutomatedBackups.html)
3. [Quản lý snapshot thủ công](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ManagingManualBackups.html)
4. [Phục hồi DB Instance từ snapshot](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_RestoreFromSnapshot.html)
5. [RDS Tuỳ chỉnh](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-custom.html)

Tiếp theo, hãy tìm hiểu về cách triển khai **RDS Multi-AZ** để nâng cao tính khả dụng và khả năng chịu lỗi.