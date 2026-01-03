---
layout: post
title: "20. RDS Multi-AZ"
title2: "RDS Multi-AZ"
date: 2026-01-10
permalink: /2026/01/10/rds-multi-az
categories: [RDS, Database]
tags: [RDS, Database]
img: /assets/20_rds_multi_az/rds_multi_az-cluster.png
summary: "Bài này sẽ giới thiệu các cấu hình giúp tăng tính khả dụng và khả năng chịu tải của RDS, bao gồm triển khai Multi-AZ để tự động chuyển đổi dự phong, Read Replica để tăng hiệu năng đọc/truy vấn dữ liệu, và RDS Proxy để tối ưu kết nối."
---

Bài này sẽ giới thiệu các cấu hình giúp tăng tính khả dụng và khả năng chịu tải của RDS, bao gồm triển khai Multi-AZ để tự động chuyển đổi dự phong, Read Replica để tăng hiệu năng đọc/truy vấn dữ liệu, và RDS Proxy để tối ưu kết nối.


## Trong bài này:
- [1. RDS Multi-AZ](#multi-az-deployment)
    - [1.1. Multi-AZ DB Instance](#multi-az-instance)
    - [1.2. Multi-AZ DB Cluster](#multi-az-cluster)
- [2. RDS Read Replica](#read-replica)
- [3. RDS Proxy](#rds-proxy)
- [Tài liệu tham khảo](#reference)



<a name = "multi-az-deployment"></a>

## 1. RDS Multi-AZ

Một DB Instance đơn trong RDS được triển khai trong một AZ duy nhất, nếu AZ đó gặp sự cố, dịch vụ sẽ bị gián đoạn. Để tăng tính khả dụng và khả năng phục hồi, AWS cung cấp tính năng triển khai Multi-AZ, bao gồm hai cấu hình: Multi-AZ Instance và Multi-AZ Cluster. Người dùng có thể lựa chọn triển khai Multi-AZ khi tạo mới DB Instance, hoặc cũng có thể chuyển đổi từ DB Instance đơn đang chạy.


<a name = "multi-az-instance"></a>

### 1.1. Multi-AZ DB Instance

Ở cấu hình này, DB Instance chính được gọi là **Primary Instance**. Việc kết nối, đọc/ghi dữ liệu đều thực hiện trên Primary Instance.
RDS tạo một bản sao của Primary Instance trong một AZ khác, gọi là **Standby Instance** và không thể truy cập trực tiếp. Trong trường hợp Primary Instance gặp [sự cố](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.Failover.html) (mất kết nối, cần cập nhật phần mềm, không phản hồi, v.v.), RDS sẽ **tự động** chuyển đổi dự phòng (*failover*) sang Standby Instance, bằng cách cập nhật [endpoint](/2026/01/08/rds-fundamental#connect-to-rds) trỏ đến địa chỉ IP của Standby Instance.

<p>
<image src="/assets/20_rds_multi_az/rds_multi_az-instance.png" alt="RDS Multi-AZ Instance" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>

Việc sao chép dữ liệu từ Primary Instance sang Standby Instance là **đồng bộ** (_**synchronous replication**_). Cụ thể, mọi thao tác ghi (*write*) trên Primary Instance sẽ được ghi **đồng thời** vào Standby Instance. Chỉ khi thao tác ghi trên cả hai Instance thành công thì mới hoàn thành ghi (thuật ngữ trong SQL gọi là **commit transaction**) và trả kết quả về cho ứng dụng. 

Việc sao chép đồng bộ đảm bảo dữ liệu trên Standby Instance luôn nhất quán với Primary Instance, tránh mất mát dữ liệu trong trường hợp cần chuyển đổi. Tuy nhiên, độ trễ sẽ tăng thêm một chút do quá trình giao tiếp giữa hai Instance.

Về [sao lưu và phục hồi](/2026/01/08/rds-fundamental#backup-recovery), RDS sẽ thực hiện sao lưu từ Standby Instance thay vì Primary Instance, để tránh ảnh hưởng đến hiệu suất của Primary Instance.


<a name = "multi-az-cluster"></a>

### 1.2. Multi-AZ DB Cluster

Trong cấu hình này, DB Instance chính gọi là **Writer Instance**, có thể thực hiện cả hai thao tác đọc/ghi dữ liệu. RDS tạo thêm hai bản sao trong hai AZ khác, gọi là **Reader Instance**, chỉ cho phép đọc, giúp tăng khả năng chịu tải đọc của cơ sở dữ liệu, và không cho phép ghi để đảm bảo dữ liệu nhất quán.

<p>
<image src="/assets/20_rds_multi_az/rds_multi_az-cluster.png" alt="RDS Multi-AZ Cluster" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>


Việc sao chép dữ liệu từ Writer Instance sang hai Reader Instance là **bán đồng bộ** (_**semi-synchronous replication**_). Nghĩa là khi có thao tác ghi trên Writer Instance, dữ liệu sẽ được ghi đồng thời vào hai Reader Instance. Nhưng chỉ cần thao tác ghi trên **một trong hai** Reader Instance thành công thì hoàn thành ghi và trả kết quả về cho ứng dụng, không cần chờ cả hai xác nhận.

Khi Writer Instance gặp sự cố, RDS sẽ tự động chuyển đổi dự phòng sang một trong hai Reader Instance, biến nó thành Writer Instance mới, tuỳ vào Reader Instance nào đang có dữ liệu mới hơn. Quá trình này bản chất cũng là cập nhật endpoint trỏ đến đến Writer Instance mới, không cần thay đổi gì ở phía ứng dụng.



<a name = "read-replica"></a>

## 2. RDS Read Replica

Trong trường hợp cần tăng khả năng chịu tải đọc hơn nữa, có thể thêm **Read Replica** cho DB Instance hiện có, dù có được cấu hình Multi-AZ hay không. Read Replica là một bản sao **chỉ đọc** của DB Instance chính, không cho phép ghi dữ liệu. Một trường hợp phổ biến hay được đề cập trong các đề thi chứng chỉ AWS là sử dụng Read Replica để chạy các truy vấn phân tích, báo cáo nội bộ, tránh làm ảnh hưởng đến hiệu suất của DB Instance dùng cho ứng dụng chính.

<p>
<image src="/assets/20_rds_multi_az/rds-read-replica.png" alt="RDS Read Replica" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>

Việc sao chép dữ liệu từ DB Instance sang Read Replica là **bất đồng bộ** (_**asynchronous replication**_). Nghĩa là khi có thao tác ghi trên DB Instance chính, dữ liệu được ghi lại, sau đó mới sao chép sang các Read Replica, và cũng không chờ xác nhận gì từ chúng. Do đó, sẽ có độ trễ nhất định trong việc đồng bộ dữ liệu.



Một điểm quan trọng khác là Read Replica **có thể ở Region khác**. Việc sao chép dữ liệu vẫn là bất đồng bộ, thông qua kết nối riêng được AWS thiết lập, không cần dùng VPC Peering để kết nối hai VPC.
<!-- TODO: include link to VPC Peering Connection -->
Điều này giúp giảm độ trễ khi truy vấn trên toàn cầu, đồng thời việc sao lưu dữ liệu sang Region khác cũng tăng khả năng phục hồi của cơ sở dữ liệu lên [Global Resilience](/2025/11/12/aws_infrastructure/#resilience).


Việc tạo và xoá Read Replica được thực hiện **thủ công**, RDS không hỗ trợ tự động tăng giảm số lượng Read Replica dựa trên tải thực tế (*auto-scaling*). Tại thời điểm viết bài, với các [DB Engine](/2026/01/08/rds-fundamental#rds-concept) MySQL, PostgreSQL, MariaDB, và SQL Server, có thể tạo tối đa 15 Read Replica cho mỗi DB Instance. Với Oracle, giới hạn này là 5.


Trong trường hợp DB Instance chính gặp sự cố, RDS **không tự động chuyển đổi dự phòng** sang Read Replica. Tuy nhiên, người dùng có thể nâng cấp **thủ công** một Read Replica thành DB Instance, việc này sẽ tạo ra một DB Instance hoàn toàn mới, độc lập với DB Instance gốc, với endpoint mới, do đó cần thay đổi mã nguồn của ứng dụng để kết nối với endpoint mới này.

Các điểm cần phân biệt giữa RDS Multi-AZ và Read Replica được tổng hợp trong bảng dưới đây:

|                   | Triển khai Multi-AZ                              | Read Replica                                                                                   |
|:--------------------------------|:------------------------------------------------|:--------------------------------------------------------------|
| Mục đích             | Đảm bảo tính khả dụng cao, tự động chuyển đổi dự phòng.                            | Mở rộng khả năng chịu tải đọc đọc, cải thiện hiệu năng đọc.                                                 |
| Kiểu sao chép              | Đồng bộ                                 | Bất đồng bộ                              |
| Khả năng sử dụng           | Standby Instance trong Multi-AZ DB Instance không thể truy cập. Reader Instance trong Multi-AZ Cluster có thể đọc, không thể ghi. | Có thể đọc, không thể ghi.                                 |
| Chuyển đổi dự phòng        | Tự động chuyển đổi, giữ nguyên endpoint.          | Có thể nâng cấp thủ công thành DB Instance khác, thay đổi endpoint.           |
| Vị trí          | Khác AZ, cùng Region so với DB Instance chính.                     | Có thể cùng AZ, khác AZ, khác Region.(cross-region)                                 |


<a name = "rds-proxy"></a>

## 3. RDS Proxy

Mỗi kết nối đến DB Instance sẽ tiêu tốn tài nguyên của DB Instance (bộ nhớ, CPU). Nếu số lượng kết nối đồng thời quá lớn, DB Instance có thể quá tải, hiệu năng giảm sút hoặc thậm chí không chấp nhận kết nối mới. Hơn nữa, việc tạo và huỷ kết nối liên tục cũng làm tăng độ trễ, đặc biệt nếu ứng dụng có thời gian giữ kết nối ngắn, như các ứng dụng serverless chạy trên AWS Lambda (mỗi Lambda Function chỉ chạy trong tối đa 15 phút, nếu mỗi Function mở rồi đóng kết nối liên tục sẽ tiêu tốn nhiều tài nguyên).
<!-- TODO: include link to  Lambda-->

<p>
<image src="/assets/20_rds_multi_az/rds-proxy.png" alt="RDS Proxy" style="max-width:80%;height:auto;display:block;margin:0 auto;"/>
</p>


Để giải quyết vấn đề này, có thể sử dụng **RDS Proxy**. Như tên gọi, RDS Proxy làm trung gian giữa ứng dụng và DB Instance, quản lý một **nhóm kết nối** (_**connection pool**_) dài hạn đến DB Instance. Ứng dụng chỉ cần kết nối đến endpoint của RDS Proxy, sau đó RDS Proxy sẽ tái sử dụng các kết nối dài hạn này để phục vụ nhiều yêu cầu từ ứng dụng, thay vì phải mở kết nối mới đến DB Instance cho mỗi yêu cầu rồi đóng lại khi xử lý xong. Điều này giúp giảm tải cho DB Instance, đồng thời giảm độ trễ do việc tạo và huỷ kết nối liên tục.
RDS Proxy cũng tự động chuyển đổi dự phòng trong cấu hình Multi-AZ mà không cần thay đổi endpoint.

RDS Proxy hỗ trợ quản lý kết nối đến cả DB Instance đơn, Reader Instance trong Multi-AZ Cluster, và Read Replica, với kết nối đọc/ghi tương ứng. RDS Proxy cũng có thể sử dụng cho [Aurora](/2026/01/15/aurora).



<a name = "reference"></a>

## Tài liệu tham khảo

1. [Multi-AZ Single Standby](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZSingleStandby.html)
2. [Multi-AZ Cluster](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/multi-az-db-clusters-concepts.html)
3. [RDS Read Replica](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html)
4. [Cross-Region Read Replica](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.XRgn.html)
5. [Nâng cấp Read Replica thành DB Instance](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.Promote.html)
6. [RDS Proxy](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html)

Hy vọng qua bài này, bạn đọc đã nắm được các cấu hình giúp tăng tính khả dụng và khả năng chịu tải của RDS, và các trường hợp sử dụng phù hợp. Tiếp theo, hãy chuyển qua vấn đề bảo mật trong RDS.