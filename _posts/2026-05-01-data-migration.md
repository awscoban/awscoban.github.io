---
layout: post
title: "46. Di chuyển Dữ liệu lên AWS"
title2: "Di chuyển Dữ liệu lên AWS"
date: 2026-05-01
permalink: /2026/05/01/data-migration
categories: [Storage, Hybrid, Database]
tags: [Storage, Hybrid, Database]
img: /assets/46_data_migration/datasync.png
summary: "Bài này giới thiệu các dịch vụ và công cụ của AWS để di chuyển dữ liệu từ hạ tầng nội bộ lên AWS, bao gồm 2 phương pháp: online dùng DataSync và offline dùng Snowball. Ngoài ra cũng đề cập cách di chuyển cơ sở dữ liệu lên AWS với Database Migration Service (DMS)."
---

Bài này giới thiệu các dịch vụ và công cụ của AWS để di chuyển dữ liệu từ hạ tầng nội bộ lên AWS, bao gồm 2 phương pháp: online dùng **DataSync** và offline dùng **Snowball Edge**. Ngoài ra cũng đề cập cách di chuyển cơ sở dữ liệu lên AWS với **Database Migration Service (DMS)**. 


## Trong bài này:

- [1. Di chuyển Online với DataSync](#online)
- [2. Di chuyển Offline với Snowball Edge](#offline)
    - [2.1. Snowball Edge](#snowball-edge)
    - [2.2. Snowball và Snowmobile (ngừng hoạt động)](#deprecated-snow)
- [3. Di chuyển CSDL với Database Migration Service (DMS)](#dms)
    - [3.1. Tổng quan](#dms-overview)
    - [3.2. Schema Conversion Tool](#sct)
    - [3.3. Kết hợp DMS với Snowball Edge](#dms-snowball)
- [Tài liệu tham khảo](#reference)


<a name = "online"></a>

## 1. Di chuyển Online với DataSync

Đây là phương pháp truyền dữ liệu qua kết nối mạng từ hạ tầng nội bộ hoặc các nền tảng cloud khác lên AWS, thông qua Internet, [VPN](/2026/04/12/vpn) hay [Direct Connect](/2026/04/18/direct-connect).

Ở phía nguồn, DataSync hỗ trợ hầu hết các định dạng lưu trữ: NFS cho Linux, SMB cho Windows, Hadoop Distributed File Systems (HDFS), và các [object storage](/2025/11/30/s3-fundamental#storage-comparison) ở hạ tầng nội bộ. 

Ở phía đích, tất nhiên DataSync sẽ hỗ trợ các dịch vụ lưu trữ của AWS, gồm [S3](/2025/11/30/s3-fundamental), [EFS](/2025/12/24/efs), FSx.
<!-- TODO: include link to FSx-->


Các trường hợp sử dụng phổ biến:
- Chuyển dữ liệu đang sử dụng từ hạ tầng nội bộ lên AWS:
    - Lưu trữ dữ liệu ít truy cập (*cold data*) trên các dịch vụ lưu trữ lạnh như [S3 Glacier](/2025/12/09/s3-storage-classes#s3-glacier) hoặc [Glacier Deep Archive](/2025/12/09/s3-storage-classes#s3-glacier-deep-archive). Việc này giúp giải phóng dung lượng lưu trữ ở hạ tầng nội bộ.
    - Sao lưu dữ liệu để tối ưu chi phí, tận dụng [các lớp lưu trữ trong S3](/2025/12/09/s3-storage-classes).
- Di chuyển dữ liệu giữa các dịch vụ lưu trữ của AWS, hoặc giữa các Region khác nhau.
- Chuyển dữ liệu **ra khỏi AWS**.

<p>
<image src="/assets/46_data_migration/datasync.png" alt="DataSync" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>

Hình trên mô tả cách DataSync di chuyển dữ liệu từ hạ tầng nội bộ lên AWS. 
- Tại hạ tầng nội bộ, người dùng cài đặt **DataSync Agent** để sao chép dữ liệu từ hệ thống lưu trữ nội bộ.
- DataSync Agent sẽ mã hoá dữ liệu (sử dụng TLS) và truyền qua kết nối mạng (Internet, [VPN](/2026/04/12/vpn) hay [Direct Connect](/2026/04/18/direct-connect)) đến endpoint của DataSync Service trên AWS. 
- DataSync ghi dữ liệu vào dịch vụ lưu trữ đích.




<a name = "offline"></a>

## 2. Di chuyển Offline với Snowball Edge

<a name = "snowball-edge"></a>

### 2.1. Snowball Edge

Với các yêu cầu **di chuyển dữ liệu cực lớn (cỡ petabyte)**, việc truyền qua mạng sẽ mất rất nhiều thời gian. Trong trường hợp này, có thể cân nhắc Snowball Edge. Đây là một chiếc vali chống sốc chứa ổ cứng chuyên dụng. 

<p>
<image src="/assets/46_data_migration/SnowballEdgeAppliance.png" alt="Snowball Edge" style="max-width:30%;height:auto;display:block;margin:0 auto;"/>
</p>

*Thiết bị Snowball Edge. Nguồn: [AWS](https://docs.aws.amazon.com/snowball/latest/developer-guide/receive-device.html)*


Cách thức hoạt động rất đơn giản: 
- Người dùng gửi yêu cầu.
- AWS gửi thiết bị Snowball Edge đến địa chỉ người dùng.
- Người dùng sao chép dữ liệu vào thiết bị.
- Người dùng gửi trả thiết bị về AWS.
- AWS tải dữ liệu từ thiết bị vào đúng dịch vụ lưu trữ đích của tài khoản người dùng.


Ngoài lưu trữ đơn thuần, Snowball Edge còn **có khả năng tính toán (compute)**, cho phép **xử lý dữ liệu ngay trên thiết bị** trước khi gửi về AWS. Có 2 loại Snowball Edge:

- **Storage Optimized**: dung lượng lưu trữ 210 TB (SSD NVMe), CPU AMD 64 core, RAM 416 GB. Đây là cấu hình tại thời điểm viết bài.
- **Compute Optimized**: dung lượng lưu trữ 28 TB (SSD NVMe), CPU AMD 64 core, RAM 416 GB. Cấu hình tính toán giống Storage Optimized, chỉ ít dung lượng lưu trữ hơn, nhưng bù lại chi phí rẻ hơn.



<a name = "deprecated-snow"></a>

### 2.2. Snowball và Snowmobile (ngừng hoạt động)

**Snowball**: trước đây AWS cung cấp phiên bản Snowball tiêu chuẩn, 50 TB hoặc 80 TB, đã dừng triển khai từ 2020. Không có khả năng tính toán như Snowball Edge, chỉ đơn thuần là một thiết bị lưu trữ di động trữ lượng lớn. 


**Snowmobile**: dịch vụ này đã ngừng hoạt động từ 2024. Đây là một xe tải cỡ lớn, chứa phần cứng có thể lưu trữ tới 100 PB dữ liệu. AWS trực tiếp đưa xe tải này đến hạ tầng nội bộ, khách hàng sao chép dữ liệu vào xe tải, sau đó đưa về trung tâm dữ liệu để tải dữ liệu của khách hàng lên hạ tầng đám mây. Snowmobile chỉ phù hợp cho các khối lượng dữ liệu cực lớn, như hàng trăm PB hoặc thậm chí EB, mà việc truyền qua mạng là không khả thi về thời gian, cũng như cần rất rất nhiều lần sử dụng Snowball Edge để hoàn thành. 



<a name = "dms"></a>

## 3. Di chuyển CSDL với Database Migration Service (DMS)

Như tên gọi, DMS là dịch vụ chuyên dụng để di chuyển CSDL từ hạ tầng nội bộ hoặc các nền tảng cloud khác lên AWS, và ngược lại từ AWS ra ngoài.

<a name = "dms-overview"></a>

### 3.1. Tổng quan


<p>
<image src="/assets/46_data_migration/dms.png" alt="Database Migration Service" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>

Về cơ bản, DMS là 1 [EC2 Instance](/2025/12/16/ec2-fundamental) chạy phần mềm sao chép dữ liệu (gọi là **Replication Instance**), thực hiện song song các tác vụ sao chép (**Replication Task**) từ nguồn đến đích.

Replication Instance kết nối đến CSDL nguồn và đích qua 2 **Endpoint** tương ứng. Người dùng cung cấp thông tin kết nối đến CSDL (như loại engine, hostname, port, username, password) cho cả 2 Endpoint. Một Endpoint có thể phục vụ nhiều task.

Cấu hình quan trọng nhất trong Replication task là **loại di chuyển (migration type)**. Có 3 loại:
- **Full Load**: di chuyển toàn bộ dữ liệu trong **một lần duy nhất**. Nhưng nếu có dữ liệu mới phát sinh trong quá trình di chuyển, sẽ không được cập nhật. Phù hợp cho các trường hợp di chuyển dữ liệu cũ, hay dữ liệu chỉ đọc (*read-only*).

- **CDC (Change Data Capture)**: theo dõi và sao chép các sự kiện **thay đổi dữ liệu** (insert, update, delete) trên CSDL nguồn sang đích. Phù hợp cho các trường hợp đã di chuyển sẵn lần đầu, cần đồng bộ liên tục. Cơ chế là theo dõi transaction log của CSDL nguồn và sao chép các thao tác sang CSDL đích.

- **Full Load + CDC**: kết hợp cả 2 phương pháp trên. Đầu tiên thực hiện full load để di chuyển dữ liệu lần đầu, trong lúc đó tiếp tục theo dõi và ghi lại các thay đổi dữ liệu, rồi cập nhật để đồng bộ dần. Phù hợp khi di chuyển CSDL lớn trong môi trường production, cần giảm tối đa downtime.


Với CSDL bên ngoài, DMS hỗ trợ nhiều loại engine phổ biến, như Oracle, SQL Server, MySQL, MariaDB, PostgreSQL, MongoDB, v.v. Bên trong AWS DMS hỗ trợ tất cả dịch vụ CSDL như [RDS](/2026/01/08/rds-fundamental), [Aurora](/2026/01/15/aurora), [DynamoDB](/2026/03/18/dynamodb), Redshift, hay thậm chí [S3](/2025/11/30/s3-fundamental).


<a name = "sct"></a>

### 3.2. Schema Conversion Tool

Một tính năng rất hay của DMS là **Schema Conversion Tool** (SCT), giúp chuyển đổi schema khi CSDL nguồn và đích **có Engine khác nhau**, ví dụ như từ Oracle sang PostgreSQL, hoặc từ SQL Server sang [Aurora](/2026/01/15/aurora).
SCT cũng có thể dùng cho CSDL loại OLAP, chuyển đổi các kho dữ liệu lớn từ Teradata hoặc Vertica sang Amazon Redshift.

Bạn đọc có thể tham khảo chi tiết cách cài đặt và cấu hình SCT trong [tài liệu chính thức của AWS](https://docs.aws.amazon.com/SchemaConversionTool/latest/userguide/CHAP_Welcome.html).


<a name = "dms-snowball"></a>

### 3.3. Kết hợp DMS với Snowball Edge

Trong trường hợp cần di chuyển CSDL cực lớn, có thể kết hợp DMS với Snowball Edge. Cách thức hoạt động như sau:
- Yêu cầu AWS cấp Snowball Edge, kết nối vào hạ tầng nội bộ.
- Cài đặt [SCT](#sct) trên máy chủ hạ tầng nội bộ, kết nối với CSDL nguồn. SCT sẽ lấy dữ liệu, chuyển đổi schema và xuất dữ liệu vào Snowball Edge.
- Chuyển thiết bị Snowball Edge về AWS. AWS sẽ tải dữ liệu trong Snowball Edge vào một S3 Bucket người dùng chỉ định.
- Tạo DMS Replication Task để sao chép dữ liệu từ S3 Bucket vào CSDL đích trên AWS. Migration Type là **Full Load + CDC**:
    - Full Load lấy từ S3 Bucket.
    - CDC cập nhật thay đổi dữ liệu mới phát sinh trên CSDL nguồn. Chỉ cần kết nối DMS với hạ tầng nội bộ qua mạng thông thường cho CDC, vì lượng dữ liệu mới cần đồng bộ không lớn so với dữ liệu đã tải vào Snowball Edge.




<a name = "reference"></a>

## Tài liệu tham khảo

1. [Giới thiệu về DataSync](https://docs.aws.amazon.com/datasync/latest/userguide/what-is-datasync.html)
2. [AWS CLI cho DataSync](https://docs.aws.amazon.com/cli/latest/reference/datasync/)
3. [Cách Hoạt động của Snowball](https://docs.aws.amazon.com/snowball/latest/developer-guide/how-it-works.html)
4. [FAQs về Snowball](https://aws.amazon.com/snowball/faqs/)
5. [So sánh Storage Optimized và Compute Optimized Snowball Edge](https://docs.aws.amazon.com/snowball/latest/developer-guide/device-differences.html#device-options)
6. [Các Thành phần trong DMS](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Introduction.Components.html)
7. [Schema Conversion Tool (SCT)](https://docs.aws.amazon.com/SchemaConversionTool/latest/userguide/CHAP_Welcome.html)
8. [Kết hợp DMS với Snowball Edge](https://aws.amazon.com/blogs/storage/enable-large-scale-database-migrations-with-aws-dms-and-aws-snowball)


Trong phần DataSync, mình có đề cập đến FSx như một dịch vụ lưu trữ đích. Hãy cùng tìm hiểu sâu hơn trong bài tiếp theo.