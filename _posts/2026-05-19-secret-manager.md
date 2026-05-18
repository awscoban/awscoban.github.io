---
layout: post
title: "49. Lưu trữ Bí mật trên AWS"
title2: "Lưu trữ Bí mật trên AWS"
date: 2026-05-19
permalink: /2026/05/19/secret-manager
categories: [Security]
tags: [Security]
img: 
summary: "Bài này giới thiệu 2 dịch vụ giúp lưu trữ bí mật và tham số cấu hình cho ứng dụng trên AWS là System Manager Parameter Store và Secrets Manager. Ngoài ra bổ sung thêm một dịch vụ mã hoá tương tự KMS là CloudHSM."
---

Bài này giới thiệu 2 dịch vụ giúp lưu trữ bí mật và tham số cấu hình cho ứng dụng trên AWS là **System Manager Parameter Store** và **Secrets Manager**. Ngoài ra bổ sung thêm một dịch vụ mã hoá tương tự KMS là **CloudHSM**.


## Trong bài này:

- [1. System Manager Parameter Store](#parameter-store)
- [2. Secrets Manager](#secret-manager)
- [3. CloudHSM](#cloudhsm)
- [Tài liệu tham khảo](#reference)


<a name = "parameter-store"></a>

## 1. System Manager Parameter Store

System Manager là một hệ thống quản lý các tác vụ trên AWS. Một dịch vụ hay dùng trong System Manager là **Parameter Store**, dùng để lưu trữ các tham số cấu hình (**configuration parameters**) và các loại bí mật (**secret**) như mật khẩu, API key, v.v.
<!-- TODO: include link to System Manager-->

Parameter Store lưu tham số dưới dạng các cặp khoá-giá trị (**key-value**), hỗ trợ 3 loại dữ liệu:
- **String**: chuỗi văn bản đơn giản, như tên ứng dụng, URL, v.v.
- **StringList**: danh sách các chuỗi, dùng khi cần lưu một nhóm giá trị, như danh sách IP, tên miền, v.v.
- **SecureString**: chuỗi nhưng sẽ được **mã hoá** bằng [KMS](/2025/12/03/kms#kms), dùng để lưu mật khẩu, API key, v.v.

Ví dụ, dưới đây ta lưu một tham số dạng String dùng AWS CLI (có thể dùng SDK hoặc thao tác trên giao diện):

```bash
aws ssm put-parameter \
    --name "website" \
    --value "awscoban.com" \
    --type String \
    --region us-east-1
```

Tham số sẽ có các khoá: `Name`, `Value`, `Type`, `Version`, `LastModifiedDate`, `ARN`, v.v. với giá trị tương ứng, có thể sử dụng để truy vấn. Ví dụ, ta tìm tham số theo khoá `Name`, với giá trị khoá này là `website` như sau:


```bash
aws ssm describe-parameters \
    --parameter-filters "Key=Name,Values=website" \
    --region us-east-1
```

Kết quả trả về là một vài thông tin về tham số:

```json
{
    "Parameters": [
        {
            "Name": "website",
            "ARN": "arn:aws:ssm:us-east-1:123456789012:parameter/website",
            "Type": "String",
            "LastModifiedDate": "2026-03-16T14:42:11.002000+07:00",
            "LastModifiedUser": "arn:aws:iam::123456789012:user/iamadmin",
            "Version": 1,
            "Tier": "Standard",
            "Policies": [],
            "DataType": "text"
        }
    ]
}
```

Khoá hay dùng nhất tất nhiên là `Value` (giá trị của tham số), không được trả về bởi lệnh `describe-parameters`, mà phải dùng lệnh `get-parameter` để truy vấn (lệnh này và các phiên bản SDK tương ứng được dùng nhiều trong ứng dụng để lấy tham số khi cần):

```bash
aws ssm get-parameter \
    --name "website" \
    --region us-east-1
```

Kết quả:

```json
{
    "Parameter": {
        "Name": "website",
        "Type": "String",
        "Value": "awscoban.com",
        "Version": 1,
        "LastModifiedDate": "2026-03-16T14:42:11.002000+07:00",
        "ARN": "arn:aws:ssm:us-east-1:123456789012:parameter/website",
        "DataType": "text"
    }
}
```


Với tham số dạng StringList, danh sách các giá trị được phân tách bằng dấu phẩy, **không có khoảng trắng**:

```bash
aws ssm put-parameter \
    --name "/blog/contents" \
    --value "aws,security,ssm" \
    --type StringList \
    --region us-east-1
```

Tham số này có tên `/blog/contents`. Parameter Store hỗ trợ tổ chức tham số theo dạng phân tầng (**hierarchical**) như cây thư mục để dễ quản lý. Cơ chế tương tự như [S3 Object Key](/2025/11/30/s3-fundamental#s3-object).


Một tham số **chỉ có duy nhất một khoá `Name`**. Parameter Store hỗ trợ **phiên bản**, với khoá `Version`. Và tất nhiên có thể lấy giá trị phiên bản mong muốn.

Parameter Store cũng được tích hợp [IAM](/2025/11/07/iam) để kiểm soát truy cập, chi tiết đến từng tham số (với khoá `Policies` ở trên). 



<a name = "secret-manager"></a>

## 2. Secrets Manager

Dịch vụ này được thiết kế chuyên biệt để lưu trữ các bí mật như mật khẩu, API key, v.v. 


Trong Secrets Manager, một **secret** bao gồm thông tin về các trường dữ liệu (**key**) và giá trị (**value**) tương ứng. Giá trị có thể là chuỗi ký tự, hoặc ở dạng nhị phân. Để lưu nhiều giá trị chuỗi ký tự liên quan đến nhau trong một secret, nên dùng định dạng JSON. Ví dụ:

```json
{
  "host"       : "prod-server.databases.awscoban.com",
  "port"       : "8888",
  "username"   : "admin",
  "password"   : "ĐÃ LÀM MỜ",
  "dbname"     : "awscoban-db",
  "engine"     : "postgresql"
}
```

Tạo secret này dùng AWS CLI như sau:

```bash
aws secretsmanager create-secret \
    --name "prod/db/awscoban" \
    --description "Database credentials for production environment" \
    --secret-string '{"host":"prod-server.databases.awscoban.com","port":"8888","username":"admin","password":"ĐÃ LÀM MỜ","dbname":"awscoban-db","engine":"postgresql"}' \
    --region us-east-1
```

Có thể lưu chuỗi JSON thành một tệp rồi dùng tham số `--secret-string file://path/to/secret.json` để tạo secret từ tệp này, thay vì phải gõ chuỗi JSON trên dòng lệnh.

Kết quả trả về:

```json
{
    "ARN": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/db/awscoban-randomstring",
    "Name": "prod/db/awscoban",
    "VersionId": "a-random-string-for-version-id"
}
```

Secrets Manager cũng hỗ trợ phiên bản, với khoá `VersionId`. Có thể lấy giá trị của một phiên bản cụ thể bằng cách sử dụng khoá này.

```bash
aws secretsmanager get-secret-value \
    --secret-id "prod/db/awscoban" \
    --version-id "a-random-string-for-version-id" \
    --region us-east-1
```

Lệnh này trả về chuỗi JSON giá trị của secret.


Ưu điểm so với Parameter Store là Secrets Manager có tính năng tự động xoay vòng (**automatic rotation**), và tích hợp sẵn với nhiều dịch vụ AWS khác như [RDS](/2026/01/12/rds-security). Đây là 2 điểm cần lưu ý khi lựa chọn giữa hai dịch vụ này, hay xuất hiện trong các câu hỏi thi chứng chỉ.



<a name = "cloudhsm"></a>

## 3. CloudHSM

Parameter Store và Secret Manager là 2 dịch vụ lưu bí mật ở **tầng ứng dụng**. Các bí mật này được mã hoá bởi **khoá mã hoá** ở tầng thấp hơn.
CloudHSM là dịch vụ quản lý các khoá mã hoá này, tương tự như [KMS](/2025/12/03/kms).

**Hardware Security Module (HSM)** là một module phần cứng, lưu trữ khoá và thực hiện các phép toán sinh khoá và mã hoá một cách biệt lập, cung cấp mức độ bảo mật cao hơn so với dùng phần mềm. CloudHSM là dịch vụ HSM được quản lý bởi AWS, cho phép người dùng tạo và quản lý khoá mã hoá như đang thao tác với một HSM chuyên dụng.

Với KMS, dù AWS cũng sử dụng một HSM chuyên dụng cho các thao tác mã hoá, tuy nhiên điểm mấu chốt là **HSM của KMS được chia sẻ cho nhiều khách hàng** (đương nhiên vẫn đảm bảo tính bảo mật và cách ly dữ liệu).
Còn **với CloudHSM, khách hàng có toàn quyền sử dụng một HSM riêng**, không chia sẻ với người khác. Điều này mang lại lợi ích về mặt bảo mật, nhưng cũng đi kèm với chi phí cao hơn so với KMS.

Các trường hợp sử dụng phổ biến của CloudHSM:

- Các tổ chức có yêu cầu tuân thủ nghiêm ngặt về bảo mật, cần dịch vụ chuẩn **FIPS 140-2 Level 3**.
- Transparent Data Encryption (TDE) cho CSDL Oracle.
- Xử lý SSL/TLS cho các ứng dụng web, giảm tải (*offload*) cho máy chủ ứng dụng.
- Quản lý private key cho [certificate authority (CA)](/2026/02/25/cloudfront-security#acm).
- Tích hợp với KMS để tạo [khoá mã hoá tùy chỉnh](/2025/12/03/kms#kms-key), với mức độ bảo mật cao hơn.


<a name = "reference"></a>

## Tài liệu tham khảo

1. [Sử dụng Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-working-with.html)
2. [Giới thiệu về Secret Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
3. [Trường hợp Sử dụng CloudHSM](https://docs.aws.amazon.com/cloudhsm/latest/userguide/use-cases.html)

Tiếp theo, hãy tìm hiểu các dịch vụ giúp bảo vệ dữ liệu, ứng dụng, và hạ tầng trên AWS.