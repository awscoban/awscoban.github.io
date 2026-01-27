---
layout: post
title: "30. Elastic Load Balancer"
title2: "Elastic Load Balancer"
date: 2026-02-05
permalink: /2026/02/05/load-balancer
categories: [High Availability]
tags: [High Availability]
img: /assets/30_load_balancer/elb.png
summary: "Trong các hệ thống lớn với lượng truy cập cao, một bài toán cần giải quyết là phân phối tải giữa các máy chủ, gọi là Load Balancing, rất quan trọng để đảm bảo tính sẵn sàng cao (High Availability). Trên AWS, dịch vụ giúp cân bằng tải là Elastic Load Balancer (ELB), với 4 loại chính sẽ được đề cập dưới đây."
---

Trong các hệ thống lớn với lượng truy cập cao, một bài toán cần giải quyết là phân phối tải giữa các máy chủ, gọi là *Load Balancing*, rất quan trọng để đảm bảo tính sẵn sàng cao (*High Availability*). Trên AWS, dịch vụ giúp cân bằng tải là **Elastic Load Balancer (ELB)**, với 4 loại chính sẽ được đề cập dưới đây.


## Trong bài này:

- [1. Tổng quan](#overview)
    - [Cross-Zone Load Balancing](#cross-zone-load-balancing)
- [2. Application Load Balancer (ALB)](#alb)
- [3. Network Load Balancer (NLB)](#nlb)
- [4. Gateway Load Balancer (GLB)](#glb)
- [Tài liệu tham khảo](#reference)


<a name="overview"></a>

## 1. Tổng quan

Trên AWS, Elastic Load Balancer (ELB) tự động phân phối lưu lượng mạng đến nhiều tài nguyên đích (gọi là **target**) như [EC2 Instance](/2025/12/16/ec2-fundamental), container, hoặc các địa chỉ IP của máy chủ, trong một hoặc nhiều AZ. ELB cũng tự động theo dõi tình trạng của các tài nguyên đích, đảm bảo chỉ phân phối lưu lượng đến các tài nguyên đang hoạt động (*healthy target*). 
<!-- TODO: include link to  ECS -->


<p>
<image src="/assets/30_load_balancer/elb.png" alt="Elastic Load Balancer" style="max-width:70%;height:auto;display:block;margin:0 auto;"/>
</p>


Hình trên minh hoạ cấu trúc tổng quan của ELB. 
Một ELB thường được triển khai trên nhiều AZ, điều phối lưu lượng đến các tài nguyên đích đã đăng ký (*registered target*).
ELB tạo một **LB Node** trong mỗi AZ. Các Node này sẽ phân phối lưu lượng đến các tài nguyên đích trong AZ tương ứng.

Khi client gửi một yêu cầu đến ELB, Domain Name của ELB sẽ được phân giải thành địa chỉ IP của các LB Node. Client tự quyết định gửi yêu cầu đến LB Node nào. LB Node tiếp nhận yêu cầu sẽ chuyển tiếp đến một healthy target do nó quản lý. Mỗi loại ELB sử dụng các thuật toán phân phối tải khác nhau.

- [Application Load Balancer](#alb) sử dụng thuật toán [Round Robin](https://en.wikipedia.org/wiki/Round-robin_DNS) để phân phối tải. Đây là thuật toán đơn giản nhất, lần lượt xoay vòng từng healthy target.
- [Network Load Balancer](#nlb) và [Gateway Load Balancer](#glb) sử dụng thuật toán [flow hash](https://www.linkedin.com/pulse/hash-flow-algorithm-aws-network-load-balancer-nlb-in-depth-mishra/), tính một giá trị băm (hash) dựa trên thông tin của kết nối để xác định healthy target đích.


Tài nguyên đích của ELB thường gặp nhất là EC2 Instance, đặc biệt là thông qua các [Auto Scaling Group (ASG)](/2026/02/02/ec2-autoscaling). Khi đăng ký ASG cho ELB, ELB sẽ tự động đăng ký các Instance trong ASG vào danh sách tài nguyên đích, và cập nhật khi ASG [mở rộng](/2026/02/02/ec2-autoscaling#scale-out) hay [thu hẹp](/2026/02/02/ec2-autoscaling#scale-in). 


Về kết nối, AWS hỗ trợ hai chế độ:

- **Internet-facing**: ELB có public IP, có thể truy cập từ Internet. Có thể phân phối tải đến tài nguyên nằm trong **cả public và private** [subnet](/2025/11/13/vpc#vpc-subnet).
- **Internal**: ELB chỉ có private IP, chỉ truy cập được từ trong VPC hoặc qua VPN/Direct Connect từ on-premise. Chỉ phân phối tải đến tài nguyên nằm trong **private subnet**.
<!-- TODO: include link to VPN, Direct Connect -->

Cả hai loại ELB này đều kết nối với tài nguyên đích bằng private IP.




<a name="cross-zone-load-balancing"></a>

### Cross-Zone Load Balancing

Không phải khi tạo ELB trên nhiều AZ là đã cân bằng tải xuyên AZ rồi sao? Thực ra **cross-zone load balancing** là một tính năng khác, mình sẽ giải thích kỹ hơn dưới đây.


<p>
<image src="/assets/30_load_balancer/elb-cross-zone.png" alt="Cross-Zone Load Balancing" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>


Giả sử ta có một ELB trên 2 AZ, tức có 2 LB Node. Tài nguyên đích ở AZ 1 là 2 EC2 Instance, ở AZ 2 là 4 EC2 Instance, như hình trên. 

- Nếu cross-zone load balancing **tắt**, ELB phân phối tải **đều tới 2 LB Node**. Mỗi Node nhận 50% lưu lượng, rồi lại chia đều cho các EC2 Instance nó quản lý. Kết quả, EC2 Instance ở AZ 1 nhận nhiều tải hơn (25% mỗi Instance) so với AZ 2 (12.5% mỗi Instance).
- Nếu cross-zone load balancing **bật**, ELB phân phối tải **đều tới tất cả 6 EC2 Instance**. Mỗi Instance nhận 16.67% lưu lượng.


Như vậy, **cross-zone load balancing** giúp phân phối tải đồng đều giữa tài nguyên trong các AZ khác nhau, nên dùng khi số lượng tài nguyên ở các AZ không đồng đều.





<a name="alb"></a>

## 2. Application Load Balancer (ALB)


AWS hỗ trợ 3 loại ELB chính: Application Load Balancer (ALB), [Network Load Balancer (NLB)](#nlb), và [Gateway Load Balancer (GLB)](#glb). Ngoài ra còn có Classic Load Balancer (CLB) là phiên bản cũ, ít được dùng hiện nay. Phần còn lại của bài viết sẽ trình bày những điểm mấu chốt của 3 loại ELB, còn CLB không được khuyến khích dùng, bạn đọc quan tâm có thể tìm hiểu thêm [tại đây](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/introduction.html).


Với ALB, như tên gọi, nó hoạt động ở **tầng ứng dụng** (Layer 7) trong mô hình OSI. Cụ thể, ALB chỉ phân phối tải của giao thức HTTP/HTTPS, các giao thức tầng dưới như TCP/UDP không được hỗ trợ.


<p>
<image src="/assets/30_load_balancer/alb.png" alt="Application Load Balancer" style="max-width:60%;height:auto;display:block;margin:0 auto;"/>
</p>

**Các thành phần chính:**

- **Listener**: thành phần này "lắng nghe" các yêu cầu từ client trên một cổng (port, từ 1-65535) và giao thức đã cấu hình (HTTP/HTTPS). Mỗi ALB cần **ít nhất một** Listener. Cần có SSL Certificate cho HTTPS.
- **Target Group**: là nhóm các tài nguyên đích nhận lưu lượng từ ALB khi thoả mãn các điều kiện trong Rule (trình bày ngay dưới đây). Mỗi ALB có thể có **nhiều** Target Group, nhưng một Target Group chỉ thuộc về một ALB. 
- **Rule**: tập hợp các quy tắc trên Listener để xác định phân phối tải tới Target Group nào. Một Listener có thể có nhiều Rule, đánh giá theo mức độ ưu tiên. Khi tạo một rule, người dùng chỉ định điều kiện và Target Group tương ứng. 


Ví dụ, lệnh AWS CLI dưới đây tạo một Listener trên ALB tên `awscoban-alb`, nghe cổng 80 với giao thức HTTP, với hành động mặc định (nếu không đặt thêm Rule nào khác) là chuyển tiếp tất cả lưu lượng đến Target Group `awscoban-alb-tg`.

```sh
aws elbv2 create-listener \
    --load-balancer-arn <awscoban-alb_ARN> \
    --protocol HTTP \
    --port 80 \
    --default-actions "Type=forward,TargetGroupArn=<awscoban-alb-tg_ARN>"
```

Lệnh dưới đây tạo một Rule trong Listener đó, với điều kiện nếu `host-header` (một trường trong giao thức HTTP, chỉ tên miền của máy chủ người dùng đang muốn truy cập) là `awscoban.com` hoặc `www.awscoban.com`, thì chuyển tiếp yêu cầu đến Target Group `awscoban-alb-tg`.

```sh
aws elbv2 create-rule \
    --listener-arn <awscoban-alb_ARN> \
    --priority 10 \
    --conditions "Field=host-header,Values=awscoban.com,www.awscoban.com" \
    --actions "Type=forward,TargetGroupArn=<awscoban-alb-tg_ARN>"
```

Một tham số cần lưu ý là **priority** (độ ưu tiên). Nếu có nhiều Rule thoả mãn điều kiện, Rule có giá trị priority **nhỏ hơn** sẽ được ưu tiên.

Chi tiết các loại `actions` và `conditions` có thể tham khảo trong [tài liệu AWS](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-rules.html). Rõ ràng các loại điều kiện như `host-header`, `http-header`, `query-string`, v.v. là ở tầng ứng dụng (Layer 7), nên loại ELB này được gọi là Application Load Balancer.




<a name="nlb"></a>

## 3. Network Load Balancer (NLB)

Khác với ALB, NLB hoạt động ở **tầng vận chuyển (transport layer)** (Layer 4) trong mô hình OSI, hỗ trợ các giao thức [TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol), [UDP](https://en.wikipedia.org/wiki/User_Datagram_Protocol), hay [QUIC](https://en.wikipedia.org/wiki/QUIC). 

NLB cũng có các thành phần chính tương tự ALB: Listener, Target Group, và Rule. Tuy nhiên, do hoạt động ở tầng thấp hơn, NLB không thể phân tích nội dung gói tin như ALB, nên không thể sử dụng các điều kiện phức tạp trong Rule để định tuyến lưu lượng. Thường NLB chỉ đơn giản chuyển tiếp yêu cầu đến Target Group phù hợp.

Ví dụ, dưới đây tạo một Listener trên NLB `awscoban-nlb`, lắng nghe trên cổng 443 với giao thức TCP, chuyển tiếp lưu lượng đến Target Group `awscoban-nlb-tg`.

```sh
aws elbv2 create-listener \
    --load-balancer-arn <awscoban-nlb_ARN> \
    --protocol TCP \
    --port 443 \
    --default-actions "Type=forward,TargetGroupArn=<awscoban-nlb-tg_ARN>"
``` 

Có thể thấy lệnh này rất giống với lệnh tạo Listener cho ALB, chỉ khác ở giao thức (TCP thay vì HTTP) và cổng (443 thay vì 80). Đây cũng là đặc điểm chính để phân biệt hai loại ELB này.

**Các đặc điểm nổi bật của NLB:**

- Tuy không thể phân tích nội dung gói tin, một ưu điểm lớn bù lại là NLB có tốc độ nhanh hơn ALB rất nhiều, chính vì xử lý ít phức tạp hơn. NLB có thể xử lý hàng triệu yêu cầu mỗi giây với độ trễ thấp hơn ALB khoảng 4 lần, thích hợp cho các ứng dụng yêu cầu tốc độ cao.
- NLB hỗ trợ **IP tĩnh (Elastic IP)**, phù hợp cho các ứng dụng cần địa chỉ IP cố định.


Tóm lại, sử dụng NLB nếu ứng dụng:
- Yêu cầu tốc độ cao, độ trễ thấp
- Sử dụng các giao thức không phải HTTP/HTTPS
- Cần địa chỉ IP tĩnh

Còn lại, nên dùng ALB để tận dụng các tính năng định tuyến nâng cao ở tầng ứng dụng.







<a name="glb"></a>

## 4. Gateway Load Balancer (GLB)


Nếu ứng dụng cần tích hợp các chức năng bảo mật được cung cấp bởi bên thứ ba, như chống DDoS, botnet, v.v., thì GLB là lựa chọn phù hợp.

<p>
<image src="/assets/30_load_balancer/glb.png" alt="Gateway Load Balancer" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>

Hình trên mô tả luồng yêu cầu từ client đến máy chủ dịch vụ khi tích hợp thêm GLB cho một hệ thống bảo mật bên thứ ba. Cụ thể, VPC bên trái (Application VPC) là của ứng dụng, VPC bên phải là của nhà cung cấp dịch vụ bảo mật (Security VPC). GLB được đặt trong Security VPC, cân bằng tải cho Target Group gồm các Instance chạy dịch vụ bảo mật. Ở Application VPC, ta tạo một **Gateway Load Balancer Endpoint (GLB Endpoint)**, để kết nối với GLB. Luồng yêu cầu như sau:

1. Client gửi yêu cầu đến ứng dụng. GLB Endpoint tiếp nhận yêu cầu này.
2. GLB Endpoint chuyển tiếp yêu cầu đến GLB trong Security VPC.
3. GLB phân phối yêu cầu đến các Instance trong Target Group (chạy dịch vụ bảo mật).
4. Sau khi xử lý, dịch vụ bảo mật trả kết quả (ví dụ, kết nối này có khả nghi không) về GLB.
5. GLB chuyển tiếp kết quả về GLB Endpoint.
6. GLB Endpoint chuyển kết quả cho máy chủ ứng dụng.

Luồng phản hồi từ máy chủ ứng dụng về client diễn ra tương tự, nhưng ngược chiều.




<a name = "reference"></a>

## Tài liệu tham khảo

1. [ELB Hoạt động như thế nào?](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html)
2. [Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)
3. [Listener & Rule cho ALB](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-rules.html)
4. [Network Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html)
5. [Listener cho NLB](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-listeners.html)
6. [Gateway Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/getting-started.html)
7. [Classic Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/introduction.html)


Trên đây là những điểm cốt lõi về Elastic Load Balancer trên AWS. Mình hy vọng bài viết này có ích. Tiếp theo, nhân việc đã nói về GLB Endpoint trong VPC, hãy quay lại chủ đề VPC để tìm hiểu các khái niệm nâng cao, trong đó có VPC Endpoint.