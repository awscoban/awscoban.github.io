---
layout: post
title: "25. Step Function"
title2: "Step Function"
date: 2026-01-22 00:00:00 +0700
permalink: /2026/01/22/step-function
categories: [Serverless]
tags: [Serverless]
img: /assets/25_step_func/stepfunctions_sample.png
summary: "Step Function là một dịch vụ không máy chủ giúp phối hợp các dịch vụ AWS khác thành một quy trình làm việc (workflow), giúp xây dựng các ứng dụng phân tán, tự động hóa các quy trình, hay xây dựng luồng dữ liệu dùng cho Học Máy, bù đắp các hạn chế của Lambda Function. Cùng tìm hiểu chi tiết hơn về Step Function qua bài viết này, cùng với SNS, dịch vụ gửi thông báo trên AWS."
---




## Trong bài này:

- [1. Step Function](#step-function)
    - [1.1. Quy trình](#workflow)
    - [1.2. Trạng thái](#state)
    - [1.3. Ngôn ngữ Mô tả Trạng thái](#state-lang)
- [2. Simple Notification Service (SNS)](#sns)
- [Tài liệu tham khảo](#reference)


<a name = "step-function"></a>

## 1. Step Function

[Lambda](/2026/01/18/lambda-fundamental) có giới hạn về thời gian thực thi (tối đa 15 phút), không phù hợp với các tác vụ dài và phức tạp. 
Step Function là một dịch vụ không máy chủ giúp phối hợp các dịch vụ AWS khác thành một quy trình làm việc (*workflow*), giúp xây dựng các ứng dụng phân tán, tự động hóa các quy trình, hay xây dựng luồng dữ liệu dùng cho Học Máy.


Có hai khái niệm chính trong Step Function: **quy trình** (**_workflow_**) và **trạng thái** (**_state_**).

<a name = "workflow"></a>

### 1.1. Quy trình

Là một chuỗi các bước xử lý được kích hoạt bằng sự kiện (*event-driven*). Mỗi bước trong quy trình được gọi là một **trạng thái** (**_state_**). Quy trình bắt đầu ở trạng thái **START** kết thúc ở **END**, ở giữa là các trạng thái trung gian thực hiện một tác vụ cụ thể. 

<p>
<image src="/assets/25_step_func/stepfunctions_sample.png" alt="Step Function Workflow" style="max-width:60%;height:auto;display:block;margin:0 auto;"/>
</p>

Hình trên mô tả một quy trình khi gọi API qua API Gateway, thông báo kết quả (thành công hoặc lỗi) qua [Simple Notification Service (SNS)](#sns). Mỗi khối là một trạng thái, thực thi các tác vụ tương ứng: gọi API, kiểm tra kết quả, gửi thông báo.
<!-- TODO: include link to API Gateway -->


Step Function hỗ trợ hai loại quy trình chính:
- **Standard Workflow**: phù hợp cho các quy trình dài, cần kiểm soát chính xác (vì có lưu lại lịch sử thực thi). Mỗi trạng thái **chỉ được thực thi một lần**, và có thể chạy trong tối đa 1 năm. Tại thời điểm viết bài, tốc độ thực thi của Standard Workflow là 2000 lần/giây, tốc độ chuyển trạng thái 4000 lần/giây.
- **Express Workflow**: phù hợp cho các quy trình có tần suất sự kiện cao, như xử lý dữ liệu streaming. Mỗi trạng thái được thực thi **ít nhất một lần**, và thời gian chạy tối đa 5 phút. Tốc độ thực thi 100,000 lần/giây, tốc độ chuyển trạng thái gần như không giới hạn.


Quy trình được mô tả bằng [ngôn ngữ Mô tả Trạng thái (Amazon States Language)](#state-lang), định dạng JSON.




<a name = "state"></a>

### 1.2. Trạng thái

Mỗi trạng thái trong quy trình thực hiện một tác vụ cụ thể. Các loại trạng thái trong Step Function gồm:

- **Task**: thực thi một tác vụ, như gọi Lambda function, gửi thông báo qua SNS, v.v.
- **Choice**: thực hiện phân nhánh điều kiện, tương tự cú pháp `if` trong các ngôn ngữ lập trình.
- **Parallel**: thực thi song song nhiều trạng thái.
- **Map**: lặp qua một mảng dữ liệu, thực thi một trạng thái cho mỗi mục trong mảng.
- **Wait**: tạm dừng quy trình trong một khoảng thời gian xác định.
- **Pass**: chuyển tiếp dữ liệu từ trạng thái này sang trạng thái khác, không thực thi tác vụ nào.
- **Succeed**: dùng để ra tín hiệu thành công kết thúc quy trình, một nhánh trong trạng thái Parallel, hoặc một vòng lặp trong trạng thái Map. Thường dùng làm đầu vào cho trạng thái Choice.
- **Fail**: kết thúc quy trình và trả về lỗi.





<a name = "state-lang"></a>

### 1.3. Ngôn ngữ Mô tả Trạng thái (Amazon States Language)

Đây là ngôn ngữ mô tả quy trình trong Step Function, định dạng JSON. Mỗi quy trình được biểu diễn dưới dạng một đối tượng JSON với các trường chính:
- `StartAt`: tên trạng thái bắt đầu.
- `States`: chứa tất cả các trạng thái trong quy trình, mỗi trạng thái là một đối tượng con với các trường như `Type` chỉ [loại trạng thái](#state), `Resource` là [ARN](/2025/11/07/iam#arn) của tài nguyên được sử dụng, `Next` chỉ trạng thái tiếp theo, `End` cho biết đây có phải trạng thái kết thúc hay không, v.v.

Ví dụ, dưới đây là mô tả quy trình trong hình minh hoạ phía trên. Bắt đầu từ trạng thái `Call API`, là một trạng thái Task, thực hiện gọi API qua API Gateway. Kết quả trả về được chuyển sang trạng thái Choice `Call Successful?`, nếu trả về HTTP 200, chuyển sang trạng thái `Notify Success`, ngược lại chuyển sang `Notify Failure`. Cả hai trạng thái này đều kết thúc quy trình.

```json
{
  "Comment": "Quy trình trên hình minh hoạ",
  "StartAt": "Call API",
  "QueryLanguage": "JSONata",
  "States": {
    "Call API": {
      "Type": "Task",
      "Resource": "arn:<PARTITION>:states:::apigateway:invoke",
      "Arguments": {
        "ApiEndpoint": "<API_ENDPOINT>",
        "Method": "GET",
        "AuthType": "IAM_ROLE"
      },
      "Next": "Call Successful?"
    },
    "Call Successful?": {
      "Type": "Choice",
      "Choices": [
        {
          "Next": "Notify Success",
          "Condition": "{$states.input.StatusCode = 200}"
        }
      ],
      "Default": "Notify Failure"
    },
    "Notify Success": {
      "Type": "Task",
      "Resource": "arn:<PARTITION>:states:::sns:publish",
      "Arguments": {
        "Message": "Call was successful",
        "TopicArn": "arn:<PARTITION>:sns:<REGION>:<ACCOUNT_ID>:MySnsTopic"
      },
      "End": true
    },
    "Notify Failure": {
      "Type": "Task",
      "Resource": "arn:<PARTITION>:states:::sns:publish",
      "Arguments": {
        "Message": "Call was not successful",
        "TopicArn": "arn:<PARTITION>:sns:<REGION>:<ACCOUNT_ID>:MySnsTopic"
      },
      "End": true
    }
  }
}
```




<a name = "sns"></a>

## 2. Simple Notification Service (SNS)

Như tên gọi, đây là dịch vụ gửi thông báo trên AWS. SNS hoạt động theo mô hình **publish - subscribe**, đưa tin nhắn từ *publisher* (dịch vụ nguồn) đến *subscriber* (dịch vụ đích hoặc cá nhân người nhận) theo cơ chế **bất đồng bộ**. 

<p>
<image src="/assets/25_step_func/sns.png" alt="Simple Notification Service (SNS)" style="max-width:80%;height:auto;display:block;margin:0 auto;"/>
</p>

Cụ thể, *publisher* gửi tin nhắn đến một **chủ đề** (**_topic_**), SNS sẽ phân phối tin nhắn này đến tất cả subscriber đã đăng ký chủ đề đó. Các *subscriber* có thể là:
- Người nhận trực tiếp: qua tin nhắn SMS, email, hoặc thông báo trên thiết bị di động (*push notification*).
- Ứng dụng khác: qua giao thức HTTPS, hoặc các dịch vụ AWS như Lambda, [SQS](/2026/01/25/sqs/), Kinesis Data Firehose, v.v.
<!-- TODO: include link to Kinesis Data Firehose -->

SNS hay được sử dụng trong kiến trúc **Fanout**, khi một tin nhắn nhắn tới một SNS Topic được chuyển tiếp đến nhiều dịch vụ khác nhau để xử lý song song (bất đồng bộ). Ví dụ, có thể thiết kế một hệ thống thương mại điện tử, gửi tin nhắn đến một SNS Topic khi có đơn hàng mới, rồi chuyển tiếp đến các hàng đợi SQS khác nhau ứng với các tác vụ xử lý thanh toán, cập nhật kho hàng, gửi email xác nhận, v.v., cùng lúc. 
Bạn đọc có thể tìm hiểu chi tiết hơn trong [bài sau](/2026/01/25/sqs#fanout).

<a name = "reference"></a>

## Tài liệu tham khảo

1. [Giới thiệu về Step Function](https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html)
2. [Các Trạng thái trong Step Function](https://docs.aws.amazon.com/step-functions/latest/dg/workflow-states.html)
3. [Ngôn ngữ Mô tả Trạng thái (Amazon States Language)](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-amazon-states-language.html)
4. [Giới thiệu về Simple Notification Service (SNS)](https://docs.aws.amazon.com/sns/latest/dg/welcome.html)