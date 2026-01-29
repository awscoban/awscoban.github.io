---
layout: post
title: "31. VPC Nâng cao"
title2: "VPC Nâng cao"
date: 2026-02-08 
permalink: /2026/02/08/vpc-advanced
categories: [VPC, Networking]
tags: [VPC, Networking]
img: /assets/31_vpc_advanced/private-link.png
summary: "VPC là một mạng ảo cô lập, không kết nối với bên ngoài, trừ khi được cấu hình thêm. Ngoài Internet Gateway và NAT Gateway, AWS cung cấp thêm một giải pháp nữa, đó là PrivateLink với các VPC Endpoint, sẽ được thảo luận dưới đây, cùng với VPC Peering, Transit Gateway để kết nối nhiều VPC với nhau; Egress-Only Internet Gateway cho IPv6; và VPC Flow Log để quản lý và theo dõi."
---

VPC là một mạng ảo cô lập, không kết nối với bên ngoài, trừ khi được cấu hình thêm. Ngoài Internet Gateway và NAT Gateway, AWS cung cấp thêm một giải pháp nữa, đó là PrivateLink với các VPC Endpoint, sẽ được thảo luận dưới đây, cùng với VPC Peering, Transit Gateway để kết nối nhiều VPC với nhau; Egress-Only Internet Gateway cho IPv6; và VPC Flow Log để quản lý và theo dõi.


## Trong bài này:

- [1. AWS PrivateLink](#private-link)
    - [VPC Interface Endpoint](#interface-endpoint)
- [2. VPC Gateway Endpoint](#gateway-endpoint)
- [3. VPC Peering](#vpc-peering)
- [4. Transit Gateway](#transit-gateway)
- [5. Egress-Only Internet Gateway cho IPv6](#egress-only-internet-gateway)
- [6. VPC Flow Log](#vpc-flow-log)
- [Tài liệu tham khảo](#reference)




<a name="private-link"></a>

## 1. AWS PrivateLink

Trong bài [Định tuyến trong VPC](/2025/11/25/vpc-routing), ta đã biết để kết nối dịch vụ từ trong VPC với bên ngoài, cần [Internet Gateway](/2025/11/25/vpc-routing#igw) và cả [NAT Gateway](/2025/11/25/vpc-routing#natgw) cho private subnet. Nhưng hai lựa chọn này đều sẽ thiết lập kết nối qua mạng Internet. Nếu muốn bảo mật hơn nữa, chỉ thông qua mạng riêng của AWS, có thể cân nhắc sử dụng PrivateLink.

<p>
<image src="/assets/31_vpc_advanced/private-link.png" alt="Private Link" style="max-width:80%;height:auto;display:block;margin:0 auto;"/>
</p>


PrivateLink cho phép kết nối từ trong VPC (cả public và private subnet) thông qua 4 loại VPC Endpoint sau:

- [VPC Interface Endpoint](#interface-endpoint): kết nối đến các dịch vụ AWS, dịch vụ chạy trong VPC khác (cùng hoặc khác tài khoản AWS), dịch vụ SaaS (*Software as a Service*) của bên thứ ba trên AWS Marketplace.
- [Gateway Load Balancer](/2026/02/05/load-balancer#glb): kết nối đến các dịch vụ bảo mật mạng, đã trình bày trong bài trước.
- [VPC Resource Endpoint](https://docs.aws.amazon.com/vpc/latest/privatelink/privatelink-access-resources.html): kết nối đến các tài nguyên trong VPC khác hoặc on-premise (như CSDL, EC2 Instance, DNS, địa chỉ IP, v.v.).
- [VPC Service Network Endpoint](https://docs.aws.amazon.com/vpc/latest/privatelink/privatelink-access-service-networks.html): tích hợp với [AWS Lattice](https://docs.aws.amazon.com/vpc-lattice/latest/ug/what-is-vpc-lattice.html) để kết nối và quản lý tài nguyên trong nhiều VPC.


Trong đó, hay sử dụng nhất là Interface Endpoint, sẽ được trình bày chi tiết dưới đây.


<a name="interface-endpoint"></a>

### VPC Interface Endpoint

<p>
<image src="/assets/31_vpc_advanced/interface-endpoint.png" alt="Interface Endpoint" style="max-width:80%;height:auto;display:block;margin:0 auto;"/>
</p>

Có thể dùng Interface Endpoint để kết nối đến [hầu hết các dịch vụ AWS](https://docs.aws.amazon.com/vpc/latest/privatelink/aws-services-privatelink-support.html). Về bản chất, khi tạo Interface Endpoint cho một dịch vụ, AWS sẽ tạo một [ENI](/2025/12/16/ec2-fundamental#network-interface) trong subnet, và một DNS cho dịch vụ đó để sử dụng. DNS này được phân giải thành private IP của ENI trong subnet. Và như vậy, ta có thể truy cập dịch vụ đó ngay trong VPC, vì [các thành phần trong VPC (với private IP trong CIDR của VPC) có thể truy cập lẫn nhau](/2025/11/13/vpc#private-network). 


Ví dụ, giả sử ta tạo một Interface Endpoint cho [CloudWatch](/2026/01/05/cloudwatch), DNS của CloudWatch qua Interface Endpoint sẽ có dạng:

```
vpce-svc-0fc975f3e7e5beba4.monitoring.us-east-1.vpce.amazonaws.com
```

Và nếu ứng dụng của ta chạy trong VPC đó, có thể truy cập CloudWatch thông qua DNS này. 


<a name="gateway-endpoint"></a>

## 2. VPC Gateway Endpoint

Gateway Endpoint ra mắt trước Private Link, không dùng ENI và DNS như Private Link, mà hoạt động như một **đường hầm** nối từ VPC đến dịch vụ đích, sử dụng [bảng định tuyến](/2025/11/25/vpc-routing#route-table) để kết nối ra bên ngoài VPC. Gateway Endpoint **chỉ hỗ trợ kết nối đến [S3](/2025/11/30/s3-fundamental) và DynamoDB**.
<!-- TODO: include link to DynamoDB -->

<p>
<image src="/assets/31_vpc_advanced/gateway-endpoint.png" alt="Gateway Endpoint" style="max-width:80%;height:auto;display:block;margin:0 auto;"/>
</p>

Khi tạo Gateway Endpoint cho một subnet, AWS tự động thêm một mục định tuyến trong bảng định tuyến của subnet đó:

```
Destination	        Target
prefix_list_id	        gateway_endpoint_id
```

Diễn giải (xem lại bài [Định tuyến trong VPC](/2025/11/25/vpc-routing#route-table) nếu cần): 
- [Prefix List](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-aws-managed-prefix-lists.html): là danh sách các dải IP đại diện cho một dịch vụ AWS tại một Region. Ở đây `Destination` là `prefix_list_id`, tức chỉ tất cả kết nối đến S3 hoặc DynamoDB.
- `Target`: là Gateway Endpoint đã tạo, và vì Gateway Endpoint này kết nối đến dịch vụ đích, tất cả traffic đến `prefix_list_id` sẽ được chuyển đến nó, rồi đến dịch vụ đích.


S3 và DynamoDB đều có thể kết nối từ trong VPC sử dụng **cả [Interface Endpoint](#interface-endpoint) và Gateway Endpoint**. Việc chọn phương án nào tùy thuộc vào nhu cầu cụ thể:


|  | Gateway Endpoint | Interface Endpoint |
|:---|:---|:---|
| Cơ chế | Định tuyến tới S3/DynamoDB. | Dùng ENI để tạo private IP cho S3/DynamoDB trong VPC để truy cập |
| Truy cập từ mạng doanh nghiệp (on‑premise) | Không cho phép | Cho phép |
| Truy cập từ Region khác | Không cho phép | Cho phép, nếu 2 VPC đã kết nối qua [VPC Peering](#vpc-peering) hoặc [Transit Gateway](#transit-gateway) |
| Chi phí | Miễn phí | Tính phí |





<a name="vpc-peering"></a>

## 3. VPC Peering

Do VPC là mạng ảo cô lập, mặc định không thể kết nối với VPC khác để chia sẻ tài nguyên. Một cách giải quyết vấn đề này là sử dụng VPC Peering.

<p>
<image src="/assets/31_vpc_advanced/vpc-peering.png" alt="VPC Peering" style="max-width:70%;height:auto;display:block;margin:0 auto;"/>
</p>

VPC Peering cho phép định tuyến giữa hai VPC (cùng hoặc khác tài khoản AWS) thông qua bảng định tuyến. Sau khi thiết lập Peering, chỉ cần thêm vào bảng định tuyến của các subnet trong **cả hai VPC**. 

Giả sử ta có VPC A và VPC B, với CIDR lần lượt là `10.0.0.1/16` và `172.31.0.0/16`, cùng một Peering Connection `pcx-11112222` kết nối chúng. Ta cần cập nhật các bảng định tuyến như sau:

| Bảng định tuyến | Destination | Target |
|:---|:---|:---|
| VPC A | `10.0.0.1/16` (CIDR của VPC A) | Local |
| VPC A | `172.31.0.0/16` (CIDR của VPC B) | `pcx-11112222` |
| VPC B | `172.31.0.0/16` (CIDR của VPC B) | Local |
| VPC B | `10.0.0.1/16` (CIDR của VPC A) | `pcx-11112222` |

Nghĩa là, tại VPC A, tất cả traffic đến CIDR của VPC B sẽ được chuyển đến Peering Connection `pcx-11112222`, rồi chuyển đến địa chỉ IP đích trong VPC B.


Lưu ý, VPC Peering **không hỗ trợ bắc cầu**, tức nếu VPC A kết nối với VPC B, VPC B kết nối với VPC C, nhưng A và C không kết nối với nhau thì cũng không thể truy cập lẫn nhau. 


<a name="transit-gateway"></a>

## 4. Transit Gateway

Rõ ràng, nếu cần kết nối nhiều VPC với nhau, VPC Peering không phải là lựa chọn tốt, vì cần thiết lập Peering Connection giữa từng cặp VPC, dẫn đến số lượng kết nối tăng theo cấp số nhân. Thay vào đó, có thể sử dụng Transit Gateway.

<p>
<image src="/assets/31_vpc_advanced/transit-gateway.png" alt="Transit Gateway" style="max-width:70%;height:auto;display:block;margin:0 auto;"/>
</p>

Ngoài VPC, Transit Gateway còn hỗ trợ cả VPN và Direct Connect, thậm chí cả Transit Gateway khác. Điều này giúp việc quản lý kết nối mạng đơn giản hơn rất nhiều.

Trên CLI, có thể tạo Transit Gateway bằng lệnh sau:

```sh
aws ec2 create-transit-gateway \ 
--description "AWS Co ban Transit Gateway" \
--region us-east-1
```

Sau đó, cần gán các VPC và subnet tương ứng vào Transit Gateway:

```sh
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-1234567890abcdef0 \
  --vpc-id vpc-1234567890abcdef0 \
  --subnet-ids subnet-1234567890abcdef0
```

Rồi cập nhật bảng định tuyến của các subnet trong các VPC để định tuyến qua Transit Gateway. Nếu `Destination` là CIDR của VPC khác, thì `Target` sẽ là Transit Gateway ID:

```sh
# Cho VPC A
aws ec2 create-route \
  --route-table-id rtb-1234567890abcdef0 \
  --destination-cidr-block 172.31.0.0/16 \
  --transit-gateway-id tgw-1234567890abcdef0     

# Cho VPC B
aws ec2 create-route \
  --route-table-id rtb-abcdef01234567890 \
  --destination-cidr-block 10.0.0.0/16 \
  --transit-gateway-id tgw-1234567890abcdef0
```


<a name="egress-only-internet-gateway"></a>

## 5. Egress-Only Internet Gateway cho IPv6

Với IPv4, có thể dùng [NAT Gateway](/2025/11/25/vpc-routing#natgw) để cho phép truy cập **một chiều** từ private subnet ra truy cập Internet. Với IPv6, chức năng này được thực hiện bởi Egress-Only Internet Gateway.

Tương tự như NAT Gateway, chỉ cần thêm một route trong bảng định tuyến của subnet, để chuyển tất cả traffic ra ngoài Internet (với destination `::/0`, tức tất cả địa chỉ IPv6) đến Egress-Only Internet Gateway:

```
Destination	        Target
::/0	                egress_only_igw_id
```

Egress-Only Internet Gateway lưu trạng thái kết nối, do đó các kết nối phản hồi từ Internet sẽ được tự động cho phép quay trở lại subnet. Các kết nối khởi tạo từ Internet đến subnet sẽ bị chặn lại.


<a name="vpc-flow-log"></a>

## 6. VPC Flow Log

Đây là dịch vụ ghi lại các thông tin bổ sung (*metadata*) về traffic vào và ra VPC, Subnet, hoặc ENI cụ thể trong VPC. Flow Log có thể được chuyển đến lưu trong [CloudWatch Logs](/2026/01/05/cloudwatch#cloudwatch-logs), [S3](/2025/11/30/s3-fundamental), hoặc [Kinesis Data Firehose](/2026/01/31/kinesis-glue#kinesis-data-firehose).


Một bản ghi Flow Log gồm các trường sau:

- `version`: phiên bản Flow Log.
- `account-id`: ID tài khoản AWS.
- `interface-id`: ID của ENI mà Flow Log theo dõi.
- **`srcaddr`**: địa chỉ IP nguồn.
- **`dstaddr`**: địa chỉ IP đích.
- **`srcport`**: cổng nguồn.
- **`dstport`**: cổng đích.
- **`protocol`**: giao thức (ví dụ, TCP = 6, UDP = 17). Giá trị của các giao thức được định nghĩa trong [IANA Protocol Number](https://www.iana.org/assignments/protocol-numbers/protocol-numbers.xhtml).
- `packets`: số gói tin trong kết nối.
- `bytes`: số byte trong kết nối.
- `start`: thời gian bắt đầu kết nối.
- `end`: thời gian kết thúc kết nối.
- **`action`**: hành động với kết nối (ACCEPT hoặc REJECT).
- `log-status`: trạng thái ghi log (OK, NODATA, SKIPDATA).

Ví dụ:

```
2 123456789010 eni-1235b8ca123456789 172.31.16.139 172.31.16.21 20641 22 6 20 4249 1418530010 1418530070 ACCEPT OK
```

Đây là kết nối SSH (`protocol = 6`, `dstport = 22`) từ địa chỉ IP nguồn `172.31.16.139` đến địa chỉ IP đích `172.31.16.21`. Kết nối này được chấp nhận.

Nếu bạn đang ôn thi chứng chỉ AWS, hãy ghi nhớ các trường quan trọng nhất trong Flow Log đã được in đậm ở trên: `srcaddr`, `dstaddr`, `srcport`, `dstport`, `protocol`, và `action`.


<a name = "reference"></a>

## Tài liệu tham khảo

1. [Private Link](https://docs.aws.amazon.com/vpc/latest/privatelink/concepts.html)
2. [Interface Endpoint](https://docs.aws.amazon.com/vpc/latest/privatelink/privatelink-access-aws-services.html)
3. [Gateway Endpoint](https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-endpoints.html)
4. [VPC Endpoint cho S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/privatelink-interface-endpoints.html#types-of-vpc-endpoints-for-s3)
5. [VPC Endpoint cho DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/privatelink-interface-endpoints.html#types-of-vpc-endpoints-for-ddb)
6. [VPC Peering](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html)
7. [Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/how-transit-gateways-work.html)
8. [Egress-Only Internet Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/egress-only-internet-gateway.html)
9. [Cú pháp của VPC Flow Log](https://docs.aws.amazon.com/vpc/latest/userguide/flow-log-records.html)

Bài viết này khá dài, cảm ơn các bạn đã đọc đến đây. Tiếp theo, hãy tìm hiểu về Route 53, dịch vụ DNS của AWS.