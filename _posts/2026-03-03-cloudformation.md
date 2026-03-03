---
layout: post
title: "36. CloudFormation"
title2: "CloudFormation"
date: 2026-03-03
permalink: /2026/03/03/cloudformation
categories: [IaC, CloudFormation, Infrastructure]
tags: [IaC, CloudFormation, Infrastructure]
img: /assets/36_cloudformation/cloudformation.png
summary: "Infrastructure as Code (IaC), như tên gọi, là phương pháp tạo và quản lý hạ tầng (máy chủ, mạng, cơ sở dữ liệu, v.v.) thông qua mã nguồn, thay vì tự làm thủ công. Qua đó ta có thể áp dụng các nguyên tắc quản lý phần mềm (quản lý phiên bản, kiểm thử, CI/CD) vào phần cứng, giúp việc vận hành đơn giản hơn. Trong AWS, dịch vụ IaC được cung cấp là CloudFormation."
---


Infrastructure as Code (IaC), như tên gọi, là phương pháp tạo và quản lý hạ tầng (máy chủ, mạng, cơ sở dữ liệu, v.v.) thông qua **mã nguồn**, thay vì tự làm thủ công. Qua đó ta có thể áp dụng các nguyên tắc quản lý phần mềm (quản lý phiên bản, kiểm thử, CI/CD) vào phần cứng, giúp việc vận hành đơn giản hơn. Trong AWS, dịch vụ IaC được cung cấp là **CloudFormation**.


## Trong bài này:

- [1. Khái niệm Cơ bản](#concepts)
- [2. Cấu trúc Template](#template)
    - [2.1. Resources](#resources)
    - [2.2. Intrinsic Function](#intrinsic-functions)
    - [2.3. Parameters & Rules](#parameters-rules)
    - [2.4. Outputs](#outputs)
    - [2.5. Mappings](#mappings)
    - [2.6. Conditions](#conditions)
- [Tài liệu tham khảo](#reference)


<a name = "concepts"></a>

## 1. Khái niệm Cơ bản

Hai khái niệm quan trọng nhất trong CloudFormation là **Template** và **Stack**.

- **Template**: đại diện cho **Code** trong IaC. Template là một tệp JSON hoặc YAML định nghĩa các tài nguyên AWS cần tạo và cấu hình.
- **Stack**: là phần **Infrastructure** trong IaC. Stack là đơn vị quản lý cơ bản trong CloudFormation, là tập hợp các tài nguyên AWS được tạo bởi một Template. 

<p>
<image src="/assets/36_cloudformation/cloudformation.png" alt="CloudFormation Overview" style="max-width:80%;height:auto;display:block;margin:0 auto;"/>
</p>


Ví dụ, dưới đây là một Template đơn giản, tạo một [EC2 Instance](/2025/12/16/ec2-fundamental) và gán Elastic IP (địa chỉ IP tĩnh) cho nó:

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: EC2 instance with Elastic IP
Resources:
  MyEC2Instance:
    Type: 'AWS::EC2::Instance'
    Properties:
      ImageId: ami-0ff8a91507f77f867
      InstanceType: t2.micro
      KeyName: awscoban-key
      BlockDeviceMappings:
        - DeviceName: /dev/sdm
          Ebs:
            VolumeType: io1
            Iops: 200
            DeleteOnTermination: false
            VolumeSize: 20
  MyEIP:
    Type: 'AWS::EC2::EIP'
    Properties:
      InstanceId: !Ref MyEC2Instance
```

Thông qua Template này, ta có thể dễ dàng cấu hình Instance theo ý muốn: [loại Instance](/2025/12/16/ec2-fundamental#instance-type), [AMI](/2026/01/02/ec2-config#ami), ổ [EBS](/2025/12/20/ebs), [EFS](/2025/12/24/efs), v.v., và sau đó tạo Instance chỉ với một thao tác "Create Stack", thay vì phải làm thủ công qua AWS Console hoặc CLI.

Khi "Create Stack" từ Template, CloudFormation tự động gọi các API tương ứng để tạo và cấu hình tài nguyên (tất nhiên danh tính tạo Stack cần có quyền tạo các tài nguyên đó, được cấp qua [IAM](/2025/11/07/iam)). Khi Stack bị xoá, tất cả tài nguyên sẽ bị xoá theo, trừ khi cấu hình [`DeletionPolicy`](#deletion-policy) để giữ lại.

Các thành phần trong Template sẽ được trình bày dưới đây.



<a name = "template"></a>

## 2. Cấu trúc Template

Một Template đầy đủ gồm các phần sau:

```yaml
---
AWSTemplateFormatVersion: 2010-09-09

Description:
    Mô tả ngắn gọn để hiểu nhanh về chức năng của Template.

Metadata:
    Thông tin bổ sung về Template.

Parameters:
    Các tham số đầu vào của Template.

Rules:
    Các quy tắc để kiểm tra tính hợp lệ của tham số đầu vào.

Mappings:
    Một bảng tra cứu các giá trị.

Conditions:
    Các điều kiện để kiểm soát việc tạo tài nguyên.

Transform:
    Các biến đổi để áp dụng cho Template.

Resources:
    Các tài nguyên cần tạo, là phần bắt buộc.

Outputs:
    Các giá trị đầu ra của Template
    Có thể được sử dụng bởi các stack khác hoặc hiển thị cho người dùng.
```

Phần bắt buộc duy nhất là `Resources`, nơi định nghĩa các tài nguyên AWS cần tạo. Các phần khác là tùy chọn.


<a name = "resources"></a>

### 2.1. Resources

Phần `Resources` có cấu trúc như sau:

```yaml
Resources:
  YourLogicalName:
    Type: AWS::EC2::Instance 
    Properties: 
      Key1: Value1
      Key2: Value2
```

Trong đó định nghĩa danh sách các tài nguyên cần tạo, mỗi tài nguyên có:
- `YourLogicalName`: tên định danh duy nhất trong Template, dùng để tham chiếu đến tài nguyên này trong các phần khác (ví dụ `!Ref LogicalName`). Lưu ý, đây không phải từ khoá, hãy thay thế trực tiếp giá trị này, và nên đặt tên dễ nhận biết.
- `Type`: loại tài nguyên AWS cần tạo (ví dụ `AWS::EC2::Instance`).
- `Properties`: các thuộc tính cấu hình của tài nguyên, tùy thuộc vào loại tài nguyên.


Ví dụ, dưới đây ta tạo 3 tài nguyên: một VPC, một subnet trong VPC và một EC2 Instance trong subnet:

```yaml
Resources:
  VPC:
    Type: 'AWS::EC2::VPC'
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: awscoban-vpc

  SubnetA:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC # trỏ đến VPC sẽ chứa Subnet
      CidrBlock: 10.0.0.0/24
      Tags:
        - Key: Name
          Value: subnet-a
    DependsOn: VPC # chờ VPC được tạo xong mới tạo Subnet

  WebServer:
    Type: 'AWS::EC2::Instance'
    Properties:
      ImageId: ami-0ff8a91507f77f867
      InstanceType: t2.micro
      SubnetId: !Ref SubnetA # trỏ đến Subnet sẽ chứa Instance
    DependsOn: SubnetA # chờ Subnet được tạo xong mới tạo Instance
```

Ngoài hai trường `Type` và `Properties`, còn có một số trường bổ sung như:

<a name = "depends-on"></a>

#### 2.1.1. DependsOn

Chỉ định thứ tự tạo tài nguyên. CloudFormation thường sẽ tạo nhiều tài nguyên cùng lúc nếu có thể.
Nếu có phụ thuộc, ví dụ Subnet phải được tạo sau VPC, thường CloudFormation cũng có thể xử lý để đảm bảo thứ tự. 
Tuy nhiên tốt nhất ta vẫn nên chỉ định rõ ràng buộc bằng `DependsOn`.

Cú pháp rất đơn giản: chỉ cần liệt LogicalName của các tài nguyên cần tạo trước. Trong ví dụ trên, SubnetA sẽ được tạo sau VPC, và WebServer sẽ được tạo sau SubnetA.

<a name = "deletion-policy"></a>

#### 2.1.2. DeletionPolicy
Chỉ định hành động đối với tài nguyên khi Stack bị xoá:
- `Delete`: xoá tài nguyên (mặc định).
- `Retain`: giữ lại. Thường dùng cho các tài nguyên chứa dữ liệu, như [RDS](/2026/01/08/rds-fundamental), [S3](/2025/11/30/s3-fundamental), v.v., để tránh mất dữ liệu khi Stack bị xoá.
- `Snapshot`: tạo snapshot trước khi xoá (áp dụng cho một số tài nguyên như RDS, EBS).


<a name = "intrinsic-functions"></a>

### 2.2. Intrinsic Function

Trở lại ví dụ trong phần trên, có một số giá trị được tham chiếu đến các tài nguyên khác trong Template.
Ví dụ, trong `SubnetA`, ta cần chỉ rõ VPC chứa nó qua trường `VpcId`. Trong hầu hết trường hợp, ta không biết ID của VPC được tạo cùng (ví dụ `vpc-12345678`), do đó bắt buộc phải có cách để tham chiếu đến nó. Và như đã thấy, ta sử dụng **hàm nội tại (_intrinsic function_)** `!Ref`.

`!Ref` trả về giá trị của tài nguyên  được tham chiếu, thường là ID của tài nguyên đó. Ngoài ra, một hàm nội tại khác hay dùng là `!GetAtt`, trả về một **thuộc tính cụ thể** của tài nguyên. Ví dụ, `!GetAtt VPC.CidrBlock` sẽ trả về [CIDR](/2025/11/13/vpc#cidr) của VPC. Đây là cách để **kết nối các tài nguyên với nhau** trong Template!


Ngoài ra, CloudFormation còn cung cấp nhiều hàm nội tại khác:

- **!Sub**: thay thế biến trong chuỗi.

Ví dụ: `!Sub "AMI ID: ${WebServer.ImageId}"` sẽ thay `${WebServer.ImageId}` bằng giá trị AMI ID của EC2 Instance. 

- **!Join**: nối các chuỗi lại với nhau. 

Cú pháp: `!Join [delimiter, [list_of_values]]`. Chuỗi `delimiter` sẽ được chèn giữa các phần tử trong `list_of_values`.

Ví dụ: `!Join ["-", ["awscoban", "vpc", "001"]]` sẽ chèn `-` vào giữa các phần tử, trả về `awscoban-vpc-001`.

- **!Split**: ngược lại với `!Join`, dùng để tách chuỗi thành một danh sách.

Cú pháp: `!Split [delimiter, string]`.  `delimiter` được dùng để xác định điểm tách chuỗi.

Ví dụ: `!Split ["-", "awscoban-vpc-001"]` sẽ tìm các chuỗi nằm giữa các dấu `-`, tức sẽ trả về `["awscoban", "vpc", "001"]`.

- **!Select**: chọn một phần tử từ một danh sách. 
Cú pháp: `!Select [index, list]`. Chọn phần tử tại vị trí `index` (bắt đầu từ 0) trong `list`.

Ví dụ: `!Select [1, ["awscoban", "vpc", "001"] ]` sẽ trả về phần tử có index 1, là `vpc`.

Bạn đọc có thể tham khảo danh sách đầy đủ các hàm nội tại trong [tài liệu AWS](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference.html).



<a name = "parameters-rules"></a>

### 2.3. Parameters & Rules

Có thể truyền tham số đầu vào khi tạo Stack để giúp Template linh hoạt hơn. Ví dụ, có thể tạo và dùng một tham số để người dùng chọn loại EC2 Instance khi tạo Stack. Ví dụ:

```yaml
Parameters:
  InstanceType:
    Description: Chọn loại Instance
    Type: String
    Default: t2.micro
    AllowedValues:
      - t2.micro
      - t2.small
      - t2.medium
```

Trong đó, `AllowedValues` giúp giới hạn giá trị tham số, và nếu người dùng nhập giá trị không hợp lệ, CloudFormation sẽ trả về lỗi. Với các điều kiện phức tạp hơn, có thể sử dụng `Rules` để định nghĩa các quy tắc kiểm tra tính hợp lệ của tham số đầu vào. Ví dụ:

```yaml
Rules:
  CheckInstanceType:
    Assertions:
      - Assert: !Or
          - !Equals [!Ref InstanceType, t2.micro]
          - !Equals [!Ref InstanceType, t2.small]
          - !Equals [!Ref InstanceType, t2.medium]
        AssertDescription: "Instance type must be t2.micro, t2.small, or t2.medium."
```

Tham số có thể được tham chiếu trong Template thông qua hàm nội tại `!Ref`.


**Pseudo Parameter**: là các tham số đặc biệt do CloudFormation cung cấp sẵn, như `AWS::Region` trả về Region hiện tại, `AWS::AccountId` trả về ID tài khoản AWS, v.v. Chúng rất hữu ích để tăng tính linh hoạt của Template để tái sử dụng trên nhiều môi trường khác nhau. Danh sách tất cả các pseudo parameter [tại đây](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/pseudo-parameter-reference.html).


<a name = "outputs"></a>

### 2.4. Outputs

Parameters là đầu vào, còn Outputs là đầu ra của Template. Outputs cho phép xuất ra các giá trị sau khi Stack được tạo thành công, để hiển thị cho người dùng hoặc để sử dụng trong các Stack khác hoặc công cụ khác. Ví dụ:

```yaml
Outputs:
  InstanceId:
    Description: ID của EC2 Instance được tạo
    Value: !Ref WebServer
    Export:
        Name: WebServerInstanceId # cho phép Stack khác tham chiếu đến giá trị này qua !ImportValue WebServerInstanceId
  VpcCidr:
    Description: CIDR của VPC được tạo
    Value: !GetAtt VPC.CidrBlock
```

Các trường chính:
- `Value`: giá trị đầu ra, có thể là một chuỗi tĩnh, thuộc tính của tài nguyên, hoặc tham số.
- `Export`: nếu muốn cho phép Stack khác sử dụng đầu ra này, cần đặt giá trị `Name` trong trường `Export`. Đây là cách để tạo [cross-stack reference](/2026/03/12/cnf-stack-advanced#cross-stack-reference) giữa các Stack, giúp chia hệ thống thành nhiều module nhỏ để dễ quản lý.
<!-- TODO: include link to cross stack reference-->


<a name = "mappings"></a>

### 2.5. Mappings   

Là một bảng tra cứu các giá trị, sử dụng để lấy giá trị tuỳ theo điều kiện nhất định. Ví dụ, mapping dưới đây liệt kê các AMI ID theo Region:

```yaml
Mappings:
  RegionToAMI:
    us-east-1:
      AMI1: ami-12345678901234567
      AMI2: ami-23456789012345678
    us-west-1:
      AMI1: ami-34567890123456789
      AMI2: ami-45678901234567890
    eu-west-1:
      AMI1: ami-56789012345678901
      AMI2: ami-67890123456789012
```

Vì [AMI](/2026/01/02/ec2-config#ami) được gắn với một Region, nên nếu muốn triển khai một Template trên nhiều Region, cần sử dụng mapping để định nghĩa.

Để lấy giá trị từ mapping, sử dụng hàm nội tại `!FindInMap`. Ví dụ:

```yaml
Resources:
  MyEC2Instance:
    Type: 'AWS::EC2::Instance'
    Properties:
      ImageId: !FindInMap [RegionToAMI, !Ref "AWS::Region", AMI1] 
      # tra cứu key `!Ref "AWS::Region"` trong mapping `RegionToAMI`, rồi tra cứu tiếp key `AMI1`
      InstanceType: t2.micro
```

Khi Stack được tạo, `!FindInMap` sẽ tìm `AMI1` trong Region hiện tại (được xác định bởi pseudo parameter `AWS::Region`). Đây là cách để cấu hình tài nguyên dựa trên môi trường mà không cần phải viết nhiều Template khác nhau cho từng môi trường.



<a name = "conditions"></a>

### 2.6. Conditions

Là các điều kiện đúng/sai để kiểm soát việc tạo tài nguyên. Trường hợp hay dùng là khi muốn tạo tài nguyên trong môi trường nhất định (thường là Production). 
Ví dụ, trong Template dưới đây, ta dùng tham số `EnvType` để chọn môi trường, và tạo Conditions dựa vào môi trường, để cấp phát EC2 Instance kích thước lớn hơn cho Production:

```yaml
Parameters:
  EnvType:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - prod

Conditions:
    IsProduction: !Equals [!Ref EnvType, prod]

Resources:
  ProdInstance:
    Type: 'AWS::EC2::Instance'
    Condition: IsProduction # chỉ tạo nếu `IsProduction` đúng (tức EnvType là `prod`)
    Properties:
        InstanceType: t2.xlarge
  DevInstance:
    Type: 'AWS::EC2::Instance'
    Condition: !Not [IsProduction] # chỉ tạo nếu `IsProduction` sai (tức EnvType không phải `prod`)
    Properties:
        InstanceType: t2.micro
```

Ngoài tài nguyên, có thể gán Condition cho Outputs, để chỉ đưa ra Output khi điều kiện đúng. Ví dụ:

```yaml
Outputs:
  ProdInstanceId:
    Condition: IsProduction
    Value: !Ref ProdInstance
```





<a name = "reference"></a>

## Tài liệu tham khảo

1. [Cách Hoạt động của CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-overview.html)
2. [Các Section trong Template](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Template-anatomy.html)
3. [Các Hàm Nội tại trong CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference.html)
4. [Pseudo Parameters](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/pseudo-parameter-reference.html)
5. [Conditions](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/conditions-section-structure.html)
6. [CloudFormation Best Practices](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/best-practices.html)

Bài tiếp theo, ta sẽ tìm hiểu các đoạn mã bổ trợ (*helper script*) của CloudFormation.
