---
layout: post
title: "52. Các Dịch vụ Học Máy"
title2: "Các Dịch vụ Học Máy"
date: 2026-06-06
permalink: /2026/06/06/ml-services
categories: [Machine Learning]
tags: [Machine Learning]
img: 
summary: "Bài này giới thiệu sơ lược các dịch vụ Học Máy trên AWS. Ở đây mình sẽ không đi sâu vào cách sử dụng từng dịch vụ, mà chỉ giới thiệu chức năng của chúng. Trong khuôn khổ các đề thi chứng chỉ, bạn đọc chỉ cần nhớ chức năng của các dịch vụ là đủ."
---

Bài này giới thiệu sơ lược các dịch vụ Học Máy trên AWS. Ở đây mình sẽ không đi sâu vào cách sử dụng từng dịch vụ, mà chỉ giới thiệu chức năng của chúng. Trong khuôn khổ các đề thi chứng chỉ, bạn đọc chỉ cần nhớ chức năng của các dịch vụ là đủ.



## Trong bài này:

- [1. Rekognition - Nhận diện Hình ảnh & Video](#rekognition)
- [2. Textract - OCR](#textract)
- [3. Comprehend - Xử lý Ngôn ngữ Tự nhiên (NLP)](#comprehend)
- [4. Lex - Giao diện Hội thoại](#lex)
- [5. Polly - Text-to-Speech](#polly)
- [6. Transcribe - Speech-to-Text](#transcribe)
- [7. Kendra - Truy hồi Thông tin](#kendra)
- [8. SageMaker - Nền tảng Học Máy Toàn diện](#sagemaker)
- [Tài liệu tham khảo](#reference)


<a name = "rekognition"></a>

## 1. Rekognition - Nhận diện Hình ảnh & Video

Như tên gọi chuẩn chính tả là **Recognition** (*nhận diện*), dịch vụ này có khả năng phân tích **hình ảnh** và **video** để **nhận diện đối tượng, khuôn mặt**, nội dung, v.v. Nó có thể được sử dụng cho các ứng dụng như giám sát an ninh, phân loại hình ảnh, và tìm kiếm nội dung trong video. Rekognition thậm chí có thể xử lý video theo thời gian thực với [Kinesis Video Streams](/2026/01/31/kinesis-glue#kinesis-video-streams).


<a name = "textract"></a>

## 2. Textract - OCR

OCR (*Optical Character Recognition*) là công nghệ nhận diện ký tự trong hình ảnh, Textract là dịch vụ OCR của AWS. Khác với các công cụ OCR open-source công khai, v.v. Textract không chỉ nhận diện văn bản mà còn có khả năng hiểu cấu trúc của tài liệu (đoạn văn, bảng, biểu đồ), giúp trích xuất thông tin với độ chính xác rất cao.

Các ứng dụng có thể kể đến như số hóa tài liệu, trích xuất dữ liệu từ hóa đơn, hợp đồng, hoặc các biểu mẫu. 

**Mẹo ghi nhớ**: `Textract` có thể được nhớ là `Text Extract` (*trích xuất văn bản*), giúp dễ dàng liên tưởng đến chức năng chính của dịch vụ này.


<a name = "comprehend"></a>

## 3. Comprehend - Xử lý Ngôn ngữ Tự nhiên (NLP)

Comprehend (nghĩa là **hiểu, nhận thức***) là dịch vụ NLP của AWS, giúp phân tích **văn bản**, nhằm phân loại, nhận diện thực thể, phân tích cảm xúc, và trích xuất thông tin quan trọng từ văn bản.


<a name = "lex"></a>

## 4. Lex - Giao diện Hội thoại

Lex giúp là nền tảng hỗ trợ **chatbot** và **giao diện hội thoại**, giúp xây dựng các ứng dụng **tương tác qua văn bản hoặc giọng nói**. Đây là nền móng xây dựng Alexa, trợ lý ảo của Amazon. 

Lex thường được sử dụng để xây dựng chatbot hỗ, trợ lý ảo, hoặc các ứng dụng cần tương tác với người dùng.


<a name = "polly"></a>

## 5. Polly - Text-to-Speech

Polly là dịch vụ **chuyển đổi văn bản thành giọng nói**, hỗ trợ nhiều ngôn ngữ và giọng nói khác nhau, cho phép tạo ra các trải nghiệm tương tác tự nhiên hơn.

Polly cung cấp API để tích hợp vào các ứng dụng, đầu vào văn bản, đầu ra là audio stream, có thể dùng trực tiếp để phát âm thanh hoặc lưu lại dưới dạng MP3. Ví dụ:

```python
from boto3 import client
polly = client("polly", region_name="us-east-1")
response = polly.synthesize_speech(
        Text="Hello from AWS Co Ban.",
        OutputFormat="mp3",
        VoiceId="Matthew")
```


<a name = "transcribe"></a>

## 6. Transcribe - Speech-to-Text

Ngược lại với Polly, Transcribe là dịch vụ **chuyển đổi giọng nói thành văn bản**. Ứng dụng phổ biến gồm tạo phụ đề cho video, chuyển các cuộc gọi thoại thành văn bản để phân tích, hoặc xây dựng các ứng dụng nhận diện giọng nói.


<a name = "kendra"></a>

## 7. Kendra - Truy hồi Thông tin

Kendra là dịch vụ **truy hồi thông tin** (**information retrieval**) thông minh, không chỉ giới hạn ở tìm kiếm theo từ khoá, mà có khả năng hiểu ngữ cảnh và ngữ nghĩa (semantic search). Tương tự như **Elastic Search**, nhưng tích hợp tốt hơn vào hệ sinh thái AWS, với các nguồn dữ liệu được hỗ trợ sẵn như [RDS](/2026/01/08/rds-fundamental), [S3](/2025/11/30/s3-fundamental), [DynamoDB](/2026/03/18/dynamodb), v.v. Ngoài ra, dữ liệu bên ngoài (Google Drive, SharePoint, , OneDrive) cũng có thể được sử dụng.


<a name = "sagemaker"></a>

## 8. SageMaker - Nền tảng Học Máy Toàn diện

Khác với các dịch vụ trên (vốn là các mô hình đóng gói sẵn), SageMaker là một môi trường hoàn chỉnh để **phát triển, huấn luyện, và triển khai các mô hình học máy tùy chỉnh**. Nó cung cấp các công cụ và dịch vụ hỗ trợ toàn bộ quy trình học máy, từ việc chuẩn bị dữ liệu, xây dựng mô hình, huấn luyện, đến triển khai và giám sát mô hình trong môi trường production. 




<a name = "reference"></a>

## Tài liệu tham khảo

1. [Rekognition](https://docs.aws.amazon.com/rekognition/latest/dg/what-is.html)
2. [Textract](https://docs.aws.amazon.com/textract/latest/dg/text-location.html)
3. [Comprehend](https://docs.aws.amazon.com/comprehend/latest/dg/what-is.html)
4. [Lex](https://docs.aws.amazon.com/lexv2/latest/dg/what-is.html)
5. [Polly](https://aws.amazon.com/polly/getting-started/)
6. [Transcribe](https://docs.aws.amazon.com/transcribe/latest/dg/how-input.html)
7. [Kendra](https://docs.aws.amazon.com/kendra/latest/dg/how-it-works.html)
8. [SageMaker](https://docs.aws.amazon.com/sagemaker/latest/dg/how-it-works-mlconcepts.html)