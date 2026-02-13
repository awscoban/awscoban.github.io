---
layout: post
title: "18. CloudWatch"
title2: "CloudWatch"
date: 2026-01-05
permalink: /2026/01/05/cloudwatch
categories: [Monitoring, EC2]
tags: [Monitoring, EC2]
img: /assets/18_cloudwatch/cloudwatch.png
summary: "CloudWatch là dịch vụ giám sát, theo dõi hiệu suất và trạng thái của tài nguyên AWS. CloudWatch thu thập và lưu trữ các số liệu (metric) từ các dịch vụ AWS như EC2, RDS, Lambda, v.v., cho phép người dùng tạo biểu đồ trực quan, cảnh báo, và phản hồi tự động dựa trên các số liệu này."
---

CloudWatch là dịch vụ giám sát, theo dõi hiệu suất và trạng thái của tài nguyên AWS. CloudWatch thu thập và lưu trữ các số liệu (**metric**) từ các dịch vụ AWS như EC2, RDS, Lambda, v.v., cho phép người dùng tạo biểu đồ trực quan, cảnh báo, và phản hồi tự động dựa trên các số liệu này.



## Trong bài này:

- [1. Metric](#metric)
    - [1.1. Namespace](#namespace)
    - [1.2. Dimension](#dimension)
    - [1.3. Resolution](#resolution)
    - [1.4. Period](#data-point)
    - [1.5. Statistics](#statistics)
    - [1.6. Alarm](#alarm)
    - [1.7. Dashboard](#cloudwatch-dashboard)
    - [1.8. Metric Insights](#metric-insights)
- [2. CloudWatch Logs](#cloudwatch-logs)
    - [2.1. Log Event](#log-event)
    - [2.2. Log Stream](#log-stream)
    - [2.3. Log Group](#log-group)
    - [2.4. Metric Filter](#metric-filter)
    - [2.5. Subscription Filter](#subscription-filter)
    - [2.6. Logs Insights Query](#logs-insights)
- [3. CloudWatch Agent cho EC2](#cloudwatch-agent)
- [Tài liệu tham khảo](#reference)


<p>
<image src="/assets/18_cloudwatch/cloudwatch.png" alt="CloudWatch" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>

CloudWatch thu thập dữ liệu từ các tài nguyên và dịch vụ AWS, chia thành hai loại chính:

- **Metric**: Là các số liệu định lượng, như CPU Utilization, Disk I/O, Network Traffic, v.v. Metric được thu thập theo thời gian và lưu trữ để phân tích xu hướng hiệu suất.
- **Logs**: Là các bản ghi chi tiết về hoạt động của ứng dụng và hệ thống, dưới dạng dữ liệu thuần, thường được người dùng tự cấu hình, giúp theo dõi và phân tích hành vi ứng dụng.

<a name = "metric"></a>

## 1. Metric

Các dịch vụ AWS cung cấp sẵn nhiều metric mặc định, ngoài ra người dùng có thể tạo metric tùy chỉnh. Các khái niệm quan trọng liên quan đến metric bao gồm:

<a name = "namespace"></a>

#### Namespace
Mỗi dịch vụ AWS có một namespace riêng để chứa metric, ví dụ `AWS/EC2` cho EC2, `AWS/RDS` cho [RDS](/2026/01/08/rds-fundamental) (*Relational Database Service*, một dịch vụ cơ sở dữ liệu SQL), v.v. Tên của namespace có thể tuỳ chỉnh, nên đặt tên để dễ nhận biết.

#### Metric

Khái niệm cốt lõi nhất, đây là các điểm dữ liệu theo thời gian được thu thập. Mỗi metric có một tên duy nhất trong namespace, và chỉ tồn tại trong Region nơi nó được tạo. **Không thể xoá** metric, nhưng dữ liệu của metric quá 16 thánh sẽ tự động bị xoá.

Ví dụ, trong namespace `AWS/EC2`, có metric `CPUUtilization` đo mức độ sử dụng CPU. Mỗi điểm dữ liệu của metric này có timestamp và giá trị tương ứng tại thời điểm đó. Trên AWS CLI, có thể chạy lệnh sau để xem các metric:

```sh
aws cloudwatch list-metrics --namespace AWS/EC2
```

Đầu ra:

```json
{
  "Metrics" : [
    ...
    {
        "Namespace": "AWS/EC2",
        "Dimensions": [
            {
                "Name": "InstanceId",
                "Value": "i-1234567890abcdef0"
            }
        ],
        "MetricName": "NetworkOut"
    },
    {
        "Namespace": "AWS/EC2",
        "Dimensions": [
            {
                "Name": "InstanceId",
                "Value": "i-1234567890abcdef0"
            }
        ],
        "MetricName": "CPUUtilization"
    },
    {
        "Namespace": "AWS/EC2",
        "Dimensions": [
            {
                "Name": "InstanceId",
                "Value": "i-1234567890abcdef0"
            }
        ],
        "MetricName": "NetworkIn"
    },
    ...
  ]
}
```


<a name = "dimension"></a>

#### Dimension

Ở trên ta thấy mỗi metric có thể chứa 0 hoặc nhiều dimension. Đây là các thuộc tính bổ sung để phân loại và lọc metric. Ví dụ, trong namespace EC2, metric `CPUUtilization` (đo mức độ sử dụng CPU), ngoài dimension `InstanceId` để phân biệt từng Instance, `InstanceType` cho biết loại Instance, v.v. Để xem các metric của một Instance cụ thể, có thể lọc theo dimension `InstanceId` như sau:

```sh
aws cloudwatch list-metrics --namespace AWS/EC2 --dimensions Name=InstanceId,Value=i-1234567890abcdef0
```

<a name = "resolution"></a>
#### Resolution
Là tần suất thu thập dữ liệu của metric. Có hai loại resolution:
- **Standard Resolution**: thu thập theo phút.
- **High Resolution**: thu thập theo giây.

<a name = "data-point"></a>

#### Period
Là khoảng thời gian tổng hợp dữ liệu của metric, tính bằng giây. Ví dụ, nếu period là 300 giây (5 phút), thì các điểm dữ liệu sẽ được tổng hợp trong mỗi khoảng 5 phút.


#### Statistics

Là các phép toán áp dụng lên tập điểm dữ liệu của metric trong khoảng thời gian `period` để gộp lại thành một giá trị. AWS hỗ trợ `SampleCount`, `Average`, `Sum`, `Minimum`, `Maximum`, v.v. Chi tiết [tại đây](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Statistics-definitions.html).


<a name = "alarm">

#### Alarm
Cảnh báo dựa theo ngưỡng của metric. Alarm theo dõi giá trị metric và hành động (gọi các dịch vụ AWS khác) nếu giá trị vượt qua một ngưỡng đã định. Ví dụ, có thể tạo alarm để thông báo qua [Simple Notification Service (SNS)](/2026/01/22/step-function#sns) nếu CPU Utilization của một EC2 Instance vượt quá 80% trong 5 phút liên tục, hoặc tự động thêm Instance thông qua [Auto Scaling](/2026/02/02/ec2-autoscaling). Việc này giúp tự động hoá việc phản ứng với các sự kiện theo trạng thái của tài nguyên.


<a name = "cloudwatch-dashboard"></a>

#### Dashboard

Đây là công cụ hiển thị trực quan metric và alarm trên giao diện AWS. CloudWatch cung cấp các dashboard dựng sẵn và cho phép tự tuỳ chỉnh để theo dõi hiệu suất và trạng thái của tài nguyên AWS theo thời gian thực.

<a name = "metric-insights"></a>

#### Metric Insights

Metric Insights là tính năng nâng cao của CloudWatch, cho phép truy vấn và phân tích metric sử dụng ngôn ngữ truy vấn tương tự SQL, giúp trích xuất thông tin chi tiết từ tập metric lớn một cách nhanh chóng và hiệu quả.

Ví dụ, để xem trung bình mức sử dụng CPU của các EC2 Instance, sắp xếp theo giá trị giảm dần, có thể sử dụng truy vấn sau:

```sql
SELECT AVG(CPUUtilization) 
  FROM SCHEMA("AWS/EC2", InstanceId) 
  GROUP BY InstanceId 
  ORDER BY AVG() DESC
```

Trong đó, `SCHEMA("AWS/EC2", InstanceId)` chỉ định `namespace` và `dimension` cần truy vấn.


<a name = "cloudwatch-logs"></a>

## 2. CloudWatch Logs

Metric cung cấp giá trị định lượng theo thời gian, còn Log lưu trữ chi tiết hoạt động của ứng dụng và hệ thống dưới dạng bản ghi. CloudWatch Logs giúp thu thập, lưu trữ, và phân tích log.

<a name = "log-event"></a>

#### Log Event 

Là một bản ghi đơn lẻ trong CloudWatch Logs, chỉ chứa **timestamp** và **message** (nội dung log).

<a name = "log-stream"></a>

#### Log Stream 

Là chuỗi các log event liên quan, ứng với một nguồn cụ thể, như [EC2 Instance](/2025/12/16/ec2-fundamental), [Lambda Function](/2026/01/18/lambda-fundamental#function), v.v. 

<a name = "log-group"></a>

#### Log Group

Là tập hợp các Log Stream có cùng cấu hình (dịch vụ, thời gian lưu trữ, quyền truy cập, v.v.). Ví dụ, một Log Stream tương ứng với một EC2 Instance, tất cả Log Stream của các Instance có thể được nhóm vào một Log Group.

<a name = "metric-filter"></a>

#### Metric Filter

Metric Filter cho phép **tạo metric từ log**. Người dùng định nghĩa các mẫu tìm kiếm (*filter pattern*) trong log, Metric Filter sẽ trích xuất các giá trị cụ thể và chuyển đổi chúng thành metric có thể sử dụng trong CloudWatch. 

Metric Filter được tạo trong một Log Group, và áp dụng cho tất cả các Log Stream bên trong. Ví dụ, giả sử ứng dụng ghi log với định dạng như sau:

```
[INFO]  2026-01-05-10:15:30  127.0.0.1  "GET /api/awscoban HTTP/1.1"  200
```

Để tạo Metric Filter đếm số lần xuất hiện lỗi HTTP 404 trong log, ta có thể sử dụng filter pattern sau:

```
[Message, Timestamp, IP, RequestInfo, StatusCode=404]
```

Filter pattern này khớp với cấu trúc của log (như `Message` ứng với `[INFO]`, `Timestamp` ứng với `2026-01-05-10:15:30`, v.v.). Trường cuối cùng `StatusCode=404` là điều kiện để lọc các log có lỗi 404. Metric Filter sẽ đếm số lần xuất hiện của các log này và tạo metric.

<a name = "subscription-filter"></a>

#### Subscription Filter

Subscription Filter cho phép gửi log từ CloudWatch Logs đến các dịch vụ khác như [Lambda](/2026/01/18/lambda-fundamental), [Kinesis Data Streams](/2026/01/31/kinesis-glue#kinesis-data-streams), để xử lý và phân tích thêm.
Cũng có thể sử dụng tính năng này để tập trung lưu trữ log tại một vị trí, trong trường hợp cần tập trung log từ nhiều tài khoản. Bạn đọc có thể tìm hiểu thêm [tại đây](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/SubscriptionFilters.html).


<a name = "logs-insights"></a>

#### Logs Insights Query

Đây là công cụ truy vấn và phân tích log rất mạnh, sử dụng ngôn ngữ truy vấn riêng biệt ([*Logs Insights Query Language*](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html)) để trích xuất thông tin chi tiết từ tập log lớn. Logs Insights tự động xác định các trường thông tin, hỗ trợ lọc và tổng hợp dữ liệu phức tạp, cũng như tích hợp với CloudWatch Dashboard để trực quan hoá kết quả truy vấn.

Ví dụ, truy vấn sau tìm kiếm và tổng hợp các log có chứa từ khoá `ERROR`, sắp xếp theo thời gian:

```sql
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20
```

<a name = "cloudwatch-agent"></a>

## 3. CloudWatch Agent cho EC2

Namespace `AWS/EC2` cung cấp sẵn nhiều metric quan trọng như `CPUUtilization`, `DiskReadBytes`, `DiskWriteBytes`, `NetworkIn`, `NetworkOut`. Tuy nhiên, để thu thập thêm các [metric chi tiết](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/metrics-collected-by-CloudWatch-agent.html) như **bộ nhớ**, ổ đĩa , v.v., cần cài đặt **CloudWatch Agent**.

CloudWatch Agent là một [phần mềm mã nguồn mở](https://github.com/aws/amazon-cloudwatch-agent) chạy trên EC2 Instance và cả máy chủ doanh nghiệp tự quản lý (on-premises), giúp thu thập metric/log chi tiết và gửi đến CloudWatch. Các bước cơ bản để sử dụng CloudWatch Agent gồm:

- Thêm 2 policy, cụ thể là [`AmazonSSMManagedInstanceCore`](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AmazonSSMManagedInstanceCore.html) và [`CloudWatchAgentServerPolicy`](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/CloudWatchAgentServerPolicy.html), vào [IAM Role](/2025/11/07/iam#iam-role) gắn với EC2 Instance. 2 policy cung cấp các quyền cần thiết để thu thập dữ liệu trên EC2 và gửi đến CloudWatch.
- Tải phần mềm CloudWatch Agent về EC2 Instance. Có thể sử dụng AWS Systems Manager (SSM) để tự động hoá việc này.
<!-- TODO: include link to SSM -->
- Cấu hình CloudWatch Agent bằng [file cấu hình](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html), xác định các metric cần thu thập. Với log, mặc định mỗi log file sẽ được ghi trong một Log Group riêng. Ví dụ, file cấu hình sau thu thập các metric về CPU, bộ nhớ, ổ đĩa, và log từ hai tệp hệ thống `/var/log/messages` và `/var/log/syslog`, gửi đến các Log Group tương ứng trong CloudWatch Logs. Người dùng có thể tự ghi log ứng dụng của mình vào các tệp khác, và cấu hình CloudWatch Agent để thu thập chúng.

```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "metrics": {
    "metrics_collected": {
      "cpu": {
        "metrics_collection_interval": 60,
        "usage_active": true
      },
      "disk": {
        "metrics_collection_interval": 60,
        "append_dimensions": {
          "InstanceId": "${aws:InstanceId}"
        },
        "measurement": [
          "used_percent"
        ]
      },
      "mem": {
        "metrics_collection_interval": 60,
        "measurement": [
          "mem_used_percent"
        ]
      }
    },
    "aggregation_dimensions": [
      [
        "InstanceId"
      ]
    ]
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/messages",
            "log_group_name": "/ec2/messages",
            "log_stream_name": "{instance_id}/messages.log",
            "timezone": "UTC"
          },
          {
            "file_path": "/var/log/syslog",
            "log_group_name": "/ec2/syslog",
            "log_stream_name": "{instance_id}/syslog.log",
            "timezone": "UTC"
          }
        ]
      }
    },
    "log_stream_name": "awscoban_log_stream"
  }
}
```

- Khởi động CloudWatch Agent trên EC2 Instance để bắt đầu thu thập dữ liệu.  


<a name = "reference"></a>

## Tài liệu tham khảo

1. [CloudWatch Metric](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_concepts.html)
2. [Sử dụng CloudWatch Alarm](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)
3. [Các khái niệm trong CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatchLogsConcepts.html)
4. [Metric Filter](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/MonitoringLogData.html)
5. [Subscription Filter](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/SubscriptionFilters.html)
6. [CloudWatch Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html)
7. [Các metric thu thập bởi CloudWatch Agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/metrics-collected-by-CloudWatch-agent.html)
8. [Cài đặt và cấu hình CloudWatch Agent trên EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/install-and-configure-cloudwatch-agent-using-ec2-console.html)


Trong khuôn khổ bài viết này, mình chỉ giới thiệu ngắn gọn các khái niệm cần nắm vững trong CloudWatch, hay xuất hiện trong các bài thi chứng chỉ AWS. Bạn đọc cần thực hành thực tế nhiều để hiểu sâu hơn. 