Absolutely! Let’s convert the **line-by-line Terraform explanation** into a clean **Markdown (`.md`) format**, and I’ll also explain the **ingress question for the cluster**.

---

# Terraform EKS Setup Explained (Markdown)

## 1. Provider

```hcl
provider "aws" {
  region = "us-east-1"
}
```

* Sets the **cloud provider** to AWS.
* `region = "us-east-1"` → all resources are in **N. Virginia**.

---

## 2. VPC

```hcl
resource "aws_vpc" "devopsshack_vpc" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "devopsshack-vpc"
  }
}
```

* Creates an isolated network (**VPC**) in AWS.
* `cidr_block = "10.0.0.0/16"` → network range (65,536 IP addresses).
* `tags` → helps identify resources in AWS Console.

---

## 3. Public Subnets

```hcl
resource "aws_subnet" "devopsshack_public_subnet" {
  count                   = 2
  vpc_id                  = aws_vpc.devopsshack_vpc.id
  cidr_block              = cidrsubnet(aws_vpc.devopsshack_vpc.cidr_block, 8, count.index)
  availability_zone       = element(["us-east-1a", "us-east-1b"], count.index)
  map_public_ip_on_launch = true

  tags = {
    Name = "devopsshack-public-${count.index}"
  }
}
```

* Creates **2 public subnets** for NAT and IGW.
* `cidrsubnet(...)` → automatically calculates subnet ranges.
* `map_public_ip_on_launch = true` → EC2 instances get public IPs.

---

## 4. Private Subnets

```hcl
resource "aws_subnet" "devopsshack_private_subnet" {
  count             = 2
  vpc_id            = aws_vpc.devopsshack_vpc.id
  cidr_block        = cidrsubnet(aws_vpc.devopsshack_vpc.cidr_block, 8, count.index + 2)
  availability_zone = element(["us-east-1a", "us-east-1b"], count.index)

  tags = {
    Name = "devopsshack-private-${count.index}"
  }
}
```

* **2 private subnets** for worker nodes.
* Nodes cannot be reached directly from the internet.

---

## 5. Internet Gateway

```hcl
resource "aws_internet_gateway" "devopsshack_igw" {
  vpc_id = aws_vpc.devopsshack_vpc.id

  tags = {
    Name = "devopsshack-igw"
  }
}
```

* Allows public subnets (and NAT) to access the internet.

---

## 6. Public Route Table

```hcl
resource "aws_route_table" "devopsshack_public_rt" {
  vpc_id = aws_vpc.devopsshack_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.devopsshack_igw.id
  }

  tags = {
    Name = "devopsshack-public-rt"
  }
}

resource "aws_route_table_association" "public_assoc" {
  count          = 2
  subnet_id      = aws_subnet.devopsshack_public_subnet[count.index].id
  route_table_id = aws_route_table.devopsshack_public_rt.id
}
```

* Default route `0.0.0.0/0` → directs internet traffic via IGW.
* Associates public subnets with this route table.

---

## 7. NAT Gateways

```hcl
resource "aws_eip" "nat_eip" {
  count = 2
  vpc   = true
}

resource "aws_nat_gateway" "devopsshack_nat" {
  count         = 2
  allocation_id = aws_eip.nat_eip[count.index].id
  subnet_id     = aws_subnet.devopsshack_public_subnet[count.index].id

  tags = {
    Name = "devopsshack-nat-${count.index}"
  }
}
```

* Provides internet access for **private subnets**.
* Uses **Elastic IPs** (required).

---

## 8. Private Route Tables

```hcl
resource "aws_route_table" "devopsshack_private_rt" {
  count  = 2
  vpc_id = aws_vpc.devopsshack_vpc.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.devopsshack_nat[count.index].id
  }

  tags = {
    Name = "devopsshack-private-rt-${count.index}"
  }
}

resource "aws_route_table_association" "private_assoc" {
  count          = 2
  subnet_id      = aws_subnet.devopsshack_private_subnet[count.index].id
  route_table_id = aws_route_table.devopsshack_private_rt[count.index].id
}
```

* Routes internet-bound traffic **from private subnets** through NAT gateways.

---

## 9. Cluster Security Group

```hcl
resource "aws_security_group" "devopsshack_cluster_sg" {
  vpc_id = aws_vpc.devopsshack_vpc.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "devopsshack-cluster-sg"
  }
}
```

* Security group used by **both cluster and nodes**.
* Allows all outbound traffic.
* No ingress rules needed if **no direct SSH or external access**.

> ✅ Ingress is **not required** for the EKS cluster itself, because **pods are accessed via Kubernetes services**, not directly through the node security group. If you want to expose a service externally, you do it via **LoadBalancer type Services**, not SSH.

---

## 10. IAM Roles

* **Cluster Role** → EKS control plane permissions.
* **Node Group Role** → worker nodes’ permissions for ECR, EBS, CNI, etc.
* AWS provides managed policy ARNs:

```hcl
policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
```

* You **don’t memorize ARNs**; just understand the purpose.

---

## 11. EKS Cluster

```hcl
resource "aws_eks_cluster" "devopsshack" {
  name     = "devopsshack-cluster"
  role_arn = aws_iam_role.devopsshack_cluster_role.arn

  vpc_config {
    subnet_ids         = aws_subnet.devopsshack_private_subnet[*].id
    security_group_ids = [aws_security_group.devopsshack_cluster_sg.id]
  }

  depends_on = [aws_iam_role_policy_attachment.devopsshack_cluster_policy]
}
```

* Creates the **managed Kubernetes control plane**.
* Uses **private subnets**.
* Security group ensures nodes can communicate with the control plane.

---

## 12. EKS Addon (EBS CSI Driver)

```hcl
resource "aws_eks_addon" "ebs_csi_driver" {
  cluster_name                = aws_eks_cluster.devopsshack.name
  addon_name                  = "aws-ebs-csi-driver"
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"
}
```

* Installs the **EBS CSI driver**, allowing pods to dynamically provision **EBS volumes**.

---

## 13. EKS Node Group

```hcl
resource "aws_eks_node_group" "devopsshack" {
  cluster_name    = aws_eks_cluster.devopsshack.name
  node_group_name = "devopsshack-node-group"
  node_role_arn   = aws_iam_role.devopsshack_node_group_role.arn
  subnet_ids      = aws_subnet.devopsshack_private_subnet[*].id

  scaling_config {
    desired_size = 3
    max_size     = 5
    min_size     = 2
  }

  instance_types = ["t3.medium"]

  remote_access {
    ec2_ssh_key = null
  }

  depends_on = [aws_eks_cluster.devopsshack]
}
```

* Creates **EC2 instances** that run your pods.
* Runs in **private subnets**.
* Uses the **cluster SG**.
* SSH disabled for simplicity.

---

### ✅ Summary Notes

* **Ingress is not needed** on the cluster SG unless you want direct SSH or node-level access.
* Pods and services handle external access via **LoadBalancer, NodePort, or Ingress controllers**.
* Security group mainly ensures **cluster-node communication** and outbound internet access for nodes.

---

