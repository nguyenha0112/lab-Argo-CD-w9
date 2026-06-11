# Tài Liệu Vấn Đáp Terraform & Kubernetes

## 1. Terraform Là Gì?

Terraform là công cụ Infrastructure as Code dùng để khai báo hạ tầng bằng code. Mình mô tả trạng thái mong muốn trong file `.tf`, Terraform sẽ tính toán những thay đổi cần thực hiện để đưa hạ tầng thật về trạng thái đó.

Ý chính khi trả lời vấn đáp:

- Terraform dùng ngôn ngữ HCL.
- Terraform có tính declarative: khai báo "muốn gì", không phải viết từng bước "làm như thế nào".
- Terraform dùng provider để giao tiếp với AWS, Azure, GCP, Kubernetes, GitHub...
- Terraform dùng state để ghi nhớ resource nào trong code tương ứng với resource nào ngoài cloud.
- Workflow cơ bản: `init` -> `fmt` -> `validate` -> `plan` -> `apply`.

Ví dụ:

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "my-app-dev-logs"
}
```

Trả lời ngắn:

> Terraform giúp quản lý hạ tầng bằng code, có thể version bằng Git, review bằng pull request, lặp lại được giữa các môi trường và giảm cấu hình thủ công trên console.

## 2. Terraform Workflow

### `terraform init`

Khởi tạo working directory:

- Tải provider.
- Tải module.
- Cấu hình backend.
- Tạo thư mục `.terraform`.

### `terraform fmt`

Format code HCL cho đúng chuẩn.

### `terraform validate`

Kiểm tra cú pháp và tính hợp lệ cơ bản của config.

### `terraform plan`

Tạo execution plan. Terraform so sánh config, state và hạ tầng thật để cho biết sẽ tạo, sửa, xóa resource nào.

### `terraform apply`

Thực thi plan và cập nhật state.

### `terraform destroy`

Xóa resource đang được Terraform quản lý trong state.

Câu trả lời mẫu:

> Trước khi apply nên chạy plan để review thay đổi. Plan giúp phát hiện việc xóa/sửa nhầm resource, nhất là trong production.

## 3. Terraform State

State là file Terraform dùng để lưu mapping giữa code và hạ tầng thật. Mặc định là `terraform.tfstate` ở local.

State dùng để:

- Biết resource trong code tương ứng với resource nào trên cloud.
- Lưu thuộc tính resource.
- Tính dependency.
- So sánh config với hạ tầng thật khi chạy plan.

Rủi ro của local state:

- Dễ mất file.
- Dễ conflict khi làm team.
- Có thể chứa secret.
- Không có locking mặc định cho collaboration.

Production nên dùng remote state, ví dụ S3 backend, HCP Terraform, Azure Storage, GCS.

## 4. Remote Backend Và Locking

Remote backend lưu state ở nơi tập trung. Locking ngăn nhiều người hoặc nhiều pipeline cùng `apply` một lúc.

Ví dụ S3 backend:

```hcl
terraform {
  backend "s3" {
    bucket       = "my-terraform-state-bucket"
    key          = "w8/dev/terraform.tfstate"
    region       = "ap-southeast-1"
    use_lockfile = true
  }
}
```

Best practices:

- Bật versioning cho S3 bucket.
- Bật encryption.
- Giới hạn IAM access.
- Không commit state vào Git.
- Mỗi environment nên có state key riêng, ví dụ `dev/network/terraform.tfstate`.

Lưu ý khi vấn đáp:

> Trước đây hay dùng S3 + DynamoDB để locking. Theo tài liệu Terraform mới, S3 backend có `use_lockfile`; DynamoDB locking đã bị deprecate, nhưng vẫn có thể gặp trong project cũ.

## 5. Nếu `terraform apply` Bị Lỗi Giữa Chừng Thì Recover Như Thế Nào?

Đây là câu hỏi quan trọng. Terraform apply không phải lúc nào cũng atomic. Nghĩa là nếu apply bị lỗi giữa chừng, có resource đã tạo thành công, có resource chưa tạo, có resource đang tạo dở dang.

### Nguyên tắc đầu tiên

Không chạy `destroy` với tâm lý "làm lại từ đầu" nếu chưa hiểu tình trạng hiện tại. Việc cần làm là kiểm tra state, kiểm tra cloud thật, rồi chạy lại plan.

### Các bước recover an toàn

1. Đọc error message

Xem resource nào bị fail, lý do gì:

- Permission/IAM thiếu quyền.
- Quota vượt giới hạn.
- Tên resource bị trùng.
- Network/API timeout.
- Dependency chưa sẵn sàng.
- Config sai.

2. Kiểm tra state hiện tại

```bash
terraform state list
```

Lệnh này cho biết resource nào đã được ghi vào state.

Có thể xem chi tiết:

```bash
terraform state show aws_instance.web
```

3. Chạy lại `terraform plan`

```bash
terraform plan
```

Terraform sẽ so sánh config, state và cloud thật. Nếu một resource chưa tạo, plan sẽ hiện `create`. Nếu resource đã tạo và đã vào state, Terraform sẽ không tạo lại.

4. Sửa nguyên nhân lỗi

Ví dụ:

- Thêm IAM permission.
- Đổi tên bucket S3 bị trùng.
- Tăng quota.
- Sửa dependency.
- Sửa biến sai.

5. Chạy lại `terraform apply`

```bash
terraform apply
```

Thông thường Terraform sẽ tiếp tục tạo các resource còn thiếu.

### Nếu resource đã tạo trên cloud nhưng chưa vào state

Tình huống này có thể xảy ra nếu API tạo resource thành công nhưng Terraform bị crash/timeout trước khi ghi state.

Dấu hiệu:

- Chạy lại apply báo resource đã tồn tại.
- Cloud console thấy resource đã có.
- `terraform state list` không thấy resource đó.

Cách recover:

1. Viết config `.tf` đúng với resource đó.
2. Import resource vào state.

```bash
terraform import aws_s3_bucket.logs my-existing-bucket
```

3. Chạy:

```bash
terraform plan
```

Nếu plan còn đòi sửa nhiều, điều chỉnh code cho khớp resource thật.

### Nếu resource bị tạo lỗi hoặc đang dở

Ví dụ EC2 tạo thành công nhưng app chưa lên, Load Balancer chưa attach đủ target, RDS đang creating.

Cách xử lý:

- Chờ resource ổn định rồi chạy lại plan.
- Kiểm tra console/log của cloud provider.
- Nếu resource không nằm trong state và không dùng được, xóa thủ công resource đó sau khi chắc chắn nó không cần nữa, rồi apply lại.
- Nếu resource nằm trong state nhưng bị hỏng, có thể dùng `terraform apply -replace=...`.

Ví dụ replace:

```bash
terraform apply -replace=aws_instance.web
```

### Câu trả lời vấn đáp mẫu

> Nếu `terraform apply` lỗi giữa chừng, em không destroy ngay. Em đọc error, kiểm tra `terraform state list`, sau đó chạy lại `terraform plan` để Terraform tính lại resource nào đã tạo, resource nào còn thiếu. Nếu resource đã tạo trên cloud nhưng chưa có trong state, em import vào state bằng `terraform import`, rồi plan lại. Sau khi sửa nguyên nhân lỗi như IAM, quota, naming conflict hoặc dependency, em apply lại. Terraform sẽ tiếp tục từ trạng thái hiện tại để đưa hạ tầng về desired state.

## 6. HCL Blocks Hay Gặp

### Provider

```hcl
provider "aws" {
  region = var.aws_region
}
```

Provider là plugin giúp Terraform gọi API của nền tảng bên ngoài.

### Resource

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
```

Resource là đối tượng Terraform quản lý vòng đời: create, update, delete.

### Data Source

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
}
```

Data source chỉ đọc thông tin resource có sẵn, không sở hữu vòng đời của nó.

### Variable

```hcl
variable "env" {
  type    = string
  default = "dev"
}
```

### Output

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}
```

### Local

```hcl
locals {
  common_tags = {
    Env = var.env
  }
}
```

## 7. Module

Module là một nhóm Terraform config có thể tái sử dụng. Root module là thư mục mình chạy Terraform. Child module là module được gọi bằng block `module`.

Ví dụ:

```hcl
module "vpc" {
  source = "./modules/vpc"

  cidr_block = "10.0.0.0/16"
  env        = "dev"
}
```

Module nên có:

```text
modules/vpc/
  main.tf
  variables.tf
  outputs.tf
  versions.tf
```

Câu trả lời mẫu:

> Module giúp tái sử dụng và chuẩn hóa cấu hình. Em tách module khi một nhóm resource có boundary rõ ràng, được dùng lặp lại, ví dụ VPC, S3 bucket, EKS cluster. Không nên tách module quá sớm cho logic quá đơn giản.

## 8. Dependency Trong Terraform

Terraform tự suy luận dependency qua reference:

```hcl
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}
```

Subnet phụ thuộc VPC vì tham chiếu `aws_vpc.main.id`.

Dùng `depends_on` khi dependency không nằm trong attribute:

```hcl
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  depends_on = [aws_iam_role_policy.app]
}
```

Nguyên tắc:

- Ưu tiên dependency ngầm định qua reference.
- Chỉ dùng `depends_on` khi cần.

## 9. `count`, `for_each`, Lifecycle

### `count`

Dùng khi cần tạo N resource giống nhau:

```hcl
resource "aws_instance" "web" {
  count         = 2
  ami           = var.ami_id
  instance_type = "t3.micro"
}
```

### `for_each`

Dùng khi mỗi resource có key riêng:

```hcl
resource "aws_s3_bucket" "bucket" {
  for_each = toset(["logs", "assets"])

  bucket = "my-app-${each.key}"
}
```

Trả lời nhanh:

> `count` dựa vào index nên có thể bị xáo trộn khi list thay đổi. `for_each` dựa vào key nên ổn định hơn cho resource có danh tính riêng.

### Lifecycle

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  lifecycle {
    prevent_destroy = true
  }
}
```

Hay gặp:

- `prevent_destroy`: chặn xóa nhầm.
- `create_before_destroy`: tạo mới trước khi xóa cũ.
- `ignore_changes`: bỏ qua thay đổi của một số field.

## 10. Drift Và Import

Drift là khi resource thật bị sửa ngoài Terraform, ví dụ sửa trên AWS Console.

Phát hiện drift:

```bash
terraform plan
```

Xử lý:

- Nếu thay đổi thủ công là đúng, cập nhật code Terraform.
- Nếu thay đổi thủ công là sai, apply lại để đưa về desired state.

Import dùng khi resource đã tồn tại ngoài cloud và muốn đưa vào Terraform state:

```bash
terraform import aws_s3_bucket.logs my-existing-bucket
```

Import chỉ cập nhật state, vẫn phải viết config `.tf` tương ứng.

## 11. Kubernetes Overview

Kubernetes là container orchestration platform. Nó quản lý việc deploy, scale, expose và self-heal containerized application.

Thành phần hay hỏi:

- Pod: đơn vị nhỏ nhất để chạy container.
- ReplicaSet: đảm bảo số lượng Pod replica mong muốn.
- Deployment: quản lý rollout/rollback và tạo ReplicaSet.
- Service: endpoint ổn định để truy cập Pod.
- ConfigMap/Secret: cấu hình và dữ liệu nhạy cảm.
- Ingress: expose HTTP/HTTPS vào cluster.
- Namespace: chia logical environment trong cluster.

## 12. Pod Là Gì?

Pod là đơn vị deploy nhỏ nhất trong Kubernetes. Một Pod có thể chứa một hoặc nhiều container, chia sẻ network namespace và storage volume.

Đặc điểm:

- Mỗi Pod có IP riêng trong cluster.
- Pod có tính ephemeral, có thể bị xóa và tạo lại.
- Không nên truy cập Pod trực tiếp bằng IP vì IP thay đổi.
- Thường truy cập Pod thông qua Service.

Trả lời mẫu:

> Pod là đơn vị nhỏ nhất trong Kubernetes để chạy container. Pod có IP riêng nhưng không ổn định, vì vậy cần Service để cung cấp endpoint ổn định.

## 13. Deployment, ReplicaSet Và Pod Liên Quan Như Thế Nào?

Mối quan hệ:

```text
Deployment -> ReplicaSet -> Pod
```

Deployment là cấp cao nhất trong ba thành phần này:

- Deployment khai báo desired state của app, ví dụ image nào, cần mấy replica.
- Deployment tạo và quản lý ReplicaSet.
- ReplicaSet đảm bảo đúng số lượng Pod đang chạy.
- Pod là nơi container thực sự chạy.

Ví dụ:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
```

Giải thích:

- Deployment `nginx` muốn có 3 replica.
- Kubernetes tạo ReplicaSet.
- ReplicaSet tạo 3 Pod từ `template`.
- Nếu 1 Pod chết, ReplicaSet tạo Pod mới để quay lại đủ 3.
- Nếu update image, Deployment tạo ReplicaSet mới và rollout dần dần.

Lệnh xem quan hệ:

```bash
kubectl get deployment
kubectl get replicaset
kubectl get pod
```

Câu trả lời vấn đáp mẫu:

> Deployment quản lý rollout và desired state của ứng dụng. Deployment tạo ReplicaSet. ReplicaSet đảm bảo số lượng Pod đúng như khai báo. Pod là đơn vị chạy container thật. Khi update image, Deployment tạo ReplicaSet mới; ReplicaSet mới tạo Pod mới, ReplicaSet cũ giảm Pod dần dần.

## 14. Service Là Gì?

Service cung cấp endpoint ổn định để truy cập một nhóm Pod. Service dùng label selector để chọn Pod backend.

Vì sao cần Service:

- Pod IP thay đổi khi Pod restart.
- Nhiều Pod cần load balancing nội bộ.
- Client cần một DNS/name ổn định.

Ví dụ:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

Service trên chọn các Pod có label `app: nginx`.

## 15. ClusterIP Và LoadBalancer Khác Nhau?

`ClusterIP` và `LoadBalancer` đều là Service type trong Kubernetes, nhưng mục đích expose khác nhau.

### ClusterIP

ClusterIP là Service mặc định. Nó tạo một IP nội bộ chỉ truy cập được bên trong cluster.

Dùng khi:

- Service chỉ cần app trong cluster gọi nhau.
- Ví dụ backend gọi database, frontend gọi backend nội bộ.
- Không cần public ra internet.

Ví dụ:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

Trả lời ngắn:

> ClusterIP expose service bên trong cluster. Bên ngoài cluster không truy cập trực tiếp được.

### LoadBalancer

LoadBalancer expose Service ra bên ngoài cluster thông qua external load balancer của cloud provider.

Dùng khi:

- Cần public app ra internet hoặc network bên ngoài.
- Chạy trên cloud có integration với Load Balancer, ví dụ AWS ELB/NLB, GCP LB, Azure LB.
- Mỗi Service type LoadBalancer thường tạo một cloud load balancer riêng, có thể tốn chi phí.

Ví dụ:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 8080
```

Trả lời ngắn:

> LoadBalancer expose service ra ngoài cluster bằng external load balancer của cloud provider. Phù hợp cho entrypoint public, nhưng thường tốn chi phí hơn ClusterIP.

### Bảng so sánh

| Tiêu chí | ClusterIP | LoadBalancer |
|---|---|---|
| Phạm vi truy cập | Nội bộ cluster | Bên ngoài cluster |
| External IP | Không có | Có |
| Mặc định Service type | Có | Không |
| Phụ thuộc cloud LB | Không | Có, trừ minikube/metallb local |
| Use case | Internal service | Public/external service |
| Chi phí cloud | Thường không tạo LB riêng | Thường tạo LB riêng |

Câu trả lời vấn đáp mẫu:

> ClusterIP là service nội bộ trong cluster, dùng cho các workload gọi nhau bằng DNS nội bộ. LoadBalancer là service expose ra ngoài cluster, trên cloud sẽ tạo external load balancer và external IP. Nếu chỉ backend nội bộ thì dùng ClusterIP; nếu cần user bên ngoài truy cập trực tiếp thì dùng LoadBalancer hoặc Ingress.

## 16. NodePort, Ingress Và Quan Hệ Với Service

### NodePort

NodePort expose Service qua port trên mỗi node:

```text
NodeIP:NodePort -> Service -> Pod
```

Thường dùng cho lab/dev, không phải cách đẹp cho production public traffic.

### Ingress

Ingress quản lý HTTP/HTTPS routing vào các Service trong cluster.

Thường flow production:

```text
Internet -> Load Balancer -> Ingress Controller -> Service -> Pod
```

Ingress phù hợp khi:

- Nhiều app/domain/path cần chung một entrypoint.
- Cần TLS termination.
- Cần routing theo host/path.

Trả lời nhanh:

> Service expose Pod ở layer 4, còn Ingress routing HTTP/HTTPS ở layer 7 vào Service. Ingress cần Ingress Controller như NGINX Ingress, ALB Controller hoặc Traefik.

## 17. Kubernetes Labels Và Selectors

Label là key-value gắn vào object. Selector dùng để chọn object theo label.

Ví dụ Deployment tạo Pod có label:

```yaml
metadata:
  labels:
    app: nginx
```

Service chọn Pod:

```yaml
selector:
  app: nginx
```

Nếu selector sai, Service không tìm thấy Pod backend.

Debug:

```bash
kubectl get pod --show-labels
kubectl describe service nginx
kubectl get endpoints nginx
```

Câu trả lời:

> Service không nối trực tiếp tới Deployment. Service chọn Pod bằng label selector. Deployment chỉ tạo Pod có label phù hợp. Nếu label khớp, Service route traffic tới các Pod đó.

## 18. ConfigMap Và Secret

ConfigMap lưu cấu hình không nhạy cảm:

- ENV name.
- Feature flag.
- App config.

Secret lưu dữ liệu nhạy cảm:

- Password.
- Token.
- Certificate.

Lưu ý:

> Secret trong Kubernetes mặc định chỉ base64 encode, không phải mã hóa mạnh nếu cluster chưa bật encryption at rest. Cần RBAC và secret management đúng cách.

## 19. Probe

Kubernetes có các probe hay gặp:

- Liveness probe: container còn sống không? Fail thì restart container.
- Readiness probe: container đã sẵn sàng nhận traffic chưa? Fail thì remove khỏi Service endpoint.
- Startup probe: ứng dụng khởi động lâu đã xong chưa? Bảo vệ app khỏi bị liveness kill quá sớm.

Trả lời mẫu:

> Liveness để restart app bị treo. Readiness để quyết định Pod có nhận traffic không. Startup dùng cho app khởi động chậm.

## 20. NetworkPolicy

NetworkPolicy định nghĩa rule cho traffic vào/ra Pod.

Mặc định nếu không có NetworkPolicy, nhiều cluster cho phép traffic nội bộ tương đối mở. Khi áp dụng policy, có thể giới hạn Pod nào được nói chuyện với Pod nào.

Trả lời nhanh:

> NetworkPolicy giống firewall rule ở mức Pod, dùng label selector để giới hạn ingress/egress traffic. Nó cần CNI plugin hỗ trợ NetworkPolicy.

## 21. Minikube

Minikube là công cụ tạo một Kubernetes cluster local trên laptop để học, dev và lab. Trong W8, minikube giúp mình chạy Kubernetes mà không cần tạo cluster cloud như EKS/GKE/AKS, nên không tốn chi phí cloud và dễ thử nghiệm.

Trả lời ngắn:

> Minikube là Kubernetes cluster chạy local trên máy cá nhân. Nó phù hợp cho học và lab, nhưng không đại diện hoàn toàn cho production vì thường chỉ có một node, networking/load balancer/storage đơn giản hơn cloud thật.

### Minikube dùng để làm gì?

- Học Kubernetes local.
- Test manifest YAML trước khi đưa lên cluster thật.
- Chạy lab Deployment, Service, ConfigMap, Secret, NetworkPolicy.
- Thử Ingress, metrics-server, dashboard.
- Debug nhanh app containerized mà không cần cloud.

### Minikube khác cluster cloud như EKS/GKE/AKS thế nào?

| Tiêu chí | Minikube | Cluster cloud |
|---|---|---|
| Mục đích | Học/dev/lab local | Production/staging thật |
| Số node | Thường 1 node, có thể multi-node | Nhiều node, autoscaling |
| LoadBalancer | Cần `minikube tunnel` hoặc addon | Cloud tự tạo LB |
| Storage | Local/default StorageClass đơn giản | Cloud disk như EBS/GCE Disk/Azure Disk |
| Chi phí | Gần như không tốn cloud cost | Tốn tiền compute/LB/storage |
| Độ giống production | Vừa đủ học concept | Đầy đủ integration cloud |

### Lệnh minikube hay dùng

```bash
minikube start
minikube start --driver=docker
minikube status
minikube stop
minikube delete
minikube version
kubectl get nodes
```

Giải thích:

- `minikube start`: tạo và khởi động cluster local.
- `--driver=docker`: chạy minikube bằng Docker driver.
- `minikube status`: kiểm tra cluster đang chạy chưa.
- `minikube stop`: dừng cluster nhưng giữ dữ liệu.
- `minikube delete`: xóa cluster local.
- `kubectl get nodes`: kiểm tra kubectl đã nói chuyện được với cluster chưa.

### Làm sao truy cập app trong minikube?

Nếu Service là `NodePort`:

```bash
minikube service <service-name> --url
```

Lệnh này trả về URL để truy cập Service từ máy local.

Nếu muốn port-forward để debug:

```bash
kubectl port-forward svc/<service-name> 8080:80
```

Sau đó truy cập:

```text
http://localhost:8080
```

Trả lời vấn đáp:

> Với minikube, em thường truy cập app bằng `minikube service <svc> --url` nếu Service là NodePort, hoặc dùng `kubectl port-forward` để debug nhanh từ local vào Service/Pod.

### Vì sao Service `LoadBalancer` bị Pending trên minikube?

Trên cloud thật, Service type `LoadBalancer` sẽ gọi cloud provider để tạo external load balancer. Minikube chạy local nên không có cloud provider thật để cấp external IP.

Dấu hiệu:

```bash
kubectl get svc
```

Thấy `EXTERNAL-IP` là `<pending>`.

Cách xử lý:

```bash
minikube tunnel
```

Hoặc đổi Service sang `NodePort` cho lab local.

Trả lời vấn đáp:

> LoadBalancer pending trên minikube vì local cluster không có cloud load balancer thật. Có thể dùng `minikube tunnel` để giả lập external IP, hoặc dùng NodePort/port-forward cho lab.

### Image local trong minikube

Một lỗi rất hay gặp: mình build Docker image trên máy local nhưng Pod trong minikube báo `ImagePullBackOff` hoặc `ErrImagePull`.

Lý do:

- Docker daemon của máy host và môi trường image bên trong minikube có thể khác nhau.
- Kubernetes trong minikube không thấy image mình vừa build ở Docker host.
- Nếu image tag không có trong cluster, Kubernetes cố pull từ registry.

Cách xử lý 1: load image vào minikube

```bash
docker build -t myapp:1.0 .
minikube image load myapp:1.0
```

Trong manifest nên đặt:

```yaml
image: myapp:1.0
imagePullPolicy: IfNotPresent
```

Cách xử lý 2: build trực tiếp trong Docker environment của minikube

```bash
minikube docker-env
```

Sau đó cấu hình shell theo output rồi build image. Cách này tùy shell/OS nên trong lab Windows thường dùng `minikube image load` dễ nhớ hơn.

Trả lời vấn đáp:

> Nếu Pod trên minikube không pull được image local, em kiểm tra image tag, dùng `minikube image load <image>` để nạp image vào cluster và đặt `imagePullPolicy: IfNotPresent`. Nếu không, Kubernetes sẽ cố pull image từ registry bên ngoài.

### Addons hay dùng trong minikube

Liệt kê addon:

```bash
minikube addons list
```

Bật Ingress:

```bash
minikube addons enable ingress
```

Bật metrics-server để dùng HPA:

```bash
minikube addons enable metrics-server
```

Mở dashboard:

```bash
minikube dashboard
```

Các addon đáng nhớ:

- `ingress`: chạy Ingress Controller local.
- `metrics-server`: cung cấp metric CPU/memory cho HPA.
- `dashboard`: UI quản trị cluster.
- `storage-provisioner`: hỗ trợ cấp PVC local.

### Ingress trên minikube

Ingress dùng để route HTTP/HTTPS theo host/path vào Service. Trên minikube cần bật addon ingress trước:

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

Flow:

```text
Browser -> Ingress Controller -> Service -> Pod
```

Trả lời nhanh:

> Ingress chỉ là rule routing; muốn Ingress hoạt động phải có Ingress Controller. Trên minikube có thể bật bằng `minikube addons enable ingress`.

### HPA trên minikube

HPA là Horizontal Pod Autoscaler, tự tăng/giảm số Pod theo metric như CPU.

Để HPA hoạt động trên minikube cần metrics-server:

```bash
minikube addons enable metrics-server
kubectl top pod
kubectl top node
```

Nếu `kubectl top` chưa có dữ liệu, HPA cũng chưa scale được.

Trả lời nhanh:

> HPA cần metrics-server để lấy metric. Trên minikube phải bật `metrics-server`, sau đó kiểm tra bằng `kubectl top pod` hoặc `kubectl top node`.

### Debug minikube và Kubernetes lab

Khi app không chạy:

```bash
minikube status
kubectl get nodes
kubectl get pods -A
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl get events --sort-by=.lastTimestamp
```

Khi Service không truy cập được:

```bash
kubectl get svc
kubectl describe svc <service-name>
kubectl get endpoints <service-name>
kubectl get pod --show-labels
minikube service <service-name> --url
```

Khi image lỗi:

```bash
kubectl describe pod <pod-name>
minikube image ls
minikube image load <image-name>:<tag>
```

Các lỗi hay gặp:

- `ImagePullBackOff`: image không tồn tại trong minikube hoặc registry.
- `CrashLoopBackOff`: container start rồi crash liên tục.
- Service không có endpoint: selector không khớp label Pod hoặc Pod chưa Ready.
- LoadBalancer pending: cần `minikube tunnel` hoặc dùng NodePort.
- HPA không hoạt động: chưa bật metrics-server.

### Câu trả lời tổng hợp về minikube

> Minikube là cluster Kubernetes local dùng cho học và lab. Em dùng nó để apply manifest, test Deployment, Service, Ingress, HPA và debug app trước khi lên cloud. Khi truy cập app, em có thể dùng `minikube service --url`, `kubectl port-forward`, hoặc `minikube tunnel` nếu Service type LoadBalancer. Lỗi thường gặp là image local không có trong cluster, LoadBalancer bị pending, selector Service không khớp Pod label, hoặc thiếu metrics-server cho HPA.

## 22. Câu Hỏi Tương Tự Và Cách Trả Lời

### Nếu Pod chết thì ai tạo lại?

ReplicaSet tạo lại Pod nếu Pod thuộc Deployment/ReplicaSet. Nếu tạo Pod standalone thì Pod chết có thể không được tạo lại theo cách mong muốn.

### Nếu Deployment replicas = 3 thì có mấy Pod?

Mong muốn có 3 Pod available. Thực tế có thể tạm thời ít hơn hoặc nhiều hơn trong lúc rolling update, crash hoặc scheduling.

### Service có route đến Deployment không?

Không route đến Deployment trực tiếp. Service route đến Pod thông qua label selector.

### Vì sao Service có IP ổn định còn Pod IP không ổn định?

Pod ephemeral, có thể bị recreate nên IP thay đổi. Service là abstraction ổn định, có DNS name và virtual IP để client gọi.

### Rolling update diễn ra như thế nào?

Deployment tạo ReplicaSet mới với image/config mới, tăng Pod mới lên dần dần, đồng thời giảm Pod cũ xuống theo strategy.

### Rollback Deployment như thế nào?

```bash
kubectl rollout undo deployment/nginx
```

### Xem rollout status?

```bash
kubectl rollout status deployment/nginx
```

### Debug Service không vào được app?

Kiểm tra theo chuỗi:

```bash
kubectl get pod -o wide
kubectl get svc
kubectl get endpoints
kubectl describe svc <service-name>
kubectl logs <pod-name>
kubectl describe pod <pod-name>
```

Hay gặp:

- Selector của Service không khớp label Pod.
- `targetPort` sai.
- Pod chưa ready.
- App listen sai port.
- NetworkPolicy chặn traffic.

### Debug Pod CrashLoopBackOff?

```bash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl describe pod <pod-name>
```

Nguyên nhân hay gặp:

- App crash do config sai.
- Thiếu env/secret.
- Image sai.
- Command/entrypoint sai.
- Liveness probe quá chặt.

### Terraform apply lỗi do resource đã tồn tại thì làm gì?

Nếu resource đó nên do Terraform quản lý, import vào state:

```bash
terraform import <resource-address> <real-resource-id>
```

Sau đó `terraform plan` và sửa config cho khớp.

### Terraform state bị lock thì làm gì?

Kiểm tra có ai/pipeline đang apply không. Nếu chắc chắn lock bị kẹt do crash, có thể dùng:

```bash
terraform force-unlock <LOCK_ID>
```

Không force-unlock nếu chưa chắc chắn, vì có thể làm hỏng state khi có apply đang chạy.

## 23. Cheat Sheet Lệnh

Terraform:

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
terraform destroy
terraform state list
terraform state show <address>
terraform import <address> <id>
terraform apply -replace=<address>
```

Kubernetes:

```bash
kubectl get pod
kubectl get pod -o wide
kubectl get deployment
kubectl get replicaset
kubectl get svc
kubectl get endpoints
kubectl describe pod <pod>
kubectl describe svc <service>
kubectl logs <pod>
kubectl rollout status deployment/<name>
kubectl rollout undo deployment/<name>
```

Minikube:

```bash
minikube start --driver=docker
minikube status
minikube stop
minikube delete
minikube service <service-name> --url
minikube tunnel
minikube image load <image>:<tag>
minikube addons list
minikube addons enable ingress
minikube addons enable metrics-server
minikube dashboard
```

## 24. Câu Trả Lời Tổng Hợp 2 Phút

> Terraform là công cụ IaC giúp khai báo hạ tầng bằng HCL và quản lý vòng đời resource thông qua provider. Workflow cơ bản là init, validate, plan, apply. Điểm rất quan trọng của Terraform là state, vì state lưu mapping giữa code và hạ tầng thật. Làm cá nhân có thể dùng local state, nhưng làm team nên dùng remote backend như S3, bật versioning và locking. Nếu apply lỗi giữa chừng, em sẽ không destroy ngay; em đọc error, xem state, chạy plan lại, sửa nguyên nhân lỗi, import resource nếu đã tạo ngoài cloud nhưng chưa vào state, rồi apply tiếp.
>
> Với Kubernetes, Pod là đơn vị chạy container, ReplicaSet đảm bảo số lượng Pod, Deployment quản lý ReplicaSet và rollout/rollback. Service cung cấp endpoint ổn định cho Pod. ClusterIP chỉ expose nội bộ trong cluster, còn LoadBalancer expose ra ngoài cluster bằng external load balancer. Service chọn Pod bằng label selector, không route trực tiếp đến Deployment. Với lab local, minikube giúp chạy Kubernetes trên máy cá nhân; em cần nhớ các bẫy như image local chưa load vào cluster, LoadBalancer pending nếu chưa chạy `minikube tunnel`, và HPA cần `metrics-server`.
