---
layout: post
title: "23. Lambda Cơ bản"
title2: "Lambda Cơ bản"
date: 2026-01-18
permalink: /2026/01/18/lambda-fundamental
categories: [Lambda, Serverless]
tags: [Lambda, Serverless]
img: /assets/23_lambda/lambda-async.png
summary: "Lambda là dịch vụ serverless (không máy chủ) tiêu biểu của AWS. Khái niệm này không có nghĩa là không cần máy chủ vật lý nào để chạy ứng dụng, mà là người dùng không cần quan tâm đến việc quản lý, vận hành máy chủ. Như EC2, AWS chỉ cung cấp tài nguyên, người dùng vẫn phải tự quản lý hệ điều hành, phần mềm, bảo mật, v.v. Với Lambda, chỉ cần tập trung vào mã nguồn."
---


Lambda là dịch vụ **serverless** (*không máy chủ*) tiêu biểu của AWS. Khái niệm này không có nghĩa là không cần máy chủ vật lý nào để chạy ứng dụng, mà là người dùng không cần quan tâm đến việc quản lý, vận hành máy chủ. Như [EC2](/2025/12/16/ec2-fundamental), AWS chỉ cung cấp tài nguyên, người dùng vẫn phải tự quản lý hệ điều hành, phần mềm, bảo mật, v.v. Với Lambda, chỉ cần tập trung vào mã nguồn.


## Trong bài này:

- [1. Function](#function)
- [2. Môi trường Thực thi](#runtime-environment)
- [3. Kích hoạt](#trigger)
    - [3.1. Kích hoạt Đồng bộ](#synchronous-invocation)
    - [3.2. Kích hoạt Bất Đồng bộ](#asynchronous-invocation)
- [4. Quản lý Truy cập](#access-management)
    - [4.1. Cấp Quyền cho Lambda Function Truy cập các Dịch vụ Khác](#execution-role)
    - [4.2. Cấp Quyền cho các Danh tính Khác Kích hoạt Lambda Function](#resource-policy)
- [Tài liệu tham khảo](#reference)




Các khái niệm chính trong Lambda gồm:

<a name = "function"></a>

## 1. Function

Function là một đoạn mã nguồn, thực thi một tác vụ cụ thể khi được kích hoạt bằng một sự kiện (*event*). Ví dụ, một function được kích hoạt khi một tệp được tải lên S3.

**Function handler** là hàm chính trong function, được gọi ngay khi function được kích hoạt, nhận đầu vào là sự kiện và ngữ cảnh thực thi (*context*), trả về kết quả sau khi xử lý. Ví dụ:

```python
import json
def lambda_handler(event, context):
    # Lấy object key của tệp mới tải lên S3
    object_key = event['Records'][0]['s3']['object']['key']
    return {
        'statusCode': 200,
        'body': json.dumps(f'Uploaded object key: {object_key}')
    }
```

Đầu vào của handler gồm:

- `event`: chứa thông tin về sự kiện kích hoạt function, định dạng JSON. Ví dụ, sự kiện khi một tệp được tải lên S3 có `event` như sau:

```json
{
  "Records": [
    {
      "eventVersion": "2.0",
      "eventSource": "aws:s3",
      "awsRegion": "us-east-1",
      "eventTime": "2026-01-18T12:34:56.000Z",
      "eventName": "ObjectCreated:Put",
      "s3": {
        "bucket": {
          "name": "my-bucket"
        },
        "object": {
          "key": "my-file.txt",
          "size": 1234
        }
      }
    }
  ]
}
```

Ở đây mình chỉ liệt kê các trường quan trọng, event thực tế có nhiều trường hơn. Ta có thể truy cập các trường này trong handler để xử lý logic phù hợp, ví dụ lấy tên bucket và [object key](/2025/11/30/s3-fundamental##s3-object) của tệp mới tải lên để xử lý.


- `context`: chứa thông tin về [môi trường thực thi](#runtime-environment), như tên function, thời gian chạy còn lại, v.v. Đây là một đối tượng (trong lập trình hướng đối tượng, không phải S3 object), với các thuộc tính (*attribute*) và phương thức (*method*) để sử dụng nếu cần, ví dụ:

```python
def lambda_handler(event, context):
    function_name = context.function_name  
    memory_limit = context.memory_limit_in_mb
    log_group_name = context.log_group_name
    remaining_time = context.get_remaining_time_in_millis()
```

Để tạo function, có 3 cách:

- Tự viết mã nguồn trên giao diện Lambda.
- Tải lên mã nguồn dưới dạng [tệp nén ZIP](https://docs.aws.amazon.com/lambda/latest/dg/configuration-function-zip.html), cần đảm bảo cài đặt đủ các thư viện phụ thuộc.
- Sử dụng [Docker image](https://docs.aws.amazon.com/lambda/latest/dg/images-create.html).



<a name = "runtime-environment"></a>

## 2. Môi trường Thực thi 

**Runtime environment** là môi trường mà Lambda sử dụng để chạy function, bao gồm cấu hình (RAM), ngôn ngữ lập trình, các thư viện phụ thuộc, v.v. 

Lambda hỗ trợ nhiều ngôn ngữ phổ biến như Python, Node.js, Java, C#, Go, Ruby. 
Khi tạo function, ta chọn runtime phù hợp với ngôn ngữ đã viết mã nguồn. Ví dụ, nếu viết function bằng Python, ta chọn runtime là "Python 3.14" (hoặc các phiên bản Python khác trên giao diện).

Có thể cấu hình năng lực tính toán của môi trường thực thi bằng cách **chọn dung lượng bộ nhớ** cho function, từ 128 MB đến 10240 MB (10 GB). vCPU được cấp phát **gián tiếp**, tăng theo lượng RAM đã chọn. Ví dụ, 1769 MB bộ nhớ tương ứng với 1 vCPU, tại thời điểm viết bài.


**Timeout**: là thời gian tối đa một function có thể chạy. Mặc định là 3 giây, **tối đa 15 phút**. Nếu function không hoàn thành trong thời gian này, Lambda sẽ dừng và ghi nhận lỗi.

Về lưu trữ, Lambda cung cấp **512 MB** (tối đa 10 GB) dung lượng lưu trữ **tạm thời** (*ephemeral storage*) tại đường dẫn `/tmp` của môi trường thực thi. Dữ liệu trong thư mục này chỉ tồn tại trong một phiên thực thi function. Nếu function được gọi lại, môi trường mới sẽ không có dữ liệu từ lần gọi trước.


Có thể thay đổi các cấu hình của function trực tiếp trên giao diện, hoặc qua lệnh [`update-function-configuration`](https://docs.aws.amazon.com/cli/latest/reference/lambda/update-function-configuration.html) trên CLI như sau:

```sh
aws lambda update-function-configuration \
  --function-name awscoban-function \
  --memory-size 2048 \
  --timeout 30 \
  --ephemeral-storage '{"Size": 1024}'
```

<a name = "trigger"></a>

## 3. Kích hoạt

Sau khi viết mã nguồn và *deploy* function, ngoài việc để function tự động thực thi khi có sự kiện từ các dịch vụ khác, có thể chủ động kích hoạt function đó để thực thi mã nguồn theo (bằng CLI, SDK, trên giao diện Lambda, hoặc Function URL). Có hai loại kích hoạt: đồng bộ (**synchronous invocation**) và bất đồng bộ (**asynchronous invocation**).


<a name="synchronous-invocation"></a>

## 3.1. Kích hoạt Đồng bộ

<p>
<image src="/assets/23_lambda/lambda-sync.png" alt="Lambda Synchronous Invocation" style="max-width:50%;height:auto;display:block;margin:0 auto;"/>
</p>

Theo cách này, người gọi sẽ **chờ** function hoàn thành. Kết nối giữa người gọi và Lambda sẽ được giữ trong suốt quá trình thực thi function. Nếu function chạy thành công, kết quả sẽ được trả về. Nếu gặp lỗi, thông tin lỗi cũng sẽ được trả về.

Việc thử lại nếu gặp lỗi (*retry*) sẽ do người gọi quản lý.

Trên CLI, có thể kích hoạt function đồng bộ như sau:

```sh
aws lambda invoke \
  --function-name awscoban-function \
  --invocation-type RequestResponse \
  --cli-binary-format raw-in-base64-out \
  --payload '{"key1": "value1", "key2": "value2"}' \
  response.json
```

Trong đó, chỉ rõ `--invocation-type` của kích hoạt đồng bộ là `RequestResponse`, đây cũng là giá trị mặc định, nên có thể bỏ qua tham số này.
`--payload` là chuỗi chứa sự kiện đầu vào ở định dạng JSON. Kết quả trả về sẽ được lưu trong tệp `response.json`. 


<a name="asynchronous-invocation"></a>

## 3.2. Kích hoạt Bất Đồng bộ

<p>
<image src="/assets/23_lambda/lambda-async.png" alt="Lambda Asynchronous Invocation" style="max-width:80%;height:auto;display:block;margin:0 auto;"/>
</p>

Nhiều dịch vụ AWS như S3, SNS, Events Bridge kích hoạt Lambda Function theo cách này. Khi kích hoạt bất đồng bộ, người gọi sẽ **không chờ** function hoàn thành. Thay vào đó, quá trình diễn ra như sau:

1. Người dùng kích hoạt bất đồng bộ, Lambda nhận sự kiện đầu vào và xếp vào hàng đợi (*queue*).
2. Lambda lấy sự kiện từ hàng đợi (*polling*).
3. Lambda thực thi function với sự kiện đầu vào, **thử lại nếu gặp lỗi** từ 0 đến 2 lần tuỳ theo [cài đặt của người dùng](https://docs.aws.amazon.com/lambda/latest/dg/invocation-async-configuring.html).
4. Nếu function vẫn lỗi sau các lần thử lại, có thể cài đặt để gửi sự kiện đến Dead Letter Queue để phân tích sau. Nếu thành công, trả kết quả cho một dịch vụ đích (có thể khác dịch vụ nguồn) nếu người dùng có cấu hình.
<!-- TODO: include link to DLQ -->



Trong ví dụ ở mục trên, có thể thay đổi kích hoạt bất đồng bộ như sau:

```sh
aws lambda invoke \
  --function-name awscoban-function \
  --invocation-type Event \
  --cli-binary-format raw-in-base64-out \
  --payload '{"key1": "value1", "key2": "value2"}' \
  response.json
``` 

Trong đó, cần chỉ rõ `--invocation-type` của kích hoạt bất đồng bộ là `Event`.




Bạn đọc có thể tìm hiểu quá trình chi tiết hơn trong blog của [ByteByteGo](https://blog.bytebytego.com/p/aws-lambda-under-the-hood), tổng hợp nội dung kỹ thuật từ các sự kiện re:Invent của AWS.


<a name = "access-management"></a>

## 4. Quản lý Truy cập

<a name="execution-role"></a>

### 4.1. Cấp Quyền cho Lambda Function Truy cập các Dịch vụ Khác

Lambda Function thường cần truy cập các dịch vụ AWS khác để thực hiện tác vụ, ví dụ đọc/ghi dữ liệu từ S3, ghi log vào CloudWatch, v.v. Để làm điều này, cần cấp quyền cho function thông qua **Execution Role**. 

Đây là một [IAM Role](/2025/11/07/iam/#iam-role) được gán cho function. Mỗi function phải có một Excution Role, một Excution Role có thể gán cho nhiều function. Khi function được kích hoạt, Lambda sử dụng các quyền được định nghĩa trong [Permission Policy](/2025/11/07/iam/#permission-policy) của Execution Role để truy cập các dịch vụ khác.

Đối tượng được cấp quyền là Lambda, nên [Trust Policy](/2025/11/07/iam/#trust-policy) của Execution Role luôn có Principal là `lambda.amazonaws.com`:


```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```


Mặc định, khi tạo function mới trên giao diện Lambda, AWS sẽ tự động tạo Execution Role cơ sở, chỉ với quyền ghi log vào [CloudWatch Logs](/2026/01/05/cloudwatch#cloudwatch-logs), với Permission Policy như sau:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "logs:CreateLogGroup",
            "Resource": "arn:aws:logs:us-east-1:123456789012:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": [
                "arn:aws:logs:us-east-1:123456789012:log-group:/aws/lambda/<function-name>:*"
            ]
        }
    ]
}
```


Để function truy cập các dịch vụ khác, cần thêm các Permission Policy tương ứng.
Ví dụ, thêm policy sau để cấp quyền đọc object trong S3:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": "arn:aws:s3:::*"
        }
    ]
}
```


<a name = "resource-policy"></a>

### 4.2. Cấp Quyền cho các Danh tính Khác Kích hoạt Lambda Function

Để cho phép người dùng hoặc các dịch vụ AWS khác kích hoạt một function, cần gán [resource policy](/2025/11/07/iam/#resource-policy) cho function đó. 

Ví dụ, resource policy sau cho phép tài khoản AWS với ID `987654321098` và S3 kích hoạt function có tên `awscoban-function`:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::987654321098:root"
            },
            "Action": "lambda:InvokeFunction",
            "Resource": "arn:aws:lambda:us-east-1:123456789012:function:awscoban-function"
        },
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "s3.amazonaws.com"
            },
            "Action": "lambda:InvokeFunction",
            "Resource": "arn:aws:lambda:us-east-1:123456789012:function:awscoban-function"
        }
    ]
}
```




<a name = "reference"></a>

## Tài liệu tham khảo

1. [Cấu trúc và Khái niệm cơ bản trong Lambda](https://docs.aws.amazon.com/lambda/latest/dg/concepts-basics.html)
2. [AWS Lambda Under the Hood - ByteByteGo](https://blog.bytebytego.com/p/aws-lambda-under-the-hood)
3. [Kích hoạt Bất đồng bộ](https://docs.aws.amazon.com/lambda/latest/dg/invocation-async.html)


Đó là các khái niệm cơ bản, tiếp theo, hãy tìm hiểu một số tính năng nâng cao trong Lambda.