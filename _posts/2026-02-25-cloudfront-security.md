---
layout: post
title: "35. Bảo mật trong CloudFront"
title2: "Bảo mật trong CloudFront"
date: 2026-02-25 
permalink: /2026/02/25/cloudfront-security
categories: [CloudFront]
tags: [CloudFront]
img: /assets/35_cloudfront_security/cloudfront-https.png
summary: "Bài này tập trung vào các khía cạnh bảo mật trong CloudFront, bao gồm việc sử dụng HTTPS, thiết lập Origin Access Control (OAC) để ngăn chặn truy cập trực tiếp Origin, và sử dụng Signed URL và Signed Cookie để kiểm soát truy cập nội dung riêng tư."
---

Bài này tập trung vào các khía cạnh bảo mật trong CloudFront, bao gồm việc sử dụng HTTPS, thiết lập Origin Access Control (OAC) để ngăn chặn truy cập trực tiếp Origin, và sử dụng Signed URL và Signed Cookie để kiểm soát truy cập nội dung riêng tư.



## Trong bài này:

- [1. HTTPS trong CloudFront](#https)
    - [1.1. AWS Certificate Manager (ACM)](#acm)
    - [1.2. HTTPS trong CloudFront](#https-cloudfront)
- [2. Origin Access Control - OAC](#oac)
- [3. Signed URL và Signed Cookie](#signed-url-cookie)
- [Tài liệu tham khảo](#reference)


<a name = "https"></a>

## 1. HTTPS trong CloudFront

Ta đã biết CloudFront hỗ trợ cả HTTP và HTTPS trong giao tiếp giữa client và Edge Location. 
Tất nhiên, HTTPS an toàn hơn, vì dữ liệu được mã hoá trong quá trình truyền tải. Trong phần này, ta sẽ tìm hiểu cách thiết lập HTTPS trong CloudFront, và một vài điểm cần lưu ý khi ôn thi chứng chỉ AWS.

Trước tiên, hãy nhắc lại một chút kiến thức cơ bản. HTTPS là giao thức mạng có mã hoá khi truyền tải. Việc mã hoá này dựa trên giao thức bảo mật SSL/TLS, trong đó cần có SSL/TLS Certificate để xác thực danh tính máy chủ và thiết lập mã hoá kết nối. 

SSL Certificate đơn giản là một tệp dữ liệu được cài đặt trên máy chủ của tên miền, giúp trình duyệt xác thực danh tính máy chủ và thiết lập mã hoá kết nối. Được cấp bởi các tổ chức uy tín gọi chung là Certificate Authority (CA), thường có thời hạn nhất định, cần gia hạn hoặc cấp phát lại để tiếp tục sử dụng.

Về mặt kỹ thuật, việc xác thực danh tính máy chủ qua SSL Certificate như sau:
- Chủ sở hữu tên miền gửi yêu cầu cấp SSL Certificate đến CA, kèm theo các thông tin cần thiết. Gọi là *Certificate Signing Request (CSR)*.
- CA sử dụng **private key** để ký số lên CSR, tạo thành SSL Certificate. Sau đó gửi SSL Certificate cho chủ sở hữu tên miền.
- SSL Certificate chứa chữ ký số của CA được cài đặt trên máy chủ tên miền. 
- Khi trình duyệt kết nối đến máy chủ qua HTTPS, máy chủ gửi SSL Certificate cho trình duyệt. Client sử dụng **public key** của CA (được cài sẵn trong trình duyệt) để **xác thực chữ ký số** trên SSL Certificate. Nếu hợp lệ (tức đúng là đã được ký bởi private key của CA), trình duyệt tin tưởng đây đúng là máy chủ đã được xác nhận bởi CA, và bắt đầu thiết lập kết nối mã hoá (gọi là *TLS Handshake*).

<a name = "acm"></a>

### 1.1. AWS Certificate Manager (ACM)

ACM là dịch vụ quản lý SSL Certificate của AWS. Ta có thể dùng ACM để:
- Tạo mới SSL Certificate miễn phí cho tên miền của mình (ACM cũng là một CA được công nhận). SSL Certificate do ACM cấp sẽ được tự động gia hạn.
- Nhập SSL Certificate từ CA bên ngoài, nhưng cần tự quản lý việc gia hạn.
- Sử dụng SSL Certificate trong các dịch vụ AWS như CloudFront, [Elastic Load Balancer](/2026/02/05/load-balancer), [API Gateway](/2026/01/28/api-gateway).

Lưu ý, SSL Certificate được tạo/nhập vào ACM ở Region nào thì chỉ có thể sử dụng trong các dịch vụ AWS ở Region đó. Ngoại trừ **các dịch vụ toàn cầu (*global service*) như CloudFront, sẽ dùng SSL Certificate tại Region `us-east-1`**.

Ngoài ACM, cũng có thể sử dụng [IAM Certificate Store](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_server-certs.html) để quản lý SSL Certificate. Cách này không được khuyến khích (trừ trường hợp Region không hỗ trợ ACM), nhưng vẫn hay xuất hiện trong đề thi chứng chỉ, bạn đọc cần lưu ý để lựa chọn đúng.


<a name = "https-cloudfront"></a>

### 1.2. HTTPS trong CloudFront

CloudFront là trung gian giữa Client (cũng gọi là Viewer) và Origin. Do đó, có hai đoạn giao tiếp cần quan tâm: giữa Client và Edge Location, và giữa Edge Location và Origin. Cả hai đoạn này đều có thể sử dụng HTTPS. 


<p>
<image src="/assets/35_cloudfront_security/cloudfront-https.png" alt="CloudFront HTTPS" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>



#### HTTPS giữa Client và Edge Location

Như đã đề cập trong [bài trước](/2026/02/20/cloudfront#cache-behavior), khi tạo Distribution, ta có thể chọn HTTPS là giao thức giữa Client và Edge Location (*Viewer Protocol Policy*). 

Nếu Distribution sử dụng tên miền mặc định của CloudFront (có dạng `d123456abcdef8.cloudfront.net`), CloudFront tự động cài đặt SSL Certificate cho tên miền này tại tất cả Edge Location, không cần cấu hình gì thêm. Khi đó chỉ cần chọn *Viewer Protocol Policy* là `Redirect HTTP to HTTPS` hoặc `HTTPS Only` để buộc Client sử dụng HTTPS. Khi Client gửi yêu cầu tới tên miền, quá trình xác thực và thiết lập kết nối mã hoá diễn ra như đã trình bày ở [phần trên](#https).

<a name = "alternative-domain-name"></a>

Nếu cần sử dụng tên miền riêng (*alternate domain name*, ví dụ: `cdn.awscoban.com`) để tăng độ nhận diện và phục vụ SEO, cần tự cài đặt SSL Certificate cho tên miền này. Có thể tạo SSL Certificate từ [ACM](#acm) (tại Region `us-east-1`) hoặc IAM Certificate Store. Cũng có thể nhập SSL Certificate từ CA bên ngoài, miễn là đúng tên miền định sử dụng. Ví dụ, có thể dùng lệnh CLI dưới đây để tạo SSL Certificate cho tên miền `cdn.awscoban.com` trong ACM:

```sh
aws acm request-certificate \
--domain-name cdn.awscoban.com \
--key-algorithm EC_Prime256v1 \
--validation-method DNS \
--idempotency-token 1234 \
--options CertificateTransparencyLoggingPreference=DISABLED,Export=ENABLED
```

Sau khi tạo, cần xác thực quyền sở hữu tên miền (`validation-method`, khuyến khích dùng `DNS`). Nếu tên miền được quản lý trong [Route 53](/2026/02/14/route-53), AWS tự động tạo DNS [CNAME Record](/2026/02/14/route-53#dns-record-type) cần thiết để xác thực. Nếu không, cần tạo thủ công CNAME Record theo hướng dẫn.


Trở lại CloudFront, khi cấu hình Distribution cho tên miền riêng, chỉ cần chọn tên miền và SSL Certificate tương ứng là được.


#### HTTPS giữa Edge Location và Origin
Lúc này CloudFront đóng vai trò là Client, Origin đóng vai trò là Server. Tại Origin cần có SSL Certificate hợp lệ, tuỳ loại Origin mà cần cấu hình khác nhau:

- Đối với S3 Bucket: chỉ hỗ trợ HTTPS nếu dùng Bucket thông thường (tên miền có dạng `bucket-name.s3.amazonaws.com`), CloudFront tự động chấp nhận SSL Certificate của S3. **Nếu dùng Bucket làm website, không hỗ trợ HTTPS**.
- Đối với AWS Service Origin (như [API Gateway](/2026/01/28/api-gateway), [Elastic Load Balancer](/2026/02/05/load-balancer)): có thể dùng SSL Certificate trong ACM hoặc IAM Certificate Store. Lưu ý, SSL Certificate phải ở trong cùng Region.
- Đối với Custom Origin (server riêng): SSL Certificate cần được cấp bởi CA, không chấp nhận tự ký (*self-signed*). Tên miền trong SSL Certificate phải khớp với tên miền của Origin.





<a name = "oac"></a>

## 2. Origin Access Control - OAC

Như tên gọi, OAC giúp ngăn chặn Client truy cập trực tiếp đến Origin. Khi OAC được kích hoạt, tất cả yêu cầu từ Client đến Origin đều phải qua CloudFront.


<p>
<image src="/assets/35_cloudfront_security/cloudfront-oac.png" alt="CloudFront OAC" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>

OAC có thể áp dụng cho tất cả các loại Origin, nhưng phổ biến nhất là với S3 Bucket. Cơ chế cũng khá đơn giản:

- Tạo OAC trên CloudFront, liên kết OAC với Origin tương ứng. Sau đó, AWS tạo một [Bucket Policy](/2025/12/06/s3-security#bucket-policy) để cho phép CloudFront truy cập S3 Bucket  ([Explicit Allow](/2025/11/07/iam#explicit-deny)), có dạng như sau:

```json
{
  "Version":"2012-10-17",		 	 	 
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipalReadOnly",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": [
        "s3:GetObject",
      ],
      "Resource": "arn:aws:s3:::awscoban-website/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::111122223333:distribution/CloudFront-distribution-ID>"
        }
      }
    }
  ]
}
```

- Sao chép Bucket Policy đó vào S3 Bucket. Khi này, chỉ có CloudFront mới có quyền truy cập S3 Bucket, Client không thể truy cập trực tiếp nữa (do cơ chế [Implicit Deny](/2025/11/07/iam#explicit-deny)).

- Nếu Bucket sử dụng [SSE-KMS](/2025/12/06/s3-security#sse-kms) để mã hoá object với [KMS Key](/2025/12/03/kms#kms), cần cấp quyền sử dụng KMS Key cho CloudFront trong [Key Policy](/2025/12/03/kms#key-policy):

```json
{
    "Sid": "AllowCloudFrontServicePrincipalSSE-KMS",
    "Effect": "Allow",
    "Principal": {
        "Service": [
            "cloudfront.amazonaws.com"
        ]
     },
    "Action": [
        "kms:Decrypt",
        "kms:Encrypt",
        "kms:GenerateDataKey*"
    ],
    "Resource": "*",
    "Condition": {
            "StringEquals": {
                "AWS:SourceArn": "arn:aws:cloudfront::111122223333:distribution/<CloudFront distribution ID>"
            }
        }
}
```


<a name = "signed-url-cookie"></a>

## 3. Signed URL và Signed Cookie

Nhiều dịch vụ Internet cung cấp nội dung trả phí dành riêng cho một nhóm người dùng (*paywall*), cần kiểm soát quyền truy cập nội dung. CloudFront hỗ trợ hai cơ chế để làm việc này: **Signed URL** và **Signed Cookie**. Chúng đều dựa trên cơ chế chữ ký số (*digital signature*) để xác thực quyền truy cập nội dung.

- **Signed URL**: một URL chứa chữ ký số, cho phép truy cập **từng tệp đơn lẻ**. Có dạng như sau: `https://cdn.awscoban.com/private_video.mp4?Expires=1672531200&Signature=abc123...&Key-Pair-Id=K12345`.
- **Signed Cookie**: dùng khi **không muốn thay đổi URL gốc**, hoặc muốn cấp quyền truy cập cho **nhiều tệp cùng lúc**. Sau khi xác nhận người dùng có quyền truy cập nội dung, ứng dụng gửi kèm Cookie chứa chữ ký số đến trình duyệt của người dùng. Cookie này được trình duyệt gửi kèm trong các yêu cầu tới CloudFront.


**Cơ chế hoạt động** như sau:

1. Đầu tiên, nên tạo [OAC](#oac) để ngăn chặn truy cập trực tiếp đến Origin.
2. Tạo cặp *public - private key*. Private key dùng để ký số URL hoặc Cookie, public key dùng để xác thực chữ ký số. Tải **public key** lên CloudFront. Giữ lại **private key** để sử dụng trong mã nguồn.
3. Trong Distribution, chỉ định *trusted key group* hoặc *trusted signer* là một hoặc nhiều public key đã tải lên. 
4. **Tại ứng dụng**, tự xác thực quyền truy cập dữ liệu của người dùng. Nếu được phép, tạo **Signed URL** hoặc **Signed Cookie** bằng cách dùng private key để ký số đường dẫn tới dữ liệu. Gửi Signed URL hoặc Signed Cookie cho người dùng.
5. Người dùng gửi yêu cầu tới CloudFront kèm theo Signed URL hoặc Signed Cookie. CloudFront xác thực chữ ký số bằng public key đã chỉ định trong Distribution. Nếu hợp lệ, CloudFront lấy dữ liệu từ Origin và trả về cho người dùng.






<a name = "reference"></a>

## Tài liệu tham khảo

1. [Cách thức Hoạt động của SSL](https://www.cloudflare.com/learning/ssl/how-does-ssl-work/)
2. [AWS Certificate Manager (ACM)](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html)
3. [IAM Certificate Store](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_server-certs.html)
2. [Sử dụng HTTPS trong CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-https.html)
3. [Sử dụng Tên miền Riêng trong CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-https-alternate-domain-names.html)
4. [Origin Access Control trong CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)
5. [Sử dụng Signed URL và Signed Cookie để Phân phối Dữ liệu Riêng tư trong CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/PrivateContent.html)

Hai bài vừa rồi đã tóm lược những kiến thức quan trọng trong CloudFront. Bạn đọc có thể thực hành thêm qua các bài lab trên [AWS Skill Builder](https://explore.skillbuilder.aws/learn/course/external/view/elearning/16553/cloudfront). Tiếp theo, ta sẽ tìm hiểu về IaC (Infrastructure as Code) trong AWS với CloudFormation.
