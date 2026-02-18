---
layout: post
title: "33. Định tuyến DNS trong Route 53"
title2: "Định tuyến DNS trong Route 53"
date: 2026-02-17
permalink: /2026/02/17/dns-routing
categories: [Route 53, Networking]
tags: [Route 53, Networking]
img: 
summary: "Một tên miền có thể trỏ tới nhiều địa chỉ IP. Khi cần phân giải tên miền, việc lựa chọn trả về IP nào được gọi là định tuyến DNS (DNS Routing). Route 53 hỗ trợ nhiều kiểu định tuyến DNS khác nhau, tuỳ yêu cầu cụ thể, cùng tìm hiểu chi tiết dưới đây."
---

Một tên miền có thể trỏ tới **nhiều** địa chỉ IP. Khi cần [phân giải tên miền](/2026/02/14/route-53#dns-resolution), việc lựa chọn trả về IP nào được gọi là **định tuyến DNS** (DNS Routing). Route 53 hỗ trợ nhiều kiểu định tuyến DNS khác nhau, tuỳ yêu cầu cụ thể, cùng tìm hiểu chi tiết dưới đây. Mình sẽ giữ nguyên thuật ngữ tiếng Anh để bạn đọc dễ tra cứu và nhận diện.


## Trong bài này:

- [1. Simple Routing](#simple-routing)
- [2. Failover Routing](#failover-routing)
- [3. Geolocation Routing](#geolocation-routing)
- [4. Geoproximity Routing](#geoproximity-routing)
- [5. Latency Routing](#latency-routing)
- [6. IP Routing](#ip-routing)
- [7. Multivalue Answer Routing](#multivalue-routing)
- [8. Weighted Routing](#weighted-routing)
- [Tài liệu tham khảo](#reference)


<a name = "simple-routing"></a>

## 1. Simple Routing

Như tên gọi, đây là kiểu định tuyến đơn giản, trực tiếp trỏ tên miền tới một hoặc nhiều IP của tài nguyên.
Trong cùng một [DNS Record](/2026/02/14/route-53#dns-record-type), ta có thể khai báo nhiều địa chỉ IP. Khi phân giải tên miền, DNS Resolver sẽ trả về tất cả các IP này cho client. Client tự chọn một IP để kết nối.

Lưu ý, các IP trong Simple Routing **không được** [Health Check](/2026/02/14/route-53#health-check). Nếu một IP không khả dụng, cần tự xử lý phía client.


Ví dụ, ta có tên miền `awscoban.com` trong Hosted Zone, và 3 máy chủ cho trang web tại 3 IPv4 `10.0.2.10`, `10.0.2.11`, `10.0.2.12`. 
Có thể sử dụng Simple Routing, tạo một DNS Record loại A để gán 3 IP này cho tên miền.
Bạn đọc có thể thao tác trên giao diện, SDK, hoặc CLI. Ở đây, để hiểu rõ bản chất, mình sẽ dùng CLI. 

Đầu tiên, ta tạo một tệp cấu hình dạng JSON (ví dụ: `simple_routing.json`):


```json
{
  "Comment": "Simple Routing với nhiều IP cho một tên miền",
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "awscoban.com",
        "Type": "A",
        "TTL": 60,
        "ResourceRecords": [
          { "Value": "10.0.2.10" },
          { "Value": "10.0.2.11" },
          { "Value": "10.0.2.12" }
        ]
      }
    }
  ]
}
```

Trong đó:
- `Action`: hành động với DNS Record, ở đây là tạo mới (`CREATE`).
- `ResourceRecordSet`: DNS Record cần tạo, bao gồm:
    - `Name`: tên miền cần phân giải.
    - `Type`: loại DNS Record, ở đây là A (cho 3 địa chỉ IPv4 của 3 máy chủ).
    - `TTL`: "Time To Live", thời gian lưu Record trong bộ nhớ đệm của DNS Resolver (tính bằng giây), để không cần phân giải tên miền quá thường xuyên.
    - `ResourceRecords`: danh sách các địa chỉ IP tương ứng với tên miền.


Sau đó, dùng tệp cấu hình này trong [lệnh CLI sau](https://docs.aws.amazon.com/cli/latest/reference/route53/change-resource-record-sets.html) để tạo Record trong Hosted Zone của tên miền:

```sh
aws route53 change-resource-record-sets \
    --hosted-zone-id <HOSTED_ZONE_ID> \
    --change-batch file://simple_routing.json # đường dẫn tới tệp cấu hình trên
```

Nên dùng Simple Routing khi:
- Tên miền chỉ trỏ tới **một tài nguyên duy nhất**.
- Muốn phân phối tải đơn giản giữa nhiều tài nguyên mà không cần kiểm tra tình trạng.


<a name = "failover-routing"></a>

## 2. Failover Routing

Đây là kiểu định tuyến giúp chuyển đổi dự phòng (*failover*) sang tài nguyên phụ khi tài nguyên chính ở trạng thái [Unhealthy](/2026/02/14/route-53#health-check). 
Khác với Simple Routing, Failover Routing là cơ chế định tuyến đặc thù trong Route 53, cho phép trỏ tên miền tới hai tài nguyên: chính (**Primary**) và phụ (**Secondary**).

Một cấu hình phổ biến của Failover Routing là Active-Passive Failover, trong đó, một tài nguyên chính phục vụ hầu hết nhu cầu (*active*), và một tài nguyên phụ được kích hoạt khi tài nguyên chính gặp sự cố (*passive*). Cho ứng dụng web, tài nguyên chính là máy chủ web, tài nguyên phụ là một trang web tĩnh (*static website*) hiển thị nội dung cố định.

Ví dụ, dưới đây là cấu hình JSON cho DNS Record sử dụng Failover Routing, trỏ tên miền `awscoban.com` tới máy chủ web chính tại một IP, và tới tài nguyên phụ là một [S3 Bucket](/2025/11/30/s3-fundamental) được cấu hình làm trang web tĩnh:

```json
{
  "Comment": "Dùng Failover Routing cho Active-Passive Failover",
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "awscoban.com",
        "Type": "A",
        "SetIdentifier": "PrimaryServer",
        "Failover": "PRIMARY",
        "TTL": 60,
        "ResourceRecords": [{ "Value": "10.0.2.10" }],
        "HealthCheckId": "abcdef12-3456-7890-abcd-ef1234567890"
      }
    },
    {   
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "awscoban.com",
        "Type": "A",
        "SetIdentifier": "S3FailoverPage",
        "Failover": "SECONDARY",
        "AliasTarget": {
          "HostedZoneId": "Z3AQBSTGFYJSTF",
          "DNSName": "s3-website-us-east-1.amazonaws.com",
          "EvaluateTargetHealth": false
        }
      }
    }  
]}
```

Trong đó, ta tạo hai DNS Record cho cùng tên miền `awscoban.com`.
- Record chính (**Primary**) trỏ tới IP của máy chủ web, có kèm `HealthCheckId` để theo dõi trạng thái.
- Record phụ (**Secondary**) trỏ tới trang web tĩnh trên S3, sử dụng [Alias Record](/2026/02/14/route-53#r53-alias), trong đó trường `HostedZoneId` là ID của Hosted Zone của S3 ở `us-east-1`. ID này cố định cho từng dịch vụ ở từng Region, dùng chung cho tất cả tài khoản, có thể tra cứu [tại đây](https://docs.aws.amazon.com/general/latest/gr/s3.html).


Sau đó, dùng tệp cấu hình này trong lệnh CLI `route53 change-resource-record-sets`, tương tự như Simple Routing, để tạo Record trong Hosted Zone của tên miền.


<a name = "geolocation-routing"></a>

## 3. Geolocation Routing

Đây là một kiểu định tuyến dựa trên vị trí địa lý của người dùng. 
Khi sử dụng Geolocation Routing, mỗi DNS Record sẽ có thêm thông tin về khu vực địa lý: châu lục, quốc gia, tiểu bang (áp dụng cho Hoa Kỳ).
Trong quá trình phân giải tên miền, Route 53 xác định vị trí địa lý của client (dựa trên địa chỉ IP), và tìm DNS Record **liên quan**:
- Cùng tiểu bang (nếu có).
- Cùng quốc gia (nếu không có tiểu bang).
- Cùng châu lục (nếu không có quốc gia).


Kiểu định tuyến này hữu ích khi muốn cung cấp nội dung phù hợp với từng khu vực (đơn giản như sử dụng ngôn ngữ bản địa), hoặc tuân thủ quy định tại từng quốc gia.
Lưu ý, Geolocation Routing **không phải trả về Record gần nhất**, đó là chức năng của Geoproximity Routing dưới đây. 


<a name = "geoproximity-routing"></a>

## 4. Geoproximity Routing

Đây mới là kiểu định tuyến dựa trên **khoảng cách địa lý** (*proximity*) giữa người dùng và tài nguyên. Khi tạo DNS Record, ta chỉ định vị trí địa lý của tài nguyên:
- Nếu là tài nguyên AWS, vị trí là Region.
- Nếu không phải tài nguyên AWS, vị trí là kinh độ và vĩ độ.

Ngoài ra, có thể sử dụng một tham số gọi là **bias** để điều chỉnh định tuyến đến một tài nguyên nhiều hơn hoặc ít hơn. Tham số này có thể dương (mở rộng vùng) hoặc âm (thu hẹp vùng):

```
Khoảng cách hiệu dụng = Khoảng cách thực tế * [1 - (bias/100)]
```

Trong đó, khoảng cách thực tế là khoảng cách địa lý giữa người dùng và tài nguyên, còn khoảng cách hiệu dụng là giá trị được sử dụng để xác định tài nguyên gần nhất: định tuyến đến tài nguyên có khoảng cách hiệu dụng **nhỏ nhất**.

Ví dụ, nếu có hai tài nguyên A và B, với khoảng cách thực tế tới người dùng lần lượt là 120 km và 100 km. Nếu đặt bias của tài nguyên A là 20:

```
Khoảng cách hiệu dụng A = 120 km * [1 - (20/100)] = 96 km
Khoảng cách hiệu dụng B = 100 km * [1 - (0/100)] = 100 km
```

Do đó, người dùng sẽ được định tuyến đến tài nguyên A, mặc dù về địa lý, tài nguyên B gần hơn.


Ảnh hưởng của bias được minh hoạ rất trực quan trong [tài liệu AWS](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geoproximity.html), bạn đọc có thể đọc thêm.


<a name = "latency-routing"></a>

## 5. Latency Routing

Như tên gọi, Latency Routing định tuyến theo độ trễ. Áp dụng khi triển khai ứng dụng ở nhiều Region khác nhau, muốn người dùng kết nối tới tài nguyên ở Region có độ trễ thấp nhất.

Dữ liệu độ trễ được AWS thu thập và quản lý. Khi phân giải tên miền, Route 53 so sánh độ trễ từ vị trí người dùng tới các Region có tài nguyên, và trả về địa chỉ IP trong DNS Record ở Region có độ trễ thấp nhất.


<a name = "ip-routing"></a>

## 6. IP Routing

Trong một vài trường hợp, so với [Geolocation Routing](#geolocation-routing) nếu muốn tinh chỉnh việc định tuyến hơn nữa, có thể sử dụng IP Routing. Ví dụ:
- Muốn điều hướng người dùng từ một nhà cung cấp dịch vụ Internet (ISP) cụ thể (xác định bằng dải IP) tới một tài nguyên nhất định.
- Muốn tích hợp dữ liệu tự thu thập về vị trí người dùng, không chỉ dựa trên dữ liệu từ AWS.

IP Routing định tuyến dựa trên IP của người dùng. Ở DNS Record, ta chỉ định [CIDR](/2025/11/13/vpc#cidr) cho từng tài nguyên. Khi phân giải tên miền, Route 53 so sánh IP của người dùng với các dải CIDR đã khai báo, và trả về giá trị (IP cho tên miền) trong DNS Record tương ứng. Nếu không khớp với CIDR nào, Route 53 trả về giá trị mặc định.   


Do dựa trên IP của người dùng (là public IP), kiểu định tuyến này không áp dụng cho [private hosted zone](/2026/02/14/route-53#hosted-zone).



<a name = "multivalue-routing"></a>

## 7. Multivalue Answer Routing

Khác với [Simple Routing](#simple-routing) chỉ cho phép **tạo 1 DNS Record** với cùng `Name` và `Type`, dù vẫn có thể có nhiều `Value` (là IP), Multivalue Answer Routing cho phép **tạo nhiều DNS Record** với cùng `Name` và `Type`. Ví dụ:


| Name | Type | Value | Set ID | Health Check |
|:---|:---|:---|:---|:---|
| awscoban.com | A | 10.0.0.11 | Server1 | HC-1 |
| awscoban.com | A | 10.0.0.12 | Server2 | HC-2 |
| awscoban.com | A | 10.0.0.13 | Server3 | HC-3 |

Hơn nữa, mỗi Record có thể kèm theo một [Health Check](/2026/02/14/route-53#health-check). Khi phân giải tên miền, Route 53 sẽ trả về **tối đa 8 IP** trong Record có trạng thái **Healthy**. Nếu tất cả Record đều **Unhealthy**, Route 53 trả về 8 IP để client tự xử lý.

Việc trả về nhiều IP cũng giúp phân phối tải giữa các tài nguyên, tuy nhiên, Multivalue Answer Routing không phải là một cơ chế cân bằng tải thực thụ như [Elastic Load Balancer](/2026/02/05/load-balancer).



<a name = "weighted-routing"></a>

## 8. Weighted Routing

Đây là dạng tổng quát hơn của [Multivalue Answer Routing](#multivalue-routing), trong đó mỗi DNS Record có thêm trọng số. Khi phân giải tên miền, Route 53 sẽ sử dụng trọng số này để xác định lựa chọn Record nào.

Ví dụ, giả sử ta cấu hình Weighted Routing cho `awscoban` với 3 Record sau:

| Name | Type | Value | Set ID | Health Check | Weight | 
|:---|:---|:---|:---|:---|:---|
| awscoban.com | A | 10.0.0.11 | Server1 | HC-1 | 20 |
| awscoban.com | A | 10.0.0.12 | Server2 | HC-2 | 30 |
| awscoban.com | A | 10.0.0.13 | Server3 | HC-3 | 50 |

<br>

Máy chủ ở IP `10.0.0.11` sẽ nhận: \\( \frac{20}{20+30+50} = 20\% \\) lưu lượng.


Cách định tuyến này hữu ích khi muốn phân phối tải đơn giản, hoặc dùng cho thử nghiệm phiên bản mới.


<a name = "reference"></a>

## Tài liệu tham khảo

1. [Simple Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-simple.html)
2. [Failover Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-failover.html)
3. [Các loại DNS Failover](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-types.html)
4. [Geolocation Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geo.html)
5. [Geoproximity Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geoproximity.html)
6. [Latency Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-latency.html)
7. [IP Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-ipbased.html)
8. [Multivalue Answer Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-multivalue.html)
9. [Weighted Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-weighted.html)
10. [Sử dụng CLI để cập nhật DNS Record trong Route 53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/route-53_example_route-53_ChangeResourceRecordSets_section.html)

Qua hai bài vừa rồi, hy vọng các bạn đã nắm được các khái niệm cơ bản trong Route 53. Đây cũng là những kiến thức hay gặp trong các kỳ thi chứng chỉ AWS. Tiếp theo, hãy nghiên cứu CloudFront, dịch vụ CDN của AWS.
