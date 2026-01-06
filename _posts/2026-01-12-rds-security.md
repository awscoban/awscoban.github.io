---
layout: post
title: "21. Bảo mật trong RDS"
title2: "Bảo mật trong RDS"
date: 2026-01-12
permalink: /2026/01/12/rds-security
categories: [RDS, Database]
tags: [RDS, Database]
img: /assets/21_rds_security/rds-iam-auth.png
summary: "Trong bài này, mình sẽ trình bày hai vấn đề bảo mật quan trọng trong RDS: xác thực quyền truy cập CSDL trong DB Instance dùng IAM, và mã hoá dữ liệu trong RDS, bao gồm cả mã hoá khi truyền tải và mã hoá khi lưu trữ."
---

Bài này trình bày hai vấn đề bảo mật quan trọng trong RDS: xác thực quyền truy cập CSDL trong DB Instance dùng IAM, và mã hoá dữ liệu trong RDS, bao gồm cả mã hoá khi truyền tải và mã hoá khi lưu trữ.

## Trong bài này:

- [1. Xác thực Quyền Truy cập CSDL trong RDS](#iam-auth)
    - [Tạo IAM Policy](#iam-policy)
    - [Sử dụng Token Xác thực](#iam-token)
- [2. Mã hoá Dữ liệu trong RDS](#rds-encryption)
    - [2.1. Mã hoá Dữ liệu khi Truyền tải](#in-transit)
    - [2.2. Mã hoá Dữ liệu khi Lưu trữ](#at-rest)
- [Tài liệu tham khảo](#reference)



<a name = "iam-auth"></a>


## 1. Xác thực Quyền Truy cập CSDL trong RDS

Bài toán cơ bản trong mọi dịch vụ là kiểm soát truy cập, để chỉ những danh tính được phép mới có thể thao tác với tài nguyên. Trong RDS, phương thức xác thực phổ biến nhất là sử dụng [IAM](/2025/11/07/iam). Quy trình như sau:


<p>
<image src="/assets/21_rds_security/rds-iam-auth.png" alt="RDS IAM Authentication" style="max-width:70%;height:auto;display:block;margin:0 auto;"/>
</p>

<a name="iam-policy"></a>

- Bước 1: Tạo một [IAM Policy](/2025/11/07/iam/#iam-policy) cho phép hành động `rds-db:connect` trên DB Instance cụ thể, trong Resource [ARN](/2025/11/07/iam/#arn) chỉ rõ username bên trong DB Instance đó.

Giả sử ta có một DB Instance chạy PostgresSQL với ID `db-instance-id-12345`, trong đó có một tài khoản CSDL tên là `awscoban`, được tạo và cấu hình xác thực IAM bằng lệnh SQL sau:

```sql
CREATE USER awscoban; 
GRANT rds_iam TO awscoban;
```

IAM Policy tương ứng sẽ như sau:

```json
{
    "Version":"2012-10-17",		 	 	 
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "rds-db:connect"
            ],
            "Resource": [
                "arn:aws:rds-db:us-east-1:111122223333:dbuser:db-instance-id-12345/awscoban"
            ]
        }
    ]
}
```

Sau khi tạo, gán IAM Policy này cho [IAM User](/2025/11/07/iam/#iam-user) hoặc [IAM Role](/2025/11/07/iam/#iam-role) tương ứng với ứng dụng hoặc người dùng cần truy cập CSDL trong RDS.



<a name="iam-token"></a>

- Bước 2: Khi ứng dụng muốn kết nối đến DB Instance, sử dụng AWS SDK hoặc CLI để tạo một **token tạm thời** _(**temporary authentication token**_) từ RDS, thời hạn 15 phút. IAM kiểm tra xem danh tính có quyền `rds-db:connect` với tài khoản CSDL trên DB Instance đó hay không.

- Bước 3: Nếu xác thực thành công, IAM trả token xác thực cho ứng dụng.

- Bước 4: Ứng dụng sử dụng token này như mật khẩu để kết nối đến DB Instance, cùng với tên tài khoản CSDL tương ứng.

Giả sử DB Instance trên có endpoint là `rdspostgres.123456789012.us-east-1.rds.amazonaws.com`, trên EC2 Instance chạy ứng dụng có thể dùng các lệnh CLI sau để tạo token xác thực và kết nối đến CSDL:

```sh
export RDSHOST="rdspostgres.123456789012.us-east-1.rds.amazonaws.com"

# Tạo token xác thực:

export AUTH_TOKEN="$(aws rds generate-db-auth-token --hostname $RDSHOST --port 5432 --region us-east-1 --username awscoban)"

# Sử dụng token để kết nối đến RDS với psql, tên tài khoản CSDL là `awscoban`, tên CSDL là `awscoban-database`:

psql "host=$RDSHOST port=5432 user=awscoban password=$AUTH_TOKEN dbname=awscoban-database"

```




Ngoài IAM, RDS cũng hỗ trợ xác thực bằng mật khẩu truyền thống, hoặc thông qua các dịch vụ bên ngoài như Kerberos, bạn đọc quan tâm có thể tham khảo thêm trong [tài liệu chính thức](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/database-authentication.html).


<a name = "rds-encryption"></a>

## 2. Mã hoá Dữ liệu trong RDS

RDS hỗ trợ mã hoá dữ liệu cả khi lưu trữ (**encryption at rest**) và khi truyền tải (**encryption in transit**).


<a name="in-transit"></a>

### 2.1. Mã hoá Dữ liệu khi Truyền tải

- Tại tầng ứng dụng: RDS sử dụng SSL/TLS để mã hoá kết nối giữa ứng dụng và DB Instance, người dùng có thể cài đặt bắt buộc sử dụng SSL/TLS.
- Tại tầng mạng: tất cả dữ liệu truyền trong mạng lưới AWS trên toàn cầu đều được mã hoá ở tầng vật lý (*physical layer*) trước khi rời khỏi hạ tầng AWS, bên ngoài các lớp mã hoá đã có.




<a name="at-rest"></a>

### 2.2. Mã hoá Dữ liệu khi Lưu trữ

Khi tạo DB Instance, có thể bật mã hoá dữ liệu khi lưu trữ và chọn một [KMS Key](/2025/12/03/kms#kms-key) tương ứng. Khi đó, toàn bộ dữ liệu trên DB Instance, và tất cả các bản sao lưu tự động hoặc snapshot thủ công đều được mã hoá. Vì các DB Instance sử dụng [EBS](/2025/12/20/ebs) để lưu trữ dữ liệu, việc mã hoá dữ liệu khi lưu trữ trong RDS cũng giống tính năng mã hoá của EBS, vẫn sử dụng quy trình *mã hoá phong bì* (*envelope encryption*) như đã thảo luận trong hai bài [KMS](#/2025/12/03/kms#data-encryption-key) và [EBS](/2025/12/20/ebs#ebs-encryption).


Ngoài ra, RDS cho SQL Server và Oracle hỗ trợ mã hoá dữ liệu ở cấp độ CSDL, gọi là **TDE** (**Transparent Data Encryption**). DB Engine của hai loại CSDL này tự động mã hoá dữ liệu khi lưu trữ và giải mã khi truy xuất, coi như một tầng mã hoá bổ sung phía trước tầng mã hoá của RDS (nếu có). 



Dưới đây là một vài lưu ý cần ghi nhớ:
- Mã hoá dữ liệu khi lưu trữ chỉ có thể bật khi tạo mới DB Instance, **không thể bật cho DB Instance đã có**. Tuy nhiên, có một cách đi đường vòng là tạo một snapshot của DB Instance (không mã hoá), rồi [**chọn mã hoá** khi sao chép snapshot đó](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_CopySnapshot.html#USER_CopyDBSnapshot), sau đó tạo DB Instance mới từ snapshot đã mã hoá này.

- Không thể tắt mã hoá dữ liệu một khi đã bật khi tạo DB Instance.

- Không thể tạo snapshot mã hoá từ DB Instance không bật mã hoá. 

- [Read Replica](/2026/01/10/rds-multi-az#read-replica) phải có trạng thái mã hoá (có hoặc không) giống như DB Instance. Không thể tạo Read Replica không mã hoá cho DB Instance có mã hoá và ngược lại.




<a name = "reference"></a>

## Tài liệu tham khảo

1. [Các Phương thức Xác thực Quyền Truy cập CSDL trong RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/database-authentication.html)
2. [Xác thực với IAM](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.html)
3. [Kết nối với RDS với Xác thực IAM qua AWS CLI và PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.Connecting.AWSCLI.PostgreSQL.html)
4. [Mã hoá Dữ liệu trong RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html)


Qua loạt bài vừa rồi, mình đã trình bày những kiến thức cơ bản và quan trọng nhất về RDS: các khái niệm nền tảng, cách triển khai, các tính năng nâng cao như Multi-AZ, Read Replica, đến xác thực IAM và mã hoá dữ liệu. Những kiến thức này rất hay gặp cả trong thực tế lẫn các kỳ thi chứng chỉ AWS. Tiếp theo, hãy nói về một dịch vụ CSDL SQL được thiết kế và phát triển bởi chính AWS: **Aurora**.

