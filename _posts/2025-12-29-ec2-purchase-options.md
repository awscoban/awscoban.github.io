---
layout: post
title: "16. Các Phương thức Thuê EC2"
title2: "Các Phương thức Thuê EC2"
date: 2025-12-29
permalink: /2025/12/29/ec2-purchase-options
categories: [EC2]
tags: [EC2]
img: 
summary: "AWS cung cấp nhiều phương thức thuê EC2 Instance, nhằm tối ưu chi phí và đáp ứng các nhu cầu sử dụng khác nhau của người dùng. Trong bài này, ta sẽ tìm hiểu các phương thức thuê phổ biến nhất: On-Demand, Spot, Reserved Instance, Saving Plan, Dedicated Host, Dedicated Instance, và Capacity Reservation."
---

AWS cung cấp nhiều phương thức thuê EC2 Instance, nhằm tối ưu chi phí và đáp ứng các nhu cầu sử dụng khác nhau của người dùng. Trong bài này, ta sẽ tìm hiểu các phương thức thuê phổ biến nhất: On-Demand, Spot, Reserved Instance, Saving Plan, Dedicated Host, Dedicated Instance, và Capacity Reservation.


## Trong bài này:

- [1. On-Demand Instance](#on-demand-instance)
- [2. Spot Instance](#spot-instance)
- [3. Reserved Instance](#reserved-instance)
- [4. Saving Plan](#saving-plan)
- [5. Dedicated Host](#dedicated-host)
- [6. Dedicated Instance](#dedicated-instance)
- [7. Capacity Reservation](#capacity-reservation)
- [Tài liệu tham khảo](#reference)

<a name="on-demand-instance">

## 1. On-Demand Instance

Nghĩa là **thuê theo nhu cầu**. Rất đơn giản, người dùng chi trả chi phí tính toán theo thời gian Instance ở trạng thái [`running`](/2025/12/16/ec2-fundamental#ec2-state). Chi phí tính theo **giây** sử dụng, mức phí cố định và được công khai trên [trang chủ AWS](https://aws.amazon.com/ec2/pricing/on-demand/).

Người dùng **không có trách nhiệm dài hạn** phải sử dụng Instance trong bao lâu, không phải trả trước, cũng không có khuyến mại. Đây là lựa chọn **mặc định**. On-Demand Instance **không bị gián đoạn** khi sử dụng, nên phù hợp với các tác vụ ngắn hạn và **không được phép ngắt quãng**.

Tuỳ đặc điểm tác vụ, có thể lựa chọn các phương thức thuê khác để tiết kiệm chi phí.

<a name="spot-instance">

## 2. Spot Instance

Bản chất mỗi EC2 Instance là một máy ảo chạy trên một Host vật lý, mỗi Host chạy nhiều Instance, nên khi số lượng Instance chạy trên Host ít, Host đó vẫn có dư năng lực tính toán (gọi là **capacity**). AWS khuyến khích tận dụng phần dư này bằng cơ chế Spot Instance, chi phí có thể **giảm tới 90%** so với On-Demand.

Tuy nhiên, hạn chế là khi người dùng khác cần On-Demand Instance trên Host, AWS sẽ dừng các Spot Instance để cung cấp tài nguyên. Do đó, Spot Instance có khả năng **bị gián đoạn**, chỉ phù hợp cho những tác vụ ít quan trọng hoặc có thể chạy lại dễ dàng.

Trên đây là đặc điểm quan trọng nhất về Spot Instance mà bạn đọc cần ghi nhớ. Cơ chế cụ thể có thể xem thêm [tại đây](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/how-spot-instances-work.html).


<a name="reserved-instance">

## 3. Reserved Instance 

Với cách thức này, người dùng cam kết sử dụng EC2 Instance trong thời gian dài, cụ thể là **1 năm** hoặc **3 năm**, AWS sẽ giảm chi phí thuê so với On-Demand Instance. Instance cũng sẽ **không bị gián đoạn** khi sử dụng. 

Các lựa chọn thanh toán:
- **Trả trước Hoàn toàn** (**All Upfront**): được giảm giá nhiều nhất
- **Trả trước Một phần** (**Partially Upfront**)
- **Trả sau** (**No Upfront**): được giảm giá ít nhất


Có 2 loại Reserved Instance:
- **Standard**: **có thể thay đổi** (*modify*) Instance giữa chừng nếu nhu cầu tính toán thay đổi. Ví dụ, chia nhỏ Reserved Instance, gộp hai hay nhiều Instance thành một Instance kích thước lớn hơn, hay thay đổi AZ của Instance. Instance vẫn phải cùng loại, hệ điều hành, v.v., chỉ khác kích thước và vị trí AZ. **Không thể trao đổi** (*exchange*) Reserved Instance thành loại khác.

- **Convertible**: giảm giá ít hơn, nhưng hỗ trợ cả **thay đổi** và **trao đổi** Instance thành loại khác (khác cấu hình, hệ điều hành, v.v.). Đọc thêm [tại đây](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ri-convertible-exchange.html)


Khi đã thuê Reserved Instance, **không thể huỷ bỏ giao dịch này**. Người dùng trả phí cho toàn bộ thời hạn, bất kể có sử dụng hay không. Do đó, Reserved Instance phù hợp với các tác vụ có kế hoạch **chạy trong thời gian dài**.

<a name="saving-plan">

## 4. Saving Plan

Đây là phương thức thuê Instance mới hơn, linh hoạt hơn Reserved Instance. Người dùng cam kết trả một mức phí cố định hàng giờ trong thời gian dài (vẫn là 1 hoặc 3 năm), đổi lại được giảm giá so với mức phí On-Demand, khi sử dụng EC2 hoặc các dịch vụ tính toán khác như Fargate (*container*), Lambda (*serverless computing*), SageMaker (Học Máy). Mức phí cố định này áp dụng bất kể người dùng sử dụng loại Instance, hệ điều hành, kích thước, hay Region nào. 
<!-- TODO: include link to Fargate -->
<!-- TODO: include link to Lambda -->
<!-- TODO: include link to SageMaker -->

Trong thực tế, Saving Plan là một phương án rất đáng cân nhắc để tiết kiệm chi phí. Bạn đọc có thể tìm hiểu thêm qua [tài liệu chính thức của AWS](https://docs.aws.amazon.com/savingsplans/latest/userguide/plan-types.html).


<a name = "dedicated-host">

## 5. Dedicated Host 

Nghĩa là **thuê toàn bộ Host vật lý** để chạy EC2 Instance. Người dùng có tuỳ ý sử dụng Host, chia thành nhiều Instance theo ý muốn, **không phải chia sẻ phần cứng** với người khác. Ngoài ra cũng phù hợp khi cần sử dụng các giấy phép phần mềm gắn trực tiếp cho socket, core, hoặc máy ảo (*BYOL - Bring Your Own License*).

<a name="dedicated-instance">

## 6. Dedicated Instance

Nếu vẫn muốn dùng riêng phần cứng và không cần kiểm soát Instance đặt ở Host nào, có thể chọn Dedicated Instance. Theo cách này, EC2 Instance của bạn vẫn chạy trên Host riêng, không chia sẻ với người dùng khác, chỉ không thể chọn Host cụ thể. Ngoài ra không có khác biệt về hiệu năng hay bảo mật so với Dedicated Host.

Bảng dưới đây tóm tắt sự khác biệt giữa Dedicated Host và Dedicated Instance, nguồn: [tài liệu AWS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/dedicated-hosts-overview.html):

|                      | Dedicated Host                                      | Dedicated Instance                        |
|:--|:--|:--|
| Máy chủ vật lý riêng                   | Có, được tự chọn | Có, nhưng không được chọn |
| Chia sẻ dung lượng Instance            | Có thể chia sẻ dung lượng Instance với tài khoản khác | Không hỗ trợ                              |
| Thanh toán                             | Tính phí theo toàn bộ Host                             | Tính phí theo từng Instance               |
| Hiển thị socket, core, host ID         | Có               | Không                            |
| Host-Instance Affinity    | Cho phép triển khai Instance lên cùng một Host vật lý cố định theo thời gian | Không hỗ trợ                              |
| Kiểm soát vị trí đặt Instance          | Có quyền kiểm soát cách Instance được đặt trên Host vật lý | Không hỗ trợ                              |
| Tự động phục hồi Instance              | Có | Có                                    |
| Mang theo giấy phép phần mềm (BYOL)    | Hỗ trợ                                               | Chỉ hỗ trợ Microsoft SQL Server with License Mobility và Windows Virtual Desktop Access (VDA)                      |


Dedicated Host và Dedicated Instance đều có chi phí cao hơn so với các phương thức thuê khác, dùng khi cần đáp ứng các yêu cầu đặc biệt về pháp lý hay ràng buộc, giấy phép phần mềm, cũng như bảo mật.

<a name = "capacity-reservation">

## 7. Capacity Reservation 

Capacity Reservation cho phép người dùng **đặt trước tài nguyên** EC2 trong một AZ cụ thể, đảm bảo có thể khởi chạy Instance khi cần, đặc biệt hữu ích trong các tình huống nhu cầu tăng cao đột ngột. Người dùng chỉ trả phí khi Instance đang ở trạng thái `running` hoặc `stopped`, không phải trả phí chỉ để giữ chỗ.

### Tổng kết

| Phương thức thuê         | Đặc điểm chính                                                                 | Cam kết thời gian | Nguy cơ bị gián đoạn | Tiết kiệm chi phí | Thanh toán         | Phù hợp cho               |
|:--|:--|:--|:--|:--|:--|:--|
| On-Demand Instance      | Thuê theo nhu cầu, không cam kết, giá cố định                                 | Không             | Không                | Không             | Theo thời gian sử dụng Instance      | Tác vụ ngắn hạn, không ổn định            |
| Spot Instance           | Giá rẻ, tận dụng tài nguyên dư, có thể bị thu hồi bất cứ lúc nào              | Không             | Có                   | Lên tới 90%       | Theo giây      | Batch job, workload chịu được gián đoạn   |
| Reserved Instance       | Cam kết sử dụng lâu dài, giảm giá mạnh                                        | 1 hoặc 3 năm       | Không                | 30-72%            | Trả trước/định kỳ  | Workload ổn định, chạy lâu dài            |
| Saving Plan             | Cam kết chi tiêu tối thiểu, linh hoạt loại Instance, giảm giá                 | 1 hoặc 3 năm       | Không                | 30-72%            | Trả trước/định kỳ  | Workload đa dạng, tối ưu chi phí           |
| Dedicated Host          | Thuê toàn bộ máy chủ vật lý, kiểm soát phần cứng, BYOL                        | Tuỳ chọn           | Không                | Không             | Theo thời gian sử dụng Host          | Yêu cầu tuân thủ, giấy phép đặc biệt      |
| Dedicated Instance      | Instance chạy trên host riêng, không chia sẻ với tài khoản khác               | Tuỳ chọn           | Không                | Không             | Theo thời gian sử dụng Instance      | Yêu cầu tách biệt vật lý, bảo mật          |
| Capacity Reservation    | Đặt trước tài nguyên, đảm bảo có Instance khi cần                             | Tuỳ chọn           | Không                | Không             | Theo thời gian sử dụng Instance      | Đảm bảo tài nguyên cho workload quan trọng |



<a name = "reference"></a>

## Tài liệu tham khảo

1. [Các Phương thức Thuê Instance - Tài liệu AWS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-purchasing-options.html)
2. [Cơ chế Cấp phát Spot Instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/how-spot-instances-work.html)
3. [Chi phí của Reserved Instance](https://aws.amazon.com/ec2/pricing/reserved-instances/pricing/)
4. [Các loại Saving Plan](https://docs.aws.amazon.com/savingsplans/latest/userguide/plan-types.html)
5. [Dedicated Host](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/dedicated-hosts-overview.html)
6. [Capacity Reservation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/capacity-reservation-overview.html)
