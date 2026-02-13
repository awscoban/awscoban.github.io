---
layout: post
title: "29. Auto Scaling cho EC2"
title2: "Auto Scaling cho EC2"
date: 2026-02-02 00:00:00 +0700
permalink: /2026/02/02/ec2-autoscaling
categories: [EC2, High Availability]
tags: [EC2, High Availability]
img: /assets/29_ec2_autoscaling/asg.png
summary: "Auto Scaling (tạm dịch: Tự động Mở rộng) là dịch vụ giúp tự động điều chỉnh số lượng EC2 Instance theo các chính sách đã định nghĩa trước. Mục tiêu là đảm bảo ứng dụng luôn có đủ tài nguyên để đáp ứng nhu cầu, đồng thời tối ưu chi phí bằng cách giảm bớt tài nguyên khi không cần thiết."
---


Auto Scaling (tạm dịch: *Tự động Mở rộng*) là dịch vụ giúp tự động điều chỉnh số lượng EC2 Instance theo các chính sách đã định nghĩa trước. Mục tiêu là đảm bảo ứng dụng luôn có đủ tài nguyên để đáp ứng nhu cầu, đồng thời tối ưu chi phí bằng cách giảm bớt tài nguyên khi không cần thiết.


## Trong bài này:

- [1. Auto Scaling Group (ASG)](#asg)
    - [Launch Template](#launch-template)
- [2. Các Chính sách Mở rộng](#scaling-policy)
    - [2.1. Manual Scaling](#manual-scaling)
    - [2.2. Scheduled Scaling](#scheduled-scaling)
    - [2.3. Dynamic Scaling](#dynamic-scaling)
        - [2.3.1. Target Tracking Scaling](#target-tracking-scaling)
        - [2.3.2. Step Scaling](#step-scaling)
        - [2.3.3. Simple Scaling](#simple-scaling)
        - [Cooldown](#cooldown)
    - [2.4. Predictive Scaling](#predictive-scaling)
- [3. ASG Lifecycle & Hook](#lifecycle)
    - [3.1. Scale Out](#scale-out)
    - [3.2. Scale In](#scale-in)
    - [3.3. Lifecycle Hook](#lifecycle-hook)
- [Tài liệu tham khảo](#reference)


<a name = "asg"></a>

## 1. Auto Scaling Group (ASG)


Để sử dụng Auto Scaling, ta tạo một **Auto Scaling Group (ASG)**. Đây đơn giản là một tập hợp các EC2 Instance, người dùng có thể đặt 3 tham số để kiểm soát: 

- **Minimum Size**: số Instance tối thiểu trong nhóm. ASG sẽ không bao giờ giảm số instance xuống dưới mức này.
- **Maximum Size**: số Instance tối đa trong nhóm. Số lượng Instance sẽ không bao giờ vượt quá mức này.
- **Desired Capacity**: là số Instance sẽ chạy trong nhóm. Nếu không đặt [scaling policy](#scaling-policy), ASG sẽ luôn duy trì số Instance bằng với Desired Capacity. Nếu có scaling policy, khi cần thêm hoặc bớt Instance, AWS tự động thay đổi giá trị Desired Capacity trong khoảng từ Minimum Size đến Maximum Size, do đó số lượng Instance được cấp phát trong nhóm cũng thay đổi theo.

<p>
<image src="/assets/29_ec2_autoscaling/asg.png" alt="Auto Scaling Group" style="max-width:70%;height:auto;display:block;margin:0 auto;"/>
</p>


<a name = "launch-template"></a>

Khi tạo ASG, ta cần chỉ rõ thông số của các Instance trong nhóm. Thông số này được định nghĩa trong một **Launch Template**, bao gồm các thông tin như:
- [AMI](/2026/01/02/ec2-config#ami) sẽ dùng.
- [Loại Instance](/2025/12/16/ec2-fundamental#instance-type).
- Ổ [EBS](/2025/12/20/ebs#ebs-overview) cũng như [Instance Store](/2025/12/16/ec2-fundamental#ec2-storage).
- [VPC](/2025/11/13/vpc#vpc), [Subnet](/2025/11/13/vpc#subnet) và [Security Group](/2025/11/27/sg-nacl).
- [IAM Role](/2025/11/07/iam#iam-role).

Launch Template có thể được tái sử dụng cho nhiều ASG, giúp tiết kiệm thời gian và công sức cấu hình. Ngoài ra, Launch Template hỗ trợ **phiên bản (versioning)**, cho phép tạo các phiên bản khác nhau của cùng một template. Khi tạo hoặc cập nhật ASG, có thể chọn một phiên bản cụ thể của Launch Template. Lưu ý, **không thể chỉnh sửa** Launch Template sau khi được tạo, chỉ có thể tạo phiên bản mới rồi sử dụng.


<a name = "scaling-policy"></a>

## 2. Các Chính sách Mở rộng


<a name = "manual-scaling"></a>

### 2.2.1. Manual Scaling

Như tên gọi, Manual Scaling là điều chỉnh số Instance trong ASG thủ công bằng cách thay đổi giá trị Desired Capacity. AWS sẽ tự động khởi tạo hoặc dừng các Instance để đạt số lượng mong muốn.


<a name = "scheduled-scaling"></a>

### 2.2.2. Scheduled Scaling

Nếu ứng dụng có lưu lượng truy cập thay đổi theo lịch trình có thể dự đoán được (ví dụ: tăng vào giờ hành chính, giảm vào ban đêm), có thể sử dụng Scheduled Scaling, thao tác trên giao diện, CLI, hoặc API.

Ví dụ, giả sử ta có một ASG tên `awscoban-asg`, để tăng số Instance lên 3 vào lúc 9 giờ sáng và giảm về 1 lúc 7 giờ tối  mỗi ngày, có thể dùng hai lệnh AWS CLI sau:

```sh
aws autoscaling put-scheduled-update-group-action --scheduled-action-name inc-at-9am \
  --auto-scaling-group-name awscoban-asg --recurrence "0 9 * * *" --desired-capacity 3 

aws autoscaling put-scheduled-update-group-action --scheduled-action-name dec-at-7pm \
  --auto-scaling-group-name awscoban-asg --recurrence "0 19 * * *" --desired-capacity 1
```

Trong đó, tham số `--recurrence` dùng định dạng [cron](https://en.wikipedia.org/wiki/Cron) để chỉ định thời gian thực hiện hành động. Cú pháp như sau:

```
Phút   Giờ   Ngày-trong-tháng   Tháng   Ngày-trong-tuần
```
Giá trị `*` của một trường nghĩa là "bất kỳ". Ví dụ, `0 9 * * *` nghĩa là "9:00 tất cả các ngày".

<a name = "dynamic-scaling"></a>

### 2.2.3. Dynamic Scaling

Đây là một nhóm phương pháp mở rộng/thu hẹp ASG dựa trên tải thực tế, thông qua các [chỉ số (metric)](/2026/01/05/cloudwatch#metric) (như `CPUUtilization`, `NetworkIn`, `NetworkOut`, v.v.).
Do đó, đều cần tích hợp với [CloudWatch](/2026/01/05/cloudwatch).
Các phương pháp phổ biến gồm:

<a name = "target-tracking-scaling"></a>

#### 2.2.3.1. Target Tracking Scaling

Như tên gọi, phương pháp này theo dõi metric (có sẵn hoặc tự tạo) và tự động điều chỉnh số Instance để **giữ metric đó ở gần giá trị mục tiêu**.

Người dùng chỉ cần định nghĩa metric cần theo dõi và giá trị mục tiêu, AWS tự động tạo và quản lý [CloudWatch Alarm](/2026/01/05/cloudwatch#alarm) để kích hoạt hành động mở rộng/thu hẹp ASG khi metric rời xa giá trị mục tiêu.

Một chính sách hay dùng là trong các hệ thống phân tán, một nhóm EC2 Instance trong ASG đóng vai trò *consumer*, xử lý các tác vụ được đưa vào một hàng đợi [SQS](/2026/01/25/sqs). Ta có thể theo dõi số lượng tác vụ trong hàng đợi qua metric `ApproximateNumberOfMessagesVisible`, và đặt mục tiêu giữ số lượng tin nhắn trong hàng đợi ở mức thấp (ví dụ: 0 hoặc 1). Khi số tin nhắn tăng lên, ASG tự động tăng số lượng Instance để xử lý nhanh hơn, và ngược lại, giảm số Instance khi hàng đợi trống.

<a name = "step-scaling"></a>

#### 2.2.3.2. Step Scaling

Phương pháp này tăng/giảm số lượng Instance dựa theo các ngưỡng xác định của metric.

Ví dụ:
- Khi `CPUUtilization` tăng tới 50%, tăng thêm 1 Instance.
- Khi `CPUUtilization` tăng tới 75%, tăng thêm 2 Instance nữa.
- Khi `CPUUtilization` giảm tới 75%, giảm bớt 2 Instance. 
- Khi `CPUUtilization` giảm tới 50%, giảm bớt 1 Instance nữa.

Lưu ý, số Instance trong ASG sẽ luôn được giữ trong khoảng từ Minimum Size đến Maximum Size đã định nghĩa trước.


<a name = "simple-scaling"></a>

#### 2.2.3.3. Simple Scaling

Đây là phương pháp cũ, ít được dùng. Tương tự như Step Scaling, nhưng chỉ có một bước điều chỉnh duy nhất. 

<a name = "cooldown"></a>

**Cooldown**: Step Scaling và Simple Scaling đều nên áp dụng *cooldown*, đây là một khoảng thời gian chờ sau khi thực hiện hành động mở rộng/thu hẹp, để hệ thống **ổn định**. Trong thời gian này, dù metric có biến động vượt quá các ngưỡng khác, việc mở rộng/thu hẹp vẫn bị bỏ qua.  Mặc định, cooldown là 300 giây (5 phút), nhưng có thể tùy chỉnh theo nhu cầu.


<a name = "predictive-scaling"></a>

### 2.2.4. Predictive Scaling

Phương pháp này sử dụng Học Máy để phân tích tải trong quá khứ, từ đó dự đoán nhu cầu trong tương lai và tự động điều chỉnh số Instance trong ASG trước khi tải tăng lên. Phù hợp với:

- Ứng dụng có lưu lượng truy cập thay đổi **theo chu kỳ** (ví dụ: tăng vào giờ hành chính, giảm vào ban đêm), hoặc **định kỳ** (ví dụ: tăng vào các ngày lễ, sự kiện đặc biệt).
- Ứng dụng cần thời gian khởi động dài, do đó cần dự đoán trước để đảm bảo tài nguyên sẵn sàng khi cần, giảm độ trễ khi cần mở rộng.


<a name = "lifecycle"></a>

## 3. ASG Lifecycle & Hook

EC2 Instance trong ASG trải qua một chuỗi các trạng thái trong vòng đời (lifecycle) khác với [trạng thái của một Instance thông thường](/2025/12/16/ec2-fundamental#ec2-state). 


<p>
<image src="/assets/29_ec2_autoscaling/autoscaling-lifecycle.png" alt="Auto Scaling Group" style="max-width:80%;height:auto;display:block;margin:0 auto;"/>
</p>


<a name = "scale-out"></a>


### 3.1. Scale Out

Scale Out là quá trình thêm Instance mới vào ASG (*out* nghĩa là mở rộng ra), bao gồm các trạng thái sau:

- `Pending`: ASG bắt đầu khởi tạo Instance dựa trên Launch Template.
- `Pending:Wait`: Trạng thái tạm dừng, chờ thực hiện các thao tác tùy chỉnh (nếu có) qua [lifecycle hook](#lifecycle-hook). Ví dụ: cài đặt phần mềm, tải dữ liệu về, v.v.
- `Pending:Proceed`: Đã thực hiện xong hook, Instance đang được thêm vào [Load Balancer](/2026/02/05/load-balancer).
- `InService`: Instance đã sẵn sàng. Thực tế, nếu không cấu hình hook nào, Instance sẽ chuyển thẳng từ `Pending` sang `InService`.


<a name = "scale-in"></a>

### 3.2. Scale In

Ngược lại, Scale In là quá trình loại bỏ Instance khỏi ASG (*in* nghĩa là thu hẹp lại), bao gồm các trạng thái sau:

- `Terminating`: ASG bắt đầu dừng và xóa Instance.
- `Terminating:Wait`: Trạng thái tạm dừng, chờ thực hiện các thao tác tùy chỉnh (nếu có) qua [lifecycle hook](#lifecycle-hook). Ví dụ như gửi thông báo, sao lưu dữ liệu, v.v.
- `Terminating:Proceed`: Đã thực hiện xong hook, Instance đang được gỡ khỏi [Load Balancer](/2026/02/05/load-balancer).
- `Terminated`: Instance đã bị xóa hoàn toàn. Nếu không cấu hình hook nào, Instance sẽ chuyển thẳng từ `Terminating` sang `Terminated`.


<a name = "lifecycle-hook"></a>

### 3.3. Lifecycle Hook

**Hook** là cơ chế cho phép can thiệp vào lifecycle của các instance trong ASG, chèn các bước xử lý tùy chỉnh để cấu hình theo ý muốn. 
Hai loại hook phổ biến là:
- `autoscaling:EC2_INSTANCE_LAUNCHING`: kích hoạt khi một Instance mới được khởi tạo trong ASG, chuyển trạng thái từ `Pending` sang `Pending:Wait`.
- `autoscaling:EC2_INSTANCE_TERMINATING`: kích hoạt khi một Instance bị loại bỏ khỏi ASG, chuyển trạng thái từ `Terminating` sang `Terminating:Wait`.

Ví dụ, dưới đây ta tạo một hook tên `waiting-user-data` cho ASG `awscoban-asg`, gắn vào các Instance đang được khởi tạo khi Scale Out. Hook này đơn giản giữ Instance ở trạng thái `Pending:Wait` trong 3600 giây (1 giờ), để chờ chạy xong [User Data Script](/2026/01/02/ec2-config#user-data) để khởi tạo Instance trước khi sử dụng. 
```sh
aws autoscaling put-lifecycle-hook \
    --auto-scaling-group-name awscoban-asg \
    --lifecycle-hook-name waiting-user-data \
    --lifecycle-transition autoscaling:EC2_INSTANCE_LAUNCHING \
    --heartbeat-timeout 3600
```


<a name = "reference"></a>

## Tài liệu tham khảo


1. [Giới thiệu về EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html)
2. [Một Câu hỏi hay trên StackOverflow về ASG](https://stackoverflow.com/questions/36270873/aws-ec2-auto-scaling-groups-i-get-min-and-max-but-whats-desired-instances-lim)
3. [Launch Template](https://docs.aws.amazon.com/autoscaling/ec2/userguide/LaunchTemplates.html)
4. [Các Chính sách Mở rộng](https://docs.aws.amazon.com/autoscaling/ec2/userguide/scaling-overview.html)
5. [Scaling Lifecycle](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-lifecycle.html)
6. [Lifecycle Hook](https://docs.aws.amazon.com/autoscaling/ec2/userguide/lifecycle-hooks.html)

Tiếp theo, hay tìm hiểu về cân bằng tải với Load Balancer.