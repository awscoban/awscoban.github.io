---
layout: post
title: "50. CloudTrail, Config và Control Tower"
title2: "CloudTrail, Config và Control Tower"
date: 2026-05-25
permalink: /2026/05/25/infra-management
categories: [Account & Organization, Infrastructure, Security]
tags: [Account & Organization, Infrastructure, Security]
img: /assets/50_infra_mgnt/config.png
summary: "Trong môi trường tổ chức, việc quản lý và giám sát hoạt động của tài khoản AWS là rất quan trọng để đảm bảo an toàn, tuân thủ và tối ưu chi phí. Bài này giới thiệu 3 dịch vụ chính giúp thực hiện chức năng này: CloudTrail, AWS Config và Control Tower."
---

Trong môi trường tổ chức, việc quản lý và giám sát hoạt động của tài khoản AWS là rất quan trọng để đảm bảo an toàn, tuân thủ và tối ưu chi phí. Bài này giới thiệu 3 dịch vụ chính giúp thực hiện chức năng này: **CloudTrail**, **AWS Config** và **Control Tower**.


## Trong bài này:

- [1. CloudTrail](#cloudtrail)
    - [1.1. Event](#cloudtrail-event)
    - [1.2. Trail](#trail)
- [2. AWS Config](#config)
    - [2.1. Config Rule](#config-rule)
    - [2.2. Remediation Action](#remediation)
    - [2.3. Conformance Pack](#conformance-pack)
- [3. Control Tower](#control-tower)
    - [3.1. Landing Zone](#landing-zone)
    - [3.2. Account Factory](#account-factory)
    - [3.3. Guardrail](#guardrail)
- [Tài liệu tham khảo](#reference)


<a name = "cloudtrail"></a>

## 1. CloudTrail

CloudTrail **ghi lại các lần gọi API trong tài khoản AWS**, như ai đã gọi API, khi nào, từ đâu, v.v. Bất cứ thao tác nào trên giao diện, CLI, SDK đều tạo ra một *event* được ghi lại trong CloudTrail. Việc này giúp theo dõi hoạt động trong tài khoản AWS, hỗ trợ việc kiểm tra và tuân thủ các quy định về bảo mật.


Các khái niệm chính:

<a name = "cloudtrail-event"></a>

### 1.1. Event

Là một bản ghi về một lần gọi API, bao gồm thông tin về người gọi, thời gian, dịch vụ được gọi, tham số liên quan, v.v.

Có 2 loại event:

- **Management Event**: ghi lại các **thao tác trên tài nguyên** (**resource-level**), ví dụ:
    - Tạo, sửa, xoá tài nguyên như [EC2 Instance](/2025/12/16/ec2-fundamental), [S3 Bucket](/2025/11/30/s3-fundamental), v.v.
    - Cấu hình bảo mật, như gán [IAM Role](/2025/11/07/iam#iam-role) cho một dịch vụ, v.v.

- **Data Event**: ghi lại các **thao tác trên dữ liệu** (**data-level**), ví dụ:
    - Đọc, ghi, xoá S3 object.
    - Kích hoạt [Lambda Function](/2026/01/18/lambda-fundamental).


Ví dụ, dưới đây là một Management Event khi gọi API `StartInstances` của EC2:


```json
{"Records": [{
    "eventVersion": "1.08",
    "userIdentity": {
        "type": "IAMUser",
        "principalId": "awscoban-principal-id-sample",
        "arn": "arn:aws:iam::123456789012:user/awscoban",
        "accountId": "123456789012",
        "accessKeyId": "awscoban-access-key-id-sample",
        "userName": "awscoban",
        "sessionContext": {
            "sessionIssuer": {},
            "webIdFederationData": {},
            "attributes": {
                "creationDate": "2023-07-19T21:11:57Z",
                "mfaAuthenticated": "false"
            }
        }
    },
    "eventTime": "2023-07-19T21:17:28Z",
    "eventSource": "ec2.amazonaws.com",
    "eventName": "StartInstances",
    "awsRegion": "us-east-1",
    "sourceIPAddress": "192.0.2.0",
    "userAgent": "aws-cli/2.13.5 Python/3.11.4 Linux/4.14.255-314-253.539.amzn2.x86_64 exec-env/CloudShell exe/x86_64.amzn.2 prompt/off command/ec2.start-instances",
    "requestParameters": {
        "instancesSet": {
            "items": [
                {
                    "instanceId": "i-abcdefgh"
                }
            ]
        }
    },
    "responseElements": {
        "requestId": "e4336db0-149f-4a6b-844d-EXAMPLEb9d16",
        "instancesSet": {
            "items": [
                {
                    "instanceId": "i-abcdefgh",
                    "currentState": {
                        "code": 0,
                        "name": "pending"
                    },
                    "previousState": {
                        "code": 80,
                        "name": "stopped"
                    }
                }
            ]
        }
    },
    "requestID": "e4336db0-149f-4a6b-844d-EXAMPLEb9d16",
    "eventID": "e755e09c-42f9-4c5c-9064-EXAMPLE228c7",
    "readOnly": false,
    "eventType": "AwsApiCall",
    "managementEvent": true,
    "recipientAccountId": "123456789012",
    "eventCategory": "Management",
     "tlsDetails": {
        "tlsVersion": "TLSv1.2",
        "cipherSuite": "ECDHE-RSA-AES128-GCM-SHA256",
        "clientProvidedHostHeader": "ec2.us-east-1.amazonaws.com"
    },
    "sessionCredentialFromConsole": "true"
}]}
```


Mặc định, CloudTrail chỉ ghi lại Management Event. Để ghi lại Data Event, bạn cần chọn thêm khi [tạo Trail](#trail).


<a name = "trail"></a>

### 1.2. Trail

Trail là một cấu hình trong CloudTrail để xác định cách ghi lại và lưu trữ event. 

Khi tạo Trail, ta có thể:

- Chỉ định S3 Bucket nào sẽ lưu trữ event, cùng với [mã hoá SSE-KMS](/2025/12/06/s3-security#sse-kms) nếu muốn.
- Chọn ghi lại Management Event, Data Event, hoặc cả hai.
- Cấu hình [CloudWatch Logs](/2026/01/05/cloudwatch#cloudwatch-logs) để giám sát, phân tích và cảnh báo dựa trên event.
- **Log file validation**: đảm bảo rằng event log không bị thay đổi sau khi ghi lại. Bản chất là lưu lại hash của tệp log gốc, và khi cần kiểm tra, so sánh hash gốc với hash của tệp log hiện tại để xác định xem có bị thay đổi hay không.



<a name = "config"></a>

## 2. AWS Config

AWS Config là dịch vụ giúp giám sát, đánh giá và quản lý các cấu hình của tài nguyên AWS:

- Liên tục **giám sát** và ghi lại các thay đổi cấu hình tài nguyên AWS.
- **Đánh giá**: so sánh cấu hình hiện tại của tài nguyên với cấu hình mong muốn.
- **Quản lý** tuân thủ (**compliance**): xác định mức độ tuân thủ của toàn bộ hệ thống, dựa trên các nguyên tắc hoặc tiêu chuẩn bảo mật nội bộ. Điều này giúp đơn giản hóa việc kiểm toán, phân tích bảo mật, quản lý thay đổi và khắc phục sự cố.


Ví dụ, một vài trường hợp sử dụng phổ biến:

- Bảo mật mạng: tự động phát hiện và xoá các [Security Group](/2025/11/27/sg-nacl#sg) đang mở cổng 22 (SSH) hoặc 3389 (RDP) cho toàn bộ Internet (`0.0.0.0/0`).
- Tối ưu chi phí: phát hiện các ổ [EBS](/2025/12/20/ebs) không được sử dụng (*unattached volume*), hoặc các Instance quá cỡ so với nhu cầu thực tế.
- Phát hiện cấu hình sai quy định: S3 Bucket không được mã hoá, hoặc có quyền truy cập công khai, v.v.



<a name = "config-rule"></a>

### 2.1. Config Rule

Là các quy tắc đánh giá để đối chiếu, xác định xem tài nguyên có tuân thủ hay không. 

Có 2 loại Rule:
- **AWS Managed Rule**: do AWS cung cấp sẵn, tập trung vào các quy tắc tuân thủ phổ biến, như kiểm tra xem S3 bucket có được mã hoá hay không, v.v.
- **Custom Rule**: do người dùng tự định nghĩa, tạo bởi[Lambda Function](/2026/01/18/lambda-fundamental) hoặc [CloudFormation Guard](https://github.com/aws-cloudformation/cloudformation-guard), một ngôn ngữ mô tả chính sách (*policy-as-code*).


<p>
<image src="/assets/50_infra_mgnt/config.png" alt="AWS Config Rule" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>


Có 2 **kết quả đánh giá** chính:
- **Compliant**: tài nguyên tuân thủ quy tắc.
- **Non-Compliant**: tài nguyên không tuân thủ quy tắc.

Ví dụ, Managed Rule [`encrypted-volumes`](https://docs.aws.amazon.com/config/latest/developerguide/encrypted-volumes.html) kiểm tra xem ổ [EBS](/2025/12/20/ebs) có được mã hoá hay không. Nếu một ổ EBS không được mã hoá, nó sẽ bị đánh dấu là **Non-Compliant**.


Rule có thể được **kích hoạt** bởi AWS Config theo 2 cách:
- **Periodic**: đánh giá theo lịch trình định kỳ, ví dụ mỗi 24 giờ.
- **Configuration Change**: đánh giá ngay khi tài nguyên đang theo dõi có thay đổi về cấu hình.


<a name = "remediation"></a>

### 2.2. Remediation Action

Khi tài nguyên bị đánh dấu là Non-Compliant, AWS Config có thể tự động thực hiện một **hành động khắc phục** (**_remediation action_**), như xoá tài nguyên, sửa lại cấu hình, v.v., giúp giảm thiểu rủi ro bảo mật và đảm bảo tuân thủ liên tục.

Nhiều Remediation Action được AWS tạo sẵn trong [Systems Manager (SSM) Automation](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-automation.html). Automation là một dịch vụ trong SSM, tương tự như [Parameter Store](/2026/05/12/secret-manager#parameter-store), nhưng tập trung vào việc tự động hoá các tác vụ quản lý hệ thống.
Người dùng cũng có thể tự tạo các Remediation Action tuỳ chỉnh bằng cách viết một SSM Automation Document, định nghĩa các bước cần thực hiện để khắc phục.


<a name = "conformance-pack"></a>

### 2.3. Conformance Pack

Đây đơn giản là tập hợp các Rule và Remediation Action được đóng gói sẵn, giúp triển khai nhanh các quy tắc tuân thủ phổ biến, như PCI DSS, HIPAA, v.v. Với các tổ chức lớn, Conformance Pack giúp việc giám sát và quản lý đa tài khoản và đa vùng dễ dàng hơn.

Bạn đọc có thể tham khảo danh sách Conformance Pack do AWS cung cấp [tại đây](https://docs.aws.amazon.com/config/latest/developerguide/conformancepack-sample-templates.html).



<a name = "control-tower"></a>

## 3. Control Tower

Control Tower giúp **thiết lập và quản lý môi trường đa tài khoản** trên AWS, tuân thủ tiêu chuẩn mà không cần phải cấu hình thủ công quá nhiều. 

<a name = "landing-zone"></a>

### 3.1. Landing Zone

Landing Zone là một [Organization](/2025/11/06/aws-account#organization) được thiết kế sẵn, dựa trên cách cấu hình tốt nhất về quản lý và bảo mật. Cấu trúc của Landing Zone bao gồm:

<p>
<image src="/assets/50_infra_mgnt/landing-zone.png" alt="Landing Zone" style="max-width:60%;height:auto;display:block;margin:0 auto;"/>
</p>


- [Tài khoản quản lý (*Management Account*)](/2025/11/06/aws-account#management-account): tài khoản chính để quản lý và thanh toán, như trong một Organization bình thường.
- **Root OU**: chứa tất cả các tài khoản thành viên và OU con.
- **Security OU**: chứa 2 tài khoản tạo sẵn:
    - **Audit Account**: tài khoản khoản dành cho đội ngũ bảo mật/kiểm toán để xem xét trạng thái tuân thủ toàn hệ thống.
    - **Log Archive Account**: tập trung tất cả log từ [CloudTrail](#cloudtrail), [Config](#config), của toàn bộ các tài khoản con (tăng khả năng kiểm soát, tránh việc xóa log để xoá dấu vết).
- **Sandbox OU**: dành cho các tài khoản thử nghiệm, phát triển, v.v., không ảnh hưởng đến môi trường production.
- Các OU khác do người dùng tự tạo theo nhu cầu.


Control Tower sử dụng [CloudFormation StackSet](/2026/03/12/cnf-stack-advanced#stackset) để triển khai tài nguyên đến các tài khoản.


<a name = "account-factory"></a>

### 3.2. Account Factory

Như tên gọi, đây là công cụ để **tạo tài khoản mới đồng nhất theo mẫu có sẵn**. Thay vì tạo thủ công, Account Factory giúp cấu hình tự động VPC, Subnet, IAM Roles, và các quy tắc bảo mật, đảm bảo tất cả tài khoản mới đều tuân thủ các chính sách và tiêu chuẩn của tổ chức. 


<a name = "guardrail"></a>

### 3.3. Guardrail

Tên gọi khác là **control**, đây là các quy tắc quản trị cấp cao được áp dụng cho toàn bộ OU và tài khoản. 

Phân loại **theo chức năng**, có 3 loại chính:

- **Preventive**: **ngăn chặn** hành động nào đó, **không cho phép** thực hiện, và ghi lại sự kiện đó trong CloudTrail. Ví dụ, guardrail `Disallow Changes to Bucket Policy for Amazon S3 Buckets` sẽ ngăn chặn việc thay đổi Bucket Policy của S3 Bucket, thường áp dụng cho Log Archive Account để đảm bảo log không bị thay đổi.

- **Detective**: **phát hiện** hành động nào đó, vẫn **cho phép** thực hiện nhưng gửi cảnh báo (ví dụ qua [SNS](/2026/01/22/step-function#sns)). Hành động cũng được ghi lại trong CloudTrail. Ví dụ, có một guardrail là `Detect Whether Encryption is Enabled for Amazon EBS Volumes Attached to Amazon EC2 Instances`, sẽ cảnh báo nếu một ổ EBS *không được mã hoá* được gắn vào EC2 Instance.

- **Proactive**: gần giống Detective, nhưng thay vì gửi cảnh báo, nó sẽ tự động thực hiện một hành động khắc phục (Remediation Action) để sửa lỗi ngay lập tức.


Phân loại **theo phạm vi áp dụng**, có 2 loại guardrail:
- **Mandatory**: áp dụng cho tất cả tài khoản và OU, không thể tắt hay thay đổi.
- **Elective**: có thể chọn áp dụng hoặc không tuỳ nhu cầu.




<a name = "reference"></a>

## Tài liệu tham khảo

1. [Các Thuật ngữ trong CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-concepts.html)
2. [Management Event](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-management-events-with-cloudtrail.html)
3. [Data Event](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-data-events-with-cloudtrail.html)
4. [Giới thiệu về AWS Config](https://docs.aws.amazon.com/config/latest/developerguide/WhatIsConfig.html)
5. [AWS Config Rule](https://docs.aws.amazon.com/config/latest/developerguide/evaluate-config_components.html)
6. [Các Mẫu Conformance Pack cho AWS Config](https://docs.aws.amazon.com/config/latest/developerguide/conformancepack-sample-templates.html)
7. [Các Tính năng của Control Tower](https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html)
8. [Cấu trúc của Landing Zone](https://docs.aws.amazon.com/controltower/latest/userguide/how-control-tower-works.html)


Trong bài này, ta đã tìm hiểu về bảo mật ở cấp độ quản lý. Tiếp theo, hãy thảo luận về bảo mật ở cấp độ dữ liệu, ứng dụng và hạ tầng cụ thể, với các dịch vụ liên quan như WAF, Macie, Shield, Inspector, hay GuardDuty.
