---
layout: post
title: "24. Lambda Nâng cao"
title2: "Lambda Nâng cao"
date: 2026-01-22
permalink: /2026/01/22/lambda-advanced
categories: [Lambda, Serverless]
tags: [Lambda, Serverless]
img: /assets/24_lambda_adv/lambda-concurrency.png
summary: "Bài này trình bày một số khái niệm nâng cao trong Lambda, bao gồm chu kỳ của môi trường thực thi, tối ưu thời gian kích hoạt function với warm-start, thực thi song song, quản lý phiên bản (version) và alias của một function."
---

Bài này trình bày một số khái niệm nâng cao trong Lambda, bao gồm chu kỳ của môi trường thực thi, tối ưu thời gian kích hoạt function với warm-start, thực thi song song, quản lý phiên bản (version) và alias của một function.


## Trong bài này:

- [1. Chu kỳ của Môi trường Thực thi](#lifecycle)
    - [1.1. Khởi tạo (Init)](#init-phase)
    - [1.2. Kích hoạt (Invoke)](#invoke-phase)
    - [1.3. Chấm dứt (Shutdown)](#shutdown-phase)
- [2. Concurrency](#concurrency)
    - [Đặt trước Concurrency](#reserved-concurrency)
- [3. Version](#version)
- [4. Alias](#alias)
- [Tài liệu tham khảo](#reference)


<a name = "lifecycle"></a>

## 1. Chu kỳ của Môi trường Thực thi


Khi một Lambda function được kích hoạt, Lambda tạo một môi trường thực thi (*execution environment*) để chạy mã nguồn của function đó. Vòng đời của một môi trường thực thi bao gồm các giai đoạn sau:

<p>
<image src="/assets/24_lambda_adv/lambda_lifecycle.png" alt="Lambda Life Cycle" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>

<a name = "init-phase"></a>

### 1.1. Khởi tạo (Init)

Trong giai đoạn này, Lambda thực hiện 3 nhiệm vụ:

- **Khởi tạo các phần mở rộng (*extension*)**. Đây là các tiến trình được tích hợp sẵn để quản lý, theo dõi, giám sát môi trường thực thi. Trong hầu hết trường hợp, người dùng không cần quan tâm đến các phần mở rộng này. 
- **Khởi tạo runtime**: Lambda tải runtime (trình biên dịch hoặc thông dịch) ứng với ngôn ngữ lập trình được chọn.
- **Khởi tạo function**: chạy các đoạn mã **bên ngoài** hàm chính (*handler*), thường bao gồm tải các thư viện, khởi tạo biến toàn cục, kết nối đến cơ sở dữ liệu, v.v.

2 quá trình đầu tiên gọi là **khởi động lạnh** (_**cold start**_), sẽ làm việc kích hoạt function lần đầu trễ một chút (khoảng 0.1-1 giây), diễn ra khi function được kích hoạt sau một khoảng thời gian không hoạt động, hoặc khi có lưu lượng truy cập tăng đột biến, cần thêm nhiều Instance để xử lý.

Quá trình cuối cùng trong giai đoạn này là khởi tạo function, các biến toàn cục sẽ được tái sử dụng cho các lần gọi function tiếp theo (nếu môi trường thực thi vẫn đang hoạt động). Điều này giúp giảm thời gian khởi tạo cho các lần gọi sau, được gọi là **khởi động ấm** (_**warm start**_).

Ví dụ, trong mã nguồn Python sau, ta khởi tạo kết nối CSDL ở bên ngoài `lambda_handler`. Kết nối này sẽ được tái sử dụng cho các lần kích hoạt function (gọi `lambda_handler`) tiếp theo.

```python
#### FUNCTION INIT STARTS ####
import json
import os
import psycopg2

db_connection = psycopg2.connect(
    host=os.getenv("DB_HOST"),
    database=os.getenv("DB_NAME"),
    user=os.getenv("DB_USER"),
    password=os.getenv("DB_PASSWORD")
)
#### FUNCTION INIT ENDS ####

def lambda_handler(event, context):
    # Sử dụng kết nối CSDL đã khởi tạo cho các lần kích hoạt function
    cursor = db_connection.cursor()
    cursor.execute("SELECT version();")
    db_version = cursor.fetchone()
    
    return {
        'statusCode': 200,
    }
```

<a name = "invoke-phase"></a>

### 1.2. Kích hoạt (Invoke)

Giai đoạn tiếp theo là kích hoạt (*Invoke*), trong đó Lambda gọi hàm chính của function để xử lý sự kiện đầu vào.
[Timeout](/2025/01/18/lambda-fundamental#time-out) của function bản chất sẽ giới hạn thời gian của giai đoạn này. Hàm chính và tất cả phần mở rộng phải hoàn thành trong thời gian này.

Nếu có lỗi xảy ra trong giai đoạn kích hoạt, Lambda [chấm dứt](#shutdown-phase) rồi lại khởi tạo lại môi trường thực thi (như đã đề cập phía trên).


<a name = "shutdown-phase"></a>

### 1.3. Chấm dứt (Shutdown)

Trong giai đoạn này, Lambda chấm dứt môi trường thực thi và giải phóng tài nguyên. Thường thì sau khi kích hoạt function xong, môi trường thực thi vẫn được giữ lại để tái sử dụng cho các lần kích hoạt tiếp theo, giúp giảm thời gian khởi tạo. Tuy nhiên, Lambda vẫn sẽ chấm dứt môi trường sau mỗi vài giờ để cập nhật và bảo trì, dù function vẫn được kích hoạt đều đặn



<a name = "concurrency"></a>

## 2. Concurrency

Concurrency (tạm dịch: *đồng thời*) là số lượng yêu cầu mà một Lambda function có thể xử lý cùng lúc. Khi có yêu cầu đến, nếu không có môi trường thực thi nào đang sẵn sàng, Lambda khởi tạo một môi trường mới để xử lý yêu cầu đó, như đề cập trong phần trên.


<p>
<image src="/assets/24_lambda_adv/lambda-concurrency.png" alt="Lambda Concurrency" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>

Diễn giải:

- Tại thời điểm `t1`, yêu cầu 1 đến, kích hoạt function. Lúc này chưa có môi trường thực thi nào, Lambda khởi tạo môi trường `Env A` để xử lý. Do đó, có giai đoạn cold start (*init*). Lúc này, `concurrency = 1`. 
- Tại thời điểm `t2`, yêu cầu 2 đến trong khi `Env A` vẫn đang xử lý yêu cầu 1. Lambda khởi tạo thêm môi trường `Env B` cho yêu cầu này. Cũng có giai đoạn cold start (*init*). Lúc này, `concurrency = 2`, do đang xử lý 2 yêu cầu cùng lúc.
- Tại thời điểm `t3`, yêu  3 đến, 2 yêu cầu trước đó vẫn chưa hoàn thành, Lambda khởi tạo thêm môi trường `Env C`. Lúc này, `concurrency = 3`.
- Tại thời điểm `t4`, yêu cầu 1 hoàn thành, yêu cầu 2, 3 chưa xử lý xong. `Env A` đang sẵn sàng. Yêu cầu 4 đến, Lambda sử dụng lại `Env A` để xử lý, không cần khởi tạo môi trường mới (*warm-start*). Lúc này, `concurrency = 3`, do vẫn đang xử lý song song 3 yêu cầu.
- Tương tự, tại thời điểm `t5`, yêu cầu 2 hoàn thành, yêu cầu 3, 4 chưa xong. Yêu cầu 5 đến, Lambda sử dụng lại `Env B` để xử lý. Lúc này, `concurrency = 3`.
- Tại thời điểm `t6`, yêu cầu 3, 4, 5 chưa hoàn thành. Yêu cầu 6 đến, Lambda khởi tạo thêm môi trường `Env D`. Lúc này, `concurrency = 4`.
- Tại thời điểm `t7`, tất cả yêu cầu đã hoàn thành, không có yêu cầu mới. Lúc này, `concurrency = 0`.


Công thức tính concurrency cho một function:

```
Concurrency = (Số yêu cầu mỗi giây) * (Thời gian xử lý mỗi yêu cầu)
```

Ví dụ, một function xử lý 100 yêu cầu mỗi giây, trung bình mỗi yêu cầu mất 200 ms (0.2 giây) để xử lý, thì concurrency là \\(100 * 0.2 = 20\\).
Tức tại một thời điểm, function này cần trung bình 20 môi trường thực thi để xử lý đồng thời tất cả yêu cầu.


<a name = "reserved-concurrency"></a>

### Đặt trước Concurrency

Mỗi tài khoản AWS có giới hạn concurrency mặc định là 1000 cho **tất cả** Lambda function trong một Region. Nếu tổng concurrency của tất cả function vượt quá giới hạn này, sẽ gặp lỗi `Throttling`.


Nếu, với một function nào đó quan trọng, và muốn ưu tiên cấp phát tài nguyên cho nó, ta có thể cố định concurrency cho function đó (**Reserved Concurrency**), miễn vẫn để lại 100 concurrency để chia cho các function khác.

Ví dụ, giả sử ta có 2 function quan trọng A và B, đặt trước 400 concurrency cho A, 400 cho B. Khi đó, còn lại 200 concurrency sẽ được sử dụng chung cho các function khác. Nếu tổng concurrency của các function khác vượt quá 200, lỗi `Throttling` sẽ xảy ra.


<a name = "version"></a>

## 3. Version

Lambda hỗ trợ tạo và quản lý nhiều phiên bản (*version*) khác nhau của một function. Mỗi phiên bản là một bản sao độc lập của function, với mã nguồn và cấu hình riêng biệt. Ví dụ, ta có thể tạo các phiên bản cho các môi trường development, staging, production tương ứng, dễ dàng quản lý và triển khai.

Trên giao diện Lambda, khi thay đổi mã nguồn hoặc cấu hình của function, ta đang thay đổi một phiên bản đặc biệt gọi là `$LATEST`. Đây là phiên bản mặc định, luôn trỏ tới mã nguồn mới nhất, và đặc biệt là **có thể thay đổi**.

Ngược lại, tại một thời điểm bất kỳ, ta có thể tạo một phiên bản cố định từ `$LATEST` (*publish new version* trên giao diện). Phiên bản sau khi được tạo sẽ  **cố định, không thể thay đổi**. Phiên bản này bao gồm mã nguồn, cấu hình, thư viện, các biến môi trường, v.v., của function tại thời điểm tạo phiên bản. Số hiệu phiên bản được tự động tăng dần, bắt đầu từ 1.


Giả sử ta có một function tên là `awscb-func`, mỗi phiên bản của function này sẽ được định danh bởi một [ARN](/2025/11/07/iam#arn) duy nhất, trỏ đến phiên bản đó, với số hiệu phiên bản được thêm vào cuối ARN. Ví dụ:

- ARN của function, trỏ tới mã nguồn mới nhất (chưa publish):
```
arn:aws:lambda:us-east-1:123456789012:function:awscb-func
```

- ARN của phiên bản `$LATEST` (tác dụng giống như ARN của function):
```
arn:aws:lambda:us-east-1:123456789012:function:awscb-func:$LATEST
```

- ARN của phiên bản 1:
```
arn:aws:lambda:us-east-1:123456789012:function:awscb-func:1
``` 

- ARN của phiên bản 2:
```
arn:aws:lambda:us-east-1:123456789012:function:awscb-func:2
```


<a name = "alias"></a>

## 4. Alias

Alias là một **con trỏ** trỏ tới một phiên bản cụ thể của function. Mỗi alias được định danh bởi một [ARN](/2025/11/07/iam#arn) duy nhất, với tên alias được thêm vào cuối ARN. Tuy nhiên, alias **có thể được cập nhật** để trỏ tới phiên bản khác khi cần.

Ví dụ, ta có thể tạo alias `PROD` trỏ tới phiên bản 1 của function `awscb-func`. Khi đó, 2 ARN sau sẽ tương đương:

```
arn:aws:lambda:us-east-1:123456789012:function:awscb-func:PROD
arn:aws:lambda:us-east-1:123456789012:function:awscb-func:1
```

Ngoài việc gán một alias cho một phiên bản cụ thể, ta có thể sử dụng **alias trọng số** (**_weighted alias_**) để phân phối lưu lượng truy cập giữa **tối đa hai** phiên bản khác nhau.

Ví dụ, trên CLI, ta có thể tạo alias `PROD` trỏ tới phiên bản 1 với trọng số 90%, và phiên bản 2 với trọng số 10% như sau:

```bash
aws lambda create-alias \
    --function-name awscb-func \
    --name PROD \
    --function-version 1 \
    --routing-config '{"AdditionalVersionWeights": {"2": 0.1}}'
```




<a name = "reference"></a>

## Tài liệu tham khảo

1. [Chu kỳ của Môi trường Thực thi](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html)
1. [Lambda Concurrency](https://docs.aws.amazon.com/lambda/latest/dg/lambda-concurrency.html)
2. [Cấu hình Phiên bản (Version)](https://docs.aws.amazon.com/lambda/latest/dg/configuration-versions.html)
3. [Alias](https://docs.aws.amazon.com/lambda/latest/dg/configuration-aliases.html)


Qua hai bài vừa rồi, mình đã trình bày các khái niệm cần thiết trong Lambda. Tiếp theo, hãy tìm hiểu về Step Function, một dịch vụ tạo luồng công việc (*workflow*) cho các ứng dụng serverless phức tạp hơn.