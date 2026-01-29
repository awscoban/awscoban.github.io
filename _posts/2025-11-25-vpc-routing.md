---
layout: post
title: "6. Định tuyến trong VPC"
title2: "Định tuyến trong VPC"
date: 2025-11-25
permalink: /2025/11/25/vpc-routing
categories: [VPC, Networking]
tags: [VPC, Networking]
img: /assets/6_vpc_routing/natgw.png
summary: "Ở hai bài trước, mình đã trình bày về kiến trúc và cách thiết kế VPC nếu cần triển khai ứng dụng trên AWS.
Một ứng dụng sẽ có nhiều thành phần (giao diện, tính toán, lưu trữ), ở cả public và private subnet. Vậy câu hỏi đặt ra tiếp theo là chúng giao tiếp với nhau và với Internet như thế nào?"
---

Ở hai bài trước, mình đã trình bày về kiến trúc và cách thiết kế VPC nếu cần triển khai ứng dụng trên AWS.
Một ứng dụng sẽ có nhiều thành phần (giao diện, tính toán, lưu trữ), ở cả public và private subnet. Vậy câu hỏi đặt ra tiếp theo là chúng giao tiếp với nhau và với Internet như thế nào?


## Trong bài này:

- [1. Bộ định tuyến VPC](#vpc-router)
- [2. Internet Gateway](#igw)
- [3. NAT Gateway](#natgw)
- [4. Với IPv6](#for-ipv6)
- [Tài liệu tham khảo](#reference)


<a name ="vpc-router"></a>

## 1. Bộ định tuyến VPC

Trong mỗi VPC (cả VPC mặc định hay tự tạo), AWS cung cấp sẵn một bộ định tuyến (*VPC Router*) để điều hướng các gói tin giữa các subnet trong VPC cũng như với Internet bên ngoài.
Bộ định tuyến này có *network interface* tại tất cả subnet, được dành riêng địa chỉ IP thứ 2: `xxx.xxx.xxx.1` (xem lại [5 IP không thể sử dụng trong subnet](/2025/11/13/vpc/#vpc-subnet)), và AWS đảm bảo nó luôn hoạt động, miễn là ta cấu hình bảng định tuyến đúng.

<a name ="route-table"></a>

Bộ định tuyến được điều khiển bởi __bảng định tuyến (*route table*)__ có trong mỗi subnet. Bảng này quyết định một gói tin xuất phát từ một IP trong subnet sẽ được gửi đi đâu.
Mặc định, khi tạo VPC, AWS sẽ tạo sẵn một bảng định tuyến (gọi là *Main route table*) và gán cho tất cả subnet. 
Nếu cần tự cấu hình theo ý muốn, ta có thể tự tạo bảng định tuyến, gỡ bảng định tuyến có sẵn khỏi subnet, rồi gán bảng mới vào.
Lưu ý, **một subnet chỉ có thể có một bảng định tuyến** tại bất kì thời điểm nào, nhưng một bảng định tuyến có thể được gán cho nhiều subnet. 

Mỗi bảng định tuyến chứa tập hợp các tuyến (*route*), quy định cách điều hướng các gói tin. Ví dụ, xét bảng định tuyến có sẵn dưới đây của [VPC mặc định](/2025/11/13/vpc/#default-vpc) (có CIDR `172.31.0.0/16`):

<p>
<image src="/assets/6_vpc_routing/sample_route_table.png" alt="Route Table" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>

Trong đó hai trường quan trọng là:


#### Destination
Là dải CIDR ứng với địa chỉ đích của gói tin. Bộ định tuyến sẽ xác định địa chỉ đích khớp với route nào. Ví dụ, hình trên có 2 route với destination là `0.0.0.0/0` (CIDR này chứa tất cả địa chỉ IPv4 có thể có, xem lại [cách đọc CDIR](/2025/11/13/vpc/#cidr)), và `172.31.0.0/16` là CIDR của VPC.

Giả sử một gói tin có địa chỉ đích là `172.31.12.34`, địa chỉ đích này khớp với cả 2 destination, vì IP này nằm trong cả 2 dải CIDR. Khi có nhiều destination trùng khớp, **nguyên tắc xác định route cho gói tin là chọn route cụ thể nhất (tức có prefix lớn nhất)**. Như vậy, gói tin `172.31.12.34` cuối cùng sẽ được khớp với `172.31.0.0/16`, có đích đến là `local`, tức gói tin sẽ được chuyển tiếp đến máy có IP này trong VPC.

Một ví dụ khác, gói tin có địa chỉ đích là `56.78.90.12` (một địa chỉ IP công cộng trên Internet) sẽ chỉ khớp với destination `0.0.0.0/0`. Destination này được dùng để chỉ tất cả kết nối, và có độ ưu tiên thấp nhất. 

#### Target
Là nơi gói tin sẽ được chuyển tiếp đến, ứng với destination cụ thể. Trong hình trên, các gói tin khớp với destination là `0.0.0.0/0` (IPv4 bất kỳ) và không nằm trong CIDR `172.31.0.0/16` sẽ được chuyển đến Internet Gateway. Route thứ hai có target là `local`, chỉ các IP trong cùng VPC, định tuyến giữa các máy trong nội bộ VPC.

<!-- Cột *Propagated* chỉ các route được tự động thêm vào khi kết nối VPN từ VPC đến on-premise network, với các route thông thường sẽ có giá trị `No`, ta không cần quan tâm nhiều. -->


<a name ="igw"></a>

## 2. Internet Gateway

Internet Gateway quản lý giao tiếp giữa VPC với [các dịch vụ AWS nằm trong Public Network](/2025/11/13/vpc#public-network) và mạng Internet. Nếu VPC không có Internet Gateway, các tài nguyên trong VPC không thể kết nối ra bên ngoài, và ngược lại, từ AWS Public Network hay Internet cũng không thể truy cập vào VPC. 

Một VPC có tối đa 1 Internet Gateway (có thể không có). Internet Gateway có khả năng phục hồi [Region Resilience](/2025/11/12/aws_infrastructure/#resilience), và cũng như bộ định tuyến VPC, được AWS quản lý, người dùng chỉ cần sử dụng.


Trong hình dưới, ta có một VPC với CIDR `10.0.0.0/16`, trong VPC có một [public subnet](/2025/11/13/vpc#vpc-subnet) với CIDR `10.0.16.0/20`, tự động gán public IP cho tài nguyên bên trong. 
Internet Gateway được tạo và gán cho VPC.
Sau đó, ta tạo một bảng định tuyến như sau:

```
Destination     Target
0.0.0.0/0       IGW
10.0.0.0/16     local
```

Khi đó các tài nguyên trong subnet này có thể kết nối Internet và được truy cập từ Internet.


<p>
<image src="/assets/6_vpc_routing/igw.png" alt="Internet Gateway" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>


Giả sử trong subnet có một EC2 instance với có private IP `10.0.16.123`, và public IP `98.76.54.32`. 
EC2 instance này gửi một gói tin đến máy chủ có IP `1.1.1.1` trên Internet.
Đối chiếu với bảng định tuyến, IP máy chủ này khớp với destination `0.0.0.0/0`, nên gói tin sẽ được chuyển tiếp đến Internet Gateway.

Internet Gateway quản lý một bản ghi, ánh xạ private IP với public IP của EC2 instance này. Khi gói tin được chuyển tới đây, `Source` (nơi gửi) của gói tin sẽ được thay thế từ private IP sang public IP của EC2 instance, rồi được chuyển tiếp đến máy chủ `1.1.1.1` qua Internet. Thao tác này giúp che giấu private IP của EC2 instance.

Ở chiều ngược lại, khi máy chủ gửi lại một gói tin, `Destination` (đích đến) của gói tin này sẽ là public IP của EC2 instance. Địa chỉ IP này trên Internet thực tế thuộc về Internet Gateway, nên gói tin sẽ được chuyển đến đây qua Internet. Tại đây, Internet Gateway lại dịch ngược public IP sang private IP của EC2 instance, rồi chuyển gói tin đến EC2 instance qua private IP này.


<a name ="natgw"></a>


## 3. NAT Gateway

Internet Gateway chỉ giúp các tài nguyên trong **public** subnet kết nối với bên ngoài VPC, do chúng được cấp phát public IP. 
Các tài nguyên trong **private** subnet nếu không có cấu hình gì khác sẽ chỉ có thể giao tiếp nội bộ trong VPC. 
Cơ chế này trong phần lớn trường hợp sẽ giúp đáp ứng yêu cầu bảo mật, do bên ngoài cũng không thể kết nối đến các tài nguyên trong private subnet.

Tuy nhiên, ở một số tình huống cần kết nối với bên ngoài, ví dụ như một EC2 instance cần tải xuống một bản vá lỗ hổng hay một phần mềm cần thiết nào đó, cấu hình mặc định trên sẽ không thể đáp ứng. 
Để giải quyết vấn đề này, có thể dùng NAT Gateway.

Một cách tổng quát, NAT (Network Address Translation) là một cơ chế cho phép nhiều thiết bị trong private network chia sẻ một public IP để có thể truy cập Internet. 
Trong AWS, các dịch vụ thực thi NAT là NAT Gateway và NAT Instance. AWS khuyến khích sử dụng NAT Gateway do có HA cao hơn và người dùng không cần quản lý như NAT Instance. Bạn đọc có thể xem thêm [so sánh hai dịch vụ này tại đây](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-comparison.html)

Như vậy, khi sử dụng NAT Gateway, các tài nguyên trong private subnet đã có public IP (của NAT Gateway), do đó có thể kết nối Internet qua Internet Gateway!

NAT Gateway được đặt trong **public** subnet, có địa chỉ IPv4 tĩnh (AWS gọi là *Elastic IP*), có khả năng phục hồi [AZ Resilience](/2025/11/12/aws_infrastructure/#resilience). Nếu cần [Region Resilience](/2025/11/12/aws_infrastructure/#resilience), ta cần đặt một NAT Gateway trong mỗi AZ của Region.

<p>
<image src="/assets/6_vpc_routing/natgw.png" alt="NAT Gateway" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>

Hình trên mô tả vị trí và quá trình dịch địa chỉ với NAT Gateway. Trong private subnet APP, giả sử ta có một EC2 instance với private IP `10.0.32.123` muốn kết nối với máy chủ `1.1.1.1` ngoài Internet để tải một bản cập nhật phần mềm, và một bảng định tuyến như sau:

```
Destination     Target
0.0.0.0/0       NATGW
10.0.0.0/16     local
```

Khi đó, gói tin với destination `1.1.1.1` sẽ khớp với route đầu tiên, và được chuyển tiếp đến NAT Gateway có IP tĩnh `56.78.90.12` ở public subnet WEB. Bảng định tuyến của subnet này như sau:

```
Destination     Target
0.0.0.0/0       IGW
10.0.0.0/16     local
```

NAT Gateway sẽ lưu lại `Source` của gói tin, tức private IP `10.0.32.123` EC2 instance, rồi thay đổi `Source` thành IP tĩnh của nó. Sau đó, đối chiếu với bảng định tuyến, gói tin sẽ được chuyển tiếp tới Internet Gateway. 
Internet Gateway không cần ánh xạ private-public IP như phần trên, do NAT Gateway đã có sẵn IP tĩnh (là public IP). Gói tin được chuyển tới đích đến cuối là máy chủ trên Internet. 

Quá trình gửi gói tin phản hồi diễn ra ngược lại, với `Destination` của gói tin này là `56.78.90.12` (tại máy chủ), `56.78.90.12` (tại Internet Gateway), và `10.0.32.123` (tại NAT Gateway, nơi dịch ngược lại private IP của EC2 instance).


Một điểm quan trọng cần lưu ý là NAT (cả Gateway và Instance) chỉ cho phép **kết nối chiều đi** (*outbound connection*). Nghĩa là kết nối phải xuất phát từ private subnet ra bên ngoài, rồi nhận lại phản hồi. NAT **không cho phép kết nối chiều ngược lại**, tức một thiết bị bên ngoài muốn **chủ động** kết nối với private subnet sẽ bị chặn lại. Cơ chế này vừa đủ để giải quyết vấn đề cần tải xuống tài nguyên trên Internet nêu trên, mà vẫn đảm bảo tương đối về bảo mật. Trong hình trên, mình không vẽ đường về của gói tin phản hồi (dù có tồn tại) để nhấn mạnh điều này.


## 4. Với IPv6

- NAT **chỉ dành cho IPv4**. Vì với IPv6, không có khái niệm public và private IP, do không gian địa chỉ IPv6 (128 bit) quá lớn nên không xảy ra vấn đề cạn kiệt địa chỉ IP, không cần thiết phải chia ra public và private như IPv4 (bạn đọc có thể xem lại [bài trước](/2025/11/20/design-vpc#ip-design) để hiểu rõ hơn). Tất cả các địa chỉ IPv6 đều có thể định tuyến công khai (*publicly routable*).

- Dó đó, NAT Gateway không dùng cho IPv6. Ngược lại, Internet Gateway có thể làm việc với tất cả IPv6. Trong bảng định tuyến, có thể thêm route với destination `::/0` với target Internet Gateway để kết nối Internet **hai chiều**. CIDR `::/0` này chỉ tất cả địa chỉ IPv6, tương tự như `0.0.0.0/0` trong IPv4.

- Còn nếu chỉ cần kết nối chiều đi (tương tự như trường hợp private subnet và NAT Gateway trong IPv4), hãy dùng route `::/0` trên với target là [**Egress-Only Internet Gateway**](/2026/02/08/vpc-advanced#egress-only-internet-gateway). Đây là dịch vụ dành riêng cho IPv6.



<a name = "reference"></a>

## Tài liệu tham khảo

1. [VPC Route Table](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)
2. [Internet Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)
3. [NAT Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
4. [NAT Gateway vs NAT Instance](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-comparison.html)
5. [Egress-Only Internet Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/egress-only-internet-gateway.html)


Hy vọng qua bài này, các bạn đã nắm được cách AWS định tuyến trong VPC. Tiếp theo ta sẽ xem xét vấn đề bảo mật trong VPC, với Security Group và Network Access Control List (NACL).
