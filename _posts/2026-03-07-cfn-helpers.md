---
layout: post
title: "37. Helper Script trong CloudFormation"
title2: "Helper Script trong CloudFormation"
date: 2026-03-07
permalink: /2026/03/07/cfn-helper-scripts
categories: [IaC, CloudFormation, Infrastructure]
tags: [IaC, CloudFormation, Infrastructure]
img: /assets/37_cfn_helpers/cfn-hup.png
summary: "Với riêng EC2 Instance và Auto Scaling Group, CloudFormation cung cấp một số đoạn mã bổ trợ (helper script) viết bằng Python để hỗ trợ cài đặt và cấu hình Instance, gồm: cfn-init, cfn-signal, cfn-hup, và cfn-get-metadata."
---


Với riêng [EC2 Instance](/2025/12/16/ec2-fundamental) và [Auto Scaling Group](/2026/02/02/ec2-autoscaling), CloudFormation cung cấp một số đoạn mã bổ trợ (*helper script*) viết bằng Python để hỗ trợ cài đặt và cấu hình Instance, gồm:

- `cfn-init`
- `cfn-signal`
- `cfn-hup`
- `cfn-get-metadata`

Các đoạn mã này được cài đặt sẵn trên AMI chạy Amazon Linux, còn với các hệ điều hành khác, chỉ cần tải xuống gói `aws-cfn-bootstrap`. 
Cùng tìm hiểu chi tiết hơn dưới đây.




## Trong bài này:

- [1. `cfn-init`](#cfn-init)
- [2. `cfn-signal`](#cfn-signal)
- [3. `cfn-hup`](#cfn-hup)
- [4. `cfn-get-metadata`](#cfn-get-metadata)
- [5. Tài liệu tham khảo](#reference)


<a name = "cfn-init"></a>

## 1. `cfn-init`

Dùng để cài đặt phần mềm, chạy lệnh, và thực hiện các tác vụ cấu hình khác trên EC2 Instance khi Stack được tạo hoặc cập nhật. 

Nghe rất quen, phải không? Đây cũng là chức năng của [EC2 User Data](/2026/01/02/ec2-config#user-data). Điểm **khác biệt** là User Data liệt kê **tác vụ cần làm**, còn `cfn-init` chỉ rõ **trạng thái mong muốn** của Instance, và CloudFormation sẽ tự động thực hiện các tác vụ cần thiết để đạt trạng thái đó.

Ví dụ, giả sử ta muốn cài đặt Apache web server trên EC2 Instance khi khởi tạo. Nếu dùng User Data, ta cần viết Template như sau:


```yaml
Resources:
  EC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref LatestAmiId
      InstanceType: !Ref InstanceType
      UserData: !Base64             # Chạy khi Instance khởi động
        Fn::Sub: |
          #!/bin/bash
          yum update -y
          yum install -y httpd
          systemctl start httpd
```

Khi Instance khởi động (sau khi Stack được tạo), các đoạn mã trong User Data sẽ được thực thi, như đã giới thiệu trong bài [Các Cấu hình EC2 Hữu ích](/2026/01/02/ec2-config#user-data). Ở đây có hai [hàm nội tại](/2026/03/03/cloudformation#intrinsic-functions):
- `!Base64` (dạng đầy đủ là `Fn::Base64`): mã hoá User Data thành base64, là định dạng bắt buộc.
- `Fn::Sub`: để chèn biến vào chuỗi. Ở đây không có biến nào, tuy nhiên đây là một thói quen tốt nên áp dụng.

Cách này phù hợp nếu chỉ cần thực thi một vài lệnh đơn giản. Nhưng với các đoạn mã dài, việc quản lý và theo dõi lỗi sẽ khó khăn hơn nhiều.


Còn nếu dùng `cfn-init`, Template sẽ trông như sau:

```yaml
Resources:
  WebServerInstance:
    Type: AWS::EC2::Instance
    Metadata:
      AWS::CloudFormation::Init:    # Định nghĩa trạng thái mong muốn của Instance
        config:                     # Cấu hình 4 trường: `packages`, `files`, `commands`, `services`.
          packages:                 # Các package cần cài đặt
            yum:
              httpd: []
          groups:                   # Các nhóm người dùng cần tạo nếu cần. Ở đây để trống.
          users:                    # Các người dùng cần tạo nếu cần. Ở đây để trống
          sources:                  # Các file cần tải về nếu cần. Ở đây để trống.
          files:                    # Tạo các file cần thiết nếu muốn. Ở đây để trống.
          commands:                 # Các lệnh cần chạy nếu cần. Ở đây để trống vì `services` đã đảm nhiệm việc khởi động dịch vụ.
          services:                 # Các dịch vụ cần quản lý
            systemd:                # `systemd` sẽ khởi động `httpd`, và đảm bảo nó luôn chạy. 
              httpd:
                enabled: true
                ensureRunning: true
    Properties:
      ImageId: !Ref LatestAmiId
      InstanceType: !Ref InstanceType
      UserData: !Base64             
        Fn::Sub: |
          #!/bin/bash
          yum install -y aws-cfn-bootstrap             
          /opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource WebServerInstance --region ${AWS::Region}
```

Ở đây, ta định nghĩa trạng thái mong muốn của Instance trong phần `Metadata` -> `AWS::CloudFormation::Init`. Sau đó trong `UserData`:
- Cài đặt `aws-cfn-bootstrap` chứa mã nguồn `cfn-init` (nếu hệ điều hành không phải Amazon Linux)
- Gọi `cfn-init` để thực thi các tác vụ cần thiết nhằm đạt trạng thái mong muốn đã định nghĩa.


Đây là cách tiếp cận tốt hơn nhiều so với User Data, vì:
- Template rõ ràng, dễ đọc và bảo trì khi hệ thống phức tạp.
- Tích hợp tốt với `cfn-signal` để theo dõi trạng thái, phát hiện lỗi nhanh chóng và dễ dàng hơn. Với User Data, mặc định CloudFormation không biết các đoạn mã trong User Data có chạy thành công hay không. 



<a name = "cfn-signal"></a>

## 2. `cfn-signal`

Mặc định, với EC2, ngay khi CloudFormation khởi tạo xong Instance, nó sẽ coi tài nguyên này được tạo thành công (`CREATE_COMPLETE`). Tuy nhiên, lúc đó quá trình cài đặt phần mềm (dù là User Data hay `cfn-init`) có thể vẫn chưa xong, và ứng dụng trên thực tế vẫn chưa sẵn sàng. Nếu có lỗi xảy ra trong lúc cài đặt, CloudFormation sẽ không biết.


Giải pháp là dùng `cfn-signal`. Khi đó, CloudFormation tạm treo trạng thái EC2 Instance ở trạng thái `CREATE_IN_PROGRESS`, và đợi tín hiệu từ `cfn-signal`.
- Nếu là tín hiệu **thành công**, trạng thái chuyển sang `CREATE_COMPLETE`.
- Nếu là tín hiệu **lỗi**, trạng thái chuyển sang `CREATE_FAILED`, CloudFormation xóa tài nguyên đã tạo.
- Nếu **không nhận được tín hiệu nào** sau một khoảng thời gian nhất định (**timeout**), trạng thái cũng chuyển sang `CREATE_FAILED`.


Để cấu hình `cfn-signal`, ta thêm `CreationPolicy` vào tài nguyên EC2 Instance như sau:

```yaml
Resources:
  EC2Instance:
    Type: AWS::EC2::Instance
    CreationPolicy:
      ResourceSignal:
        Timeout: PT10M  # Đợi tối đa 10 phút
        Count: 1       # Đợi ít nhất 1 tín hiệu thành công
    Metadata: 
      AWS::CloudFormation::Init:
      # ... (như trên)
```

Sau đó, trong `UserData`, sau khi gọi `cfn-init`, ta thêm lệnh gọi `cfn-signal` để gửi tín hiệu về CloudFormation:

```yaml
UserData: !Base64
  Fn::Sub: |
    #!/bin/bash
    yum install -y aws-cfn-bootstrap
    /opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource WebServerInstance --region ${AWS::Region}
    /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource EC2Instance --region ${AWS::Region}
```

Trong lệnh `cfn-signal`, `-e $?` sẽ gửi mã lỗi của lệnh trước đó (tức `cfn-init`) về CloudFormation. Nếu `cfn-init` chạy thành công, mã lỗi là 0, tức tín hiệu thành công. 


<a name = "cfn-hup"></a>

## 3. `cfn-hup`

Lệnh `cfn-init` được đặt trong `UserData`, tức sẽ chỉ chạy **một lần** khi Instance khởi động. Tức nếu sau này ta cập nhật phần `Metadata` -> `AWS::CloudFormation::Init` để thay đổi trạng thái mong muốn, `cfn-init` sẽ **không chạy lại** để áp dụng các thay đổi này.

Vấn đề này có thể giải quyết bằng `cfn-hup`. Đây là một tiến trình nền (*daemon*) chạy liên tục, theo dõi các thay đổi trong `Metadata` của tài nguyên, và chạy các tác vụ phản ứng cần thiết (gọi là **hook**) khi có thay đổi.

<p>
<image src="/assets/37_cfn_helpers/cfn-hup.png" alt="cfn-hup" style="max-width:80%;height:auto;display:block;margin:0 auto;"/>
</p>


Để cài đặt `cfn-hup`, cần 2 file cấu hình sau (định dạng `toml`):

- `/etc/cfn/cfn-hup.conf`: xác định Stack và tài nguyên cần theo dõi.
```toml
[main]
stack=${AWS::StackId}
region=${AWS::Region}
interval=15 # Kiểm tra mỗi 15 phút
```

- `/etc/cfn/hooks.d/cfn-auto-reloader.conf`: định nghĩa các hook sẽ chạy khi có thay đổi. Ví dụ, hook dưới đây sẽ chạy lại `cfn-init`:
```toml
[cfn-auto-reloader-hook]
triggers=post.update        # Trigger khi metadata đang theo dõi được cập nhật
path=Resources.EC2Instance.Metadata.AWS::CloudFormation::Init       # Metadata cần theo dõi
action=/opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource EC2Instance --region ${AWS::Region}   
```

Giờ ta tích hợp 2 tệp này vào Template, bằng cách thêm chúng vào phần `files` của trong `AWS::CloudFormation::Init`, và đảm bảo `cfn-hup` luôn chạy trong phần `services`:

```yaml
Metadata:
  AWS::CloudFormation::Init:
    config:
      files:
        # 1. Tạo file cfn-hup.conf
        "/etc/cfn/cfn-hup.conf":
          content: !Sub |
            [main]
            stack=${AWS::StackId}
            region=${AWS::Region}
          mode: "000400"
          owner: "root"
          group: "root"
        
        # 2. Tạo file cấu hình Hook
        "/etc/cfn/hooks.d/cfn-auto-reloader.conf":
          content: !Sub |
            [cfn-auto-reloader-hook]
            triggers=post.update
            path=Resources.EC2Instance.Metadata.AWS::CloudFormation::Init
            action=/opt/aws/bin/cfn-init -v --stack ${AWS::StackId} --resource EC2Instance --region ${AWS::Region}
          mode: "000400"
          owner: "root"
          group: "root"

      services:
        sysvinit:
          # Để đảm bảo `cfn-hup` luôn chạy
          cfn-hup:
            enabled: "true"
            ensureRunning: "true"
            files:
              - "/etc/cfn/cfn-hup.conf"
              - "/etc/cfn/hooks.d/cfn-auto-reloader.conf"
```


## 4. `cfn-get-metadata`

Đoạn mã này giúp truy xuất Metadata của tài nguyên trong Template, lấy ra để xử lý hoặc cấu hình thêm nếu cần. 

Ví dụ, giả sử ta có Resource với Metadata như sau:

```yaml
WebServerInstance:
  Type: AWS::EC2::Instance
  Metadata:
    MyConfig:
      Version: "1.2.3"
      Environment: "Production"
```

Bên trong EC2 Instance đó (hoặc một Instance khác cùng Stack), dùng `cfn-get-metadata` để truy xuất Metadata này như sau:

```sh
/opt/aws/bin/cfn-get-metadata -s { "Ref" : "AWS::StackName" } -r WebServerInstance --region { "Ref" : "AWS::Region" }
```

Kết quả:
```json
{
  "MyConfig": {
    "Version": "1.2.3",
    "Environment": "Production"
  }
}
```

Việc này hữu ích khi muốn cấu hình ứng dụng bên trong Instance dựa trên thông tin từ Template, ví dụ như lấy một vài giá trị để đặt biến môi trường.


Có thể truy xuất theo khoá mong muốn, bạn đọc có thể tham khảo thêm [tài liệu chính thức](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/cfn-get-metadata.html).



<a name = "reference"></a>

## Tài liệu tham khảo

1. [CloudFormation Helper Script](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/cfn-helper-scripts-reference.html)
2. [Bootstrap EC2 với `cfn-init`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/deploying.applications.html)
3. [`cfn-signal`](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/cfn-signal.html)
4. [`cfn-hup`](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/cfn-hup.html)
5. [`cfn-get-metadata`](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/cfn-get-metadata.html)

Tiếp theo, hãy tìm hiểu các khái niệm nâng cao trong CloudFormation, như Cross-Stack Reference, Nested Stack, StackSet, Change Set.