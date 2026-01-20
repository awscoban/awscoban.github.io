---
layout: post
title: "27. API Gateway"
title2: "API Gateway"
date: 2026-01-28 00:00:00 +0700
permalink: /2026/01/28/api-gateway
categories: [Serverless]
tags: [Serverless]
img: /assets/27_api_gateway/api-gateway.png
summary: "API Gateway (tổng quát, không chỉ của AWS) là cổng vào duy nhất cho người dùng/ứng dụng bên ngoài truy cập vào backend, giúp xác thực, điều tiết truy cập (throttling), quản lý API và phân luồng yêu cầu. Trên AWS, Amazon API Gateway tất nhiên cũng cung cấp các tính năng tương tự, ngoài ra còn tích hợp chặt chẽ với các dịch vụ AWS khác, giúp việc xây dựng ứng dụng dễ dàng hơn."
---

API Gateway (tổng quát, không chỉ của AWS) là cổng vào duy nhất cho người dùng/ứng dụng bên ngoài truy cập vào backend, giúp xác thực, điều tiết truy cập (*throttling*), quản lý API và phân luồng yêu cầu. Trên AWS, Amazon API Gateway tất nhiên cũng cung cấp các tính năng tương tự, ngoài ra còn tích hợp chặt chẽ với các dịch vụ AWS khác, giúp việc xây dựng ứng dụng dễ dàng hơn. 




## Trong bài này:

- [1. Tổng quan](#overview)
- [2. Method](#method)
- [3. Integration](#integration)
- [4. Cache](#cache)
- [Tài liệu tham khảo](#reference)




<a name = "overview"></a>

## 1. Tổng quan

Trên AWS, API Gateway được cung cấp sẵn, tính khả dụng cao, tự động mở rộng theo nhu cầu sử dụng. Backend phía sau có thể là các dịch vụ AWS như Lambda, EC2, ECS, hoặc on-premises. API Gateway hỗ trợ cả HTTP, REST API (*stateless*) và WebSocket (*stateful*).


Các khái niệm:

- **API Endpoint**: Địa chỉ URL để gọi API, cấu trúc thường là:
```
https://{api-id}.execute-api.{region}.amazonaws.com/{stage}/{resource}
```
Ví dụ: 
```
https://awscoban101.execute-api.us-east-1.amazonaws.com/prod/posts/1
```
- **Stage**: Môi trường triển khai của API (DEV, TEST, PROD), giúp quản lý các phiên bản khác nhau của API.
- **Resource**: Là các đường dẫn (*path*) trong API, trỏ đến tài nguyên Backend mà API cung cấp, ví dụ `/posts/{postId}`, `/users/{userId}`.
- **Method**: các hành động người dùng có thể thực hiện khi gọi API để truy cập tài nguyên. Ta sẽ tìm hiểu kỹ hơn trong phần dưới.


<p>
<image src="/assets/27_api_gateway/api-gateway.png" alt="API Gateway" style="max-width:90%;height:auto;display:block;margin:0 auto;"/>
</p>


<a name = "method"></a>

## 2. Method

Dưới góc nhìn của client, họ tương tác với API Gateway khi gọi API. Yêu cầu gửi từ client tới API Gateway được gọi là **Method Request**, và kết quả trả về từ API Gateway được gọi là **Method Response**. 


Method Request thực chất là một [HTTP request](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods), gồm phương thức (GET, POST, PUT, DELETE, v.v.), đường dẫn (*path*), tiêu đề (*header*), tham số truy vấn (*query parameter*) và nội dung yêu cầu (request body) tương ứng với từng loại phương thức. 
Ví dụ:

```
GET /posts/1
Host: apigateway.us-east-1.amazonaws.com
```


Tương tự, Method Response là một HTTP response, bao gồm trạng thái (status code), tiêu đề (header) và nội dung phản hồi (response body).

```
200 OK 
Content-Type: application/json
...

{
    "id": "1",
    "title": "AWS Cơ Bản"
}
```


<a name = "integration"></a>

## 3. Integration

API Gateway chuyển tiếp yêu cầu từ client đến backend để xử lý, và nhận kết quả trả về từ backend để gửi lại cho client. Trong API Gateway, khái niệm **Backend** được gọi là **Integration**. Yêu cầu từ API Gateway tới backend là **Integration Request**, và kết quả trả về từ backend là **Integration Response**.

**Các loại Integration** API Gateway hỗ trợ gồm:

- `AWS`: tích hợp với các dịch vụ AWS như Lambda. Cần cấu hình Integration Request và Integration Response, cũng như cách biến đổi dữ liệu từ Method Request sang Integration Request, và từ Integration Response sang Method Response, gọi là [**mapping template**](#mapping-template).
- `AWS_PROXY`: thường chỉ áp dụng cho Lambda, gọi là *Lambda Proxy Integration*. Khi sử dụng cấu hình này, không cần Integration Request và Integration Response, API Gateway đơn giản chỉ chuyển tiếp Method Request từ client đến Lambda, và trả kết quả từ Lambda về client. Đó là lý do cho tên gọi "*proxy*".
- `HTTP`: tích hợp với các endpoint HTTP bên ngoài (on-premise hoặc bên thứ ba). Tương tự `AWS`, cần cấu hình Integration Request và Integration Response, cũng như cách biến đổi dữ liệu.
- `HTTP_PROXY`: tích hợp với các endpoint HTTP bên ngoài. Tương tự `AWS_PROXY`, không cần cấu hình Integration Request và Integration Response.
- `MOCK`: dùng để kiểm thử API, không tích hợp với backend thực tế, API Gateway tự tạo phản hồi giả lập.


<a name = "mapping-template"></a>

**Mapping Template**: là các mẫu định dạng dữ liệu dùng để chuyển đổi dữ liệu giữa Method và Integration trong các Integration không phải là `proxy`. Mapping Template sử dụng ngôn ngữ [Velocity Template Language (VTL)](https://velocity.apache.org/engine/1.7/user-guide.html) để định nghĩa cách chuyển đổi dữ liệu.


Ví dụ, giả sử trong Method Request, client gửi dữ liệu JSON như sau:

```json
{
    "postId": "1",
    "title": "AWS Cơ Bản"
}
```

Và backend yêu cầu dữ liệu ở định dạng XML như sau:

```xml
<Post>
    <Id>1</Id>
    <Title>AWS Cơ Bản</Title>
</Post>
``` 

Ta có thể sử dụng Mapping Template sau để chuyển đổi dữ liệu từ JSON sang XML trong Integration Request:

```
#set($inputRoot = $input.path('$'))
<Post>
    <Id>$inputRoot.postId</Id>
    <Title>$inputRoot.title</Title>
</Post>
``` 



<a name = "cache"></a>

## 4. Cache

API Gateway hỗ trợ bộ nhớ đệm (*cache*) cho các [stage](#overview).
Khi bật cache, API Gateway lưu trữ kết quả trả về từ backend trong một khoảng thời gian xác định (TTL - Time To Live, mặc định 300 giây, tối đa 3600 giây). Khi có yêu cầu mới đến, nếu dữ liệu có trong cache và chưa hết hạn, API Gateway sẽ trả về dữ liệu từ cache thay vì gọi lại backend. Việc này giúp giảm trễ và tải cho backend khi có nhiều yêu cầu giống nhau. 



<a name = "reference"></a>

## Tài liệu tham khảo

1. [Giới thiệu về API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html)
2. [Các Khái niệm cơ bản trong API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-basic-concept.html)
3. [Các loại Integration](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-integration-types.html)
4. [Mapping Template](https://docs.aws.amazon.com/apigateway/latest/developerguide/models-mappings.html)