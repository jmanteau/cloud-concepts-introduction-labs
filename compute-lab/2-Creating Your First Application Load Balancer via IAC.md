

# Lab: Creating Your First Application Load Balancer via IAC



### Objective
This lab provides hands-on experience with Terraform, a powerful Infrastructure as Code (IaC) tool. By the end of this lab, students will:
- Understand the use of Terraform to manage AWS resources.
- Deploy a web application using EC2 instances and an Application Load Balancer (ALB).
- Learn about resource dependencies and Terraform outputs.



### Setup setup

* Launch an small T3.small instannce with a security group allowing SSH from your IP and creating & downloading the SSH key to access it. Use latest Amazon Linux as OS (standard login user is ec2-user).

* Configure AWS credentials

  Take the output from the Lab console to copy and paste the following into ~/.aws/credentials

```
[default]
aws_access_key_id=XXXXX
aws_secret_access_key=YYYYYY
aws_session_token=ZZZZZ
```

(create the directory if not present)

* Terraform installed on the local system.

```
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo yum -y install terraform
```



## Terraform

HashiCorp Terraform is an infrastructure as code tool that lets you  define both cloud and on-prem resources in human-readable configuration  files that you can version, reuse, and share. You can then use a  consistent workflow to provision and manage all of your infrastructure  throughout its lifecycle. Terraform can manage low-level components like compute, storage, and networking resources, as well as high-level  components like DNS entries and SaaS features.



## Lab steps



### Initialize Terraform

When you create a new configuration — or check out an existing configuration from version control — you need to initialize the directory with `terraform init`.

Initializing a configuration directory downloads and installs the providers defined in the configuration, which in this case is the `aws` provider.

Initialize the directory.



```
$ terraform init

Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 4.16"...
- Installing hashicorp/aws v4.17.0...
- Installed hashicorp/aws v4.17.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

Terraform downloads the `aws` provider and installs it in a hidden subdirectory of your current working directory, named `.terraform`. The `terraform init` command prints out which version of the provider was installed. Terraform also creates a lock file named `.terraform.lock.hcl` which specifies the exact provider versions used, so that you can control when you want to update the providers used for your project.



### Create the terraform file

Create a `alb.tf` file by adding the file below



```
# Provider Configuration
provider "aws" {
  region = "us-east-1"
}

# Get all VPCs in the region and select the first one
data "aws_vpcs" "all_vpcs" {}

# Fetch all subnets for the selected VPC
data "aws_subnets" "vpc_subnets" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpcs.all_vpcs.ids[0]]
  }
}

# Variables
variable "instance_type" {
  default = "t2.micro"
}
variable "instance_colors" {
  default = {
    blue  = "blue"
    green = "green"
    red   = "red"
  }
}

# Fetch the latest Amazon Linux 2023 AMI
data "aws_ami" "amazon_linux_2023" {
  most_recent = true

  filter {
    name   = "name"
    values = ["al2023-ami-*"]
  }

  filter {
    name   = "architecture"
    values = ["x86_64"] # or "arm64" for ARM instances
  }

  owners = ["amazon"] # Amazon's official AMIs
}

# Security Group for EC2 Instances
resource "aws_security_group" "ec2_sg" {
  name        = "ec2_security_group"
  description = "Security group for EC2 instances"
  vpc_id      = data.aws_vpcs.all_vpcs.ids[0]

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Create Target Group
resource "aws_lb_target_group" "app_tg" {
  name     = "app-target-group"
  port     = 80
  protocol = "HTTP"
  vpc_id   = data.aws_vpcs.all_vpcs.ids[0]

  health_check {
    path                = "/"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 5
    unhealthy_threshold = 2
    matcher             = "200-399"
  }
}

# Create ALB
resource "aws_lb" "app_alb" {
  name               = "app-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.ec2_sg.id]
  subnets            = data.aws_subnets.vpc_subnets.ids

  enable_deletion_protection = false
  idle_timeout               = 60
}

# Create Listener for ALB
resource "aws_lb_listener" "http_listener" {
  load_balancer_arn = aws_lb.app_alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_tg.arn
  }
}

# Launch EC2 Instances with Dynamic User Data and Subnet Distribution
resource "aws_instance" "app_ec2" {
  for_each      = var.instance_colors
  ami           = data.aws_ami.amazon_linux_2023.id
  instance_type = var.instance_type
  subnet_id     = element(data.aws_subnets.vpc_subnets.ids, index(keys(var.instance_colors), each.key))
  security_group_ids = [
    aws_security_group.ec2_sg.id
  ]

  user_data = <<-EOT
              #!/bin/bash
              sudo yum update -y
              sudo yum install nginx -y
              sudo service nginx start
              echo '<html><body style="background-color:${each.value};"><h1>EC2 ${upper(each.key)} server</h1></body></html>' | sudo tee /usr/share/nginx/html/index.html > /dev/null
              sudo service nginx reload
              EOT

  tags = {
    Name = "AppServer-${each.key}"
  }
}

# Attach EC2 Instances to Target Group
resource "aws_lb_target_group_attachment" "tg_attachment" {
  for_each         = aws_instance.app_ec2
  target_group_arn = aws_lb_target_group.app_tg.arn
  target_id        = each.value.id
  port             = 80
}

# Outputs 
output "first_vpc_id" {
  value = data.aws_vpcs.all_vpcs.ids[0]
}

output "subnet_ids" {
  value = data.aws_subnets.vpc_subnets.ids
}

output "amazon_linux_2023_ami" {
  value = data.aws_ami.amazon_linux_2023.id
}

output "alb_dns_name" {
  value = aws_lb.app_alb.dns_name
}
```



### Format and validate the configuration

We recommend using consistent formatting in all of your configuration files. The `terraform fmt` command automatically updates configurations in the current directory for readability and consistency.

Format your configuration. Terraform will print out the names of the files it modified, if any. In this case, your configuration file was already formatted correctly, so Terraform won't return any file names.

```
$ terraform fmt
```

You can also make sure your configuration is syntactically valid and internally consistent by using the `terraform validate` command.

Validate your configuration, you should see this error

```
[root@ip-172-31-75-105 environment]# terraform validate
╷
│ Error: Unsupported argument
│ 
│   on alb.tf line 114, in resource "aws_instance" "app_ec2":
│  114:   security_group_ids = [
│ 
│ An argument named "security_group_ids" is not expected here.
```

There is indeed a type and it should be vpc_security_group_ids. Correct this.

The example configuration provided above is now valid, so Terraform will return a success message.

```
$ terraform validate
Success! The configuration is valid.
```



#### Review the Terraform Plan

Before it applies any changes, Terraform prints out the *execution plan* which describes the actions Terraform will take in order to change your infrastructure to match the configuration.

The output format is similar to the diff format generated by tools such as Git. The output has a `+` next to `aws_instance.app_server`, meaning that Terraform will create this resource. Beneath that, it shows the attributes that will be set. When the value displayed is `(known after apply)`, it means that the value will not be known until the resource is created. For example, AWS assigns Amazon Resource Names (ARNs) to instances upon creation, so Terraform cannot know the value of the `arn` attribute until you apply the change and the AWS provider returns that value from the AWS API.

Run:

```
terraform plan
```

This command generates and displays the execution plan. Analyze the resources that Terraform will create, such as EC2 instances, an ALB, and security groups.



```
# terraform plan
data.aws_vpcs.all_vpcs: Reading...
data.aws_ami.amazon_linux_2023: Reading...
data.aws_vpcs.all_vpcs: Read complete after 0s [id=us-east-1]
data.aws_subnets.vpc_subnets: Reading...
data.aws_subnets.vpc_subnets: Read complete after 0s [id=us-east-1]
data.aws_ami.amazon_linux_2023: Read complete after 1s [id=ami-0220226481c20c75f]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_instance.app_ec2["blue"] will be created
  + resource "aws_instance" "app_ec2" {
      + ami                                  = "ami-0220226481c20c75f"
      + arn                                  = (known after apply)
      + associate_public_ip_address          = (known after apply)
      + availability_zone                    = (known after apply)
      + cpu_core_count                       = (known after apply)
      + cpu_threads_per_core                 = (known after apply)
      + disable_api_stop                     = (known after apply)
      + disable_api_termination              = (known after apply)
      + ebs_optimized                        = (known after apply)
      + enable_primary_ipv6                  = (known after apply)
      + get_password_data                    = false
      + host_id                              = (known after apply)
      + host_resource_group_arn              = (known after apply)
      + iam_instance_profile                 = (known after apply)
      + id                                   = (known after apply)
      + instance_initiated_shutdown_behavior = (known after apply)
      + instance_lifecycle                   = (known after apply)
      + instance_state                       = (known after apply)
      + instance_type                        = "t2.micro"
      + ipv6_address_count                   = (known after apply)
      + ipv6_addresses                       = (known after apply)
      + key_name                             = (known after apply)
      + monitoring                           = (known after apply)
      + outpost_arn                          = (known after apply)
      + password_data                        = (known after apply)
      + placement_group                      = (known after apply)
      + placement_partition_number           = (known after apply)
      + primary_network_interface_id         = (known after apply)
      + private_dns                          = (known after apply)
      + private_ip                           = (known after apply)
      + public_dns                           = (known after apply)
      + public_ip                            = (known after apply)
      + secondary_private_ips                = (known after apply)
      + security_groups                      = (known after apply)
      + source_dest_check                    = true
      + spot_instance_request_id             = (known after apply)
      + subnet_id                            = "subnet-0d4cdbcfc2c29cf38"
      + tags                                 = {
          + "Name" = "AppServer-blue"
        }
      + tags_all                             = {
          + "Name" = "AppServer-blue"
        }
      + tenancy                              = (known after apply)
      + user_data                            = "e138f63a3df8481c35a14f0928beecb4f6c5ee58"
      + user_data_base64                     = (known after apply)
      + user_data_replace_on_change          = false
      + vpc_security_group_ids               = (known after apply)

      + capacity_reservation_specification (known after apply)

      + cpu_options (known after apply)

      + ebs_block_device (known after apply)

      + enclave_options (known after apply)

      + ephemeral_block_device (known after apply)

      + instance_market_options (known after apply)

      + maintenance_options (known after apply)

      + metadata_options (known after apply)

      + network_interface (known after apply)

      + private_dns_name_options (known after apply)

      + root_block_device (known after apply)
    }

  # aws_instance.app_ec2["green"] will be created
  + resource "aws_instance" "app_ec2" {
      + ami                                  = "ami-0220226481c20c75f"
      + arn                                  = (known after apply)
      + associate_public_ip_address          = (known after apply)
      + availability_zone                    = (known after apply)
      + cpu_core_count                       = (known after apply)
      + cpu_threads_per_core                 = (known after apply)
      + disable_api_stop                     = (known after apply)
      + disable_api_termination              = (known after apply)
      + ebs_optimized                        = (known after apply)
      + enable_primary_ipv6                  = (known after apply)
      + get_password_data                    = false
      + host_id                              = (known after apply)
      + host_resource_group_arn              = (known after apply)
      + iam_instance_profile                 = (known after apply)
      + id                                   = (known after apply)
      + instance_initiated_shutdown_behavior = (known after apply)
      + instance_lifecycle                   = (known after apply)
      + instance_state                       = (known after apply)
      + instance_type                        = "t2.micro"
      + ipv6_address_count                   = (known after apply)
      + ipv6_addresses                       = (known after apply)
      + key_name                             = (known after apply)
      + monitoring                           = (known after apply)
      + outpost_arn                          = (known after apply)
      + password_data                        = (known after apply)
      + placement_group                      = (known after apply)
      + placement_partition_number           = (known after apply)
      + primary_network_interface_id         = (known after apply)
      + private_dns                          = (known after apply)
      + private_ip                           = (known after apply)
      + public_dns                           = (known after apply)
      + public_ip                            = (known after apply)
      + secondary_private_ips                = (known after apply)
      + security_groups                      = (known after apply)
      + source_dest_check                    = true
      + spot_instance_request_id             = (known after apply)
      + subnet_id                            = "subnet-07ae9d7a2d9ae8f57"
      + tags                                 = {
          + "Name" = "AppServer-green"
        }
      + tags_all                             = {
          + "Name" = "AppServer-green"
        }
      + tenancy                              = (known after apply)
      + user_data                            = "5f6316693522159401acd62a52573c21e8c5892a"
      + user_data_base64                     = (known after apply)
      + user_data_replace_on_change          = false
      + vpc_security_group_ids               = (known after apply)

      + capacity_reservation_specification (known after apply)

      + cpu_options (known after apply)

      + ebs_block_device (known after apply)

      + enclave_options (known after apply)

      + ephemeral_block_device (known after apply)

      + instance_market_options (known after apply)

      + maintenance_options (known after apply)

      + metadata_options (known after apply)

      + network_interface (known after apply)

      + private_dns_name_options (known after apply)

      + root_block_device (known after apply)
    }

  # aws_instance.app_ec2["red"] will be created
  + resource "aws_instance" "app_ec2" {
      + ami                                  = "ami-0220226481c20c75f"
      + arn                                  = (known after apply)
      + associate_public_ip_address          = (known after apply)
      + availability_zone                    = (known after apply)
      + cpu_core_count                       = (known after apply)
      + cpu_threads_per_core                 = (known after apply)
      + disable_api_stop                     = (known after apply)
      + disable_api_termination              = (known after apply)
      + ebs_optimized                        = (known after apply)
      + enable_primary_ipv6                  = (known after apply)
      + get_password_data                    = false
      + host_id                              = (known after apply)
      + host_resource_group_arn              = (known after apply)
      + iam_instance_profile                 = (known after apply)
      + id                                   = (known after apply)
      + instance_initiated_shutdown_behavior = (known after apply)
      + instance_lifecycle                   = (known after apply)
      + instance_state                       = (known after apply)
      + instance_type                        = "t2.micro"
      + ipv6_address_count                   = (known after apply)
      + ipv6_addresses                       = (known after apply)
      + key_name                             = (known after apply)
      + monitoring                           = (known after apply)
      + outpost_arn                          = (known after apply)
      + password_data                        = (known after apply)
      + placement_group                      = (known after apply)
      + placement_partition_number           = (known after apply)
      + primary_network_interface_id         = (known after apply)
      + private_dns                          = (known after apply)
      + private_ip                           = (known after apply)
      + public_dns                           = (known after apply)
      + public_ip                            = (known after apply)
      + secondary_private_ips                = (known after apply)
      + security_groups                      = (known after apply)
      + source_dest_check                    = true
      + spot_instance_request_id             = (known after apply)
      + subnet_id                            = "subnet-0b28c02ae27327b22"
      + tags                                 = {
          + "Name" = "AppServer-red"
        }
      + tags_all                             = {
          + "Name" = "AppServer-red"
        }
      + tenancy                              = (known after apply)
      + user_data                            = "3d25ff82455905e4fea90c79badb33b3bb1a621f"
      + user_data_base64                     = (known after apply)
      + user_data_replace_on_change          = false
      + vpc_security_group_ids               = (known after apply)

      + capacity_reservation_specification (known after apply)

      + cpu_options (known after apply)

      + ebs_block_device (known after apply)

      + enclave_options (known after apply)

      + ephemeral_block_device (known after apply)

      + instance_market_options (known after apply)

      + maintenance_options (known after apply)

      + metadata_options (known after apply)

      + network_interface (known after apply)

      + private_dns_name_options (known after apply)

      + root_block_device (known after apply)
    }

  # aws_lb.app_alb will be created
  + resource "aws_lb" "app_alb" {
      + arn                                                          = (known after apply)
      + arn_suffix                                                   = (known after apply)
      + client_keep_alive                                            = 3600
      + desync_mitigation_mode                                       = "defensive"
      + dns_name                                                     = (known after apply)
      + drop_invalid_header_fields                                   = false
      + enable_deletion_protection                                   = false
      + enable_http2                                                 = true
      + enable_tls_version_and_cipher_suite_headers                  = false
      + enable_waf_fail_open                                         = false
      + enable_xff_client_port                                       = false
      + enforce_security_group_inbound_rules_on_private_link_traffic = (known after apply)
      + id                                                           = (known after apply)
      + idle_timeout                                                 = 60
      + internal                                                     = false
      + ip_address_type                                              = (known after apply)
      + load_balancer_type                                           = "application"
      + name                                                         = "app-alb"
      + name_prefix                                                  = (known after apply)
      + preserve_host_header                                         = false
      + security_groups                                              = (known after apply)
      + subnets                                                      = [
          + "subnet-013375bfa51beef27",
          + "subnet-0522e9e80d209bfc6",
          + "subnet-07ae9d7a2d9ae8f57",
          + "subnet-0b28c02ae27327b22",
          + "subnet-0d3fde3bd7076fc92",
          + "subnet-0d4cdbcfc2c29cf38",
        ]
      + tags_all                                                     = (known after apply)
      + vpc_id                                                       = (known after apply)
      + xff_header_processing_mode                                   = "append"
      + zone_id                                                      = (known after apply)

      + subnet_mapping (known after apply)
    }

  # aws_lb_listener.http_listener will be created
  + resource "aws_lb_listener" "http_listener" {
      + arn                                                                   = (known after apply)
      + id                                                                    = (known after apply)
      + load_balancer_arn                                                     = (known after apply)
      + port                                                                  = 80
      + protocol                                                              = "HTTP"
      + routing_http_request_x_amzn_mtls_clientcert_header_name               = (known after apply)
      + routing_http_request_x_amzn_mtls_clientcert_issuer_header_name        = (known after apply)
      + routing_http_request_x_amzn_mtls_clientcert_leaf_header_name          = (known after apply)
      + routing_http_request_x_amzn_mtls_clientcert_serial_number_header_name = (known after apply)
      + routing_http_request_x_amzn_mtls_clientcert_subject_header_name       = (known after apply)
      + routing_http_request_x_amzn_mtls_clientcert_validity_header_name      = (known after apply)
      + routing_http_request_x_amzn_tls_cipher_suite_header_name              = (known after apply)
      + routing_http_request_x_amzn_tls_version_header_name                   = (known after apply)
      + routing_http_response_access_control_allow_credentials_header_value   = (known after apply)
      + routing_http_response_access_control_allow_headers_header_value       = (known after apply)
      + routing_http_response_access_control_allow_methods_header_value       = (known after apply)
      + routing_http_response_access_control_allow_origin_header_value        = (known after apply)
      + routing_http_response_access_control_expose_headers_header_value      = (known after apply)
      + routing_http_response_access_control_max_age_header_value             = (known after apply)
      + routing_http_response_content_security_policy_header_value            = (known after apply)
      + routing_http_response_server_enabled                                  = (known after apply)
      + routing_http_response_strict_transport_security_header_value          = (known after apply)
      + routing_http_response_x_content_type_options_header_value             = (known after apply)
      + routing_http_response_x_frame_options_header_value                    = (known after apply)
      + ssl_policy                                                            = (known after apply)
      + tags_all                                                              = (known after apply)
      + tcp_idle_timeout_seconds                                              = (known after apply)

      + default_action {
          + order            = (known after apply)
          + target_group_arn = (known after apply)
          + type             = "forward"
        }

      + mutual_authentication (known after apply)
    }

  # aws_lb_target_group.app_tg will be created
  + resource "aws_lb_target_group" "app_tg" {
      + arn                                = (known after apply)
      + arn_suffix                         = (known after apply)
      + connection_termination             = (known after apply)
      + deregistration_delay               = "300"
      + id                                 = (known after apply)
      + ip_address_type                    = (known after apply)
      + lambda_multi_value_headers_enabled = false
      + load_balancer_arns                 = (known after apply)
      + load_balancing_algorithm_type      = (known after apply)
      + load_balancing_anomaly_mitigation  = (known after apply)
      + load_balancing_cross_zone_enabled  = (known after apply)
      + name                               = "app-target-group"
      + name_prefix                        = (known after apply)
      + port                               = 80
      + preserve_client_ip                 = (known after apply)
      + protocol                           = "HTTP"
      + protocol_version                   = (known after apply)
      + proxy_protocol_v2                  = false
      + slow_start                         = 0
      + tags_all                           = (known after apply)
      + target_type                        = "instance"
      + vpc_id                             = "vpc-0c68f0752df142156"

      + health_check {
          + enabled             = true
          + healthy_threshold   = 5
          + interval            = 30
          + matcher             = "200-399"
          + path                = "/"
          + port                = "traffic-port"
          + protocol            = "HTTP"
          + timeout             = 5
          + unhealthy_threshold = 2
        }

      + stickiness (known after apply)

      + target_failover (known after apply)

      + target_group_health (known after apply)

      + target_health_state (known after apply)
    }

  # aws_lb_target_group_attachment.tg_attachment["blue"] will be created
  + resource "aws_lb_target_group_attachment" "tg_attachment" {
      + id               = (known after apply)
      + port             = 80
      + target_group_arn = (known after apply)
      + target_id        = (known after apply)
    }

  # aws_lb_target_group_attachment.tg_attachment["green"] will be created
  + resource "aws_lb_target_group_attachment" "tg_attachment" {
      + id               = (known after apply)
      + port             = 80
      + target_group_arn = (known after apply)
      + target_id        = (known after apply)
    }

  # aws_lb_target_group_attachment.tg_attachment["red"] will be created
  + resource "aws_lb_target_group_attachment" "tg_attachment" {
      + id               = (known after apply)
      + port             = 80
      + target_group_arn = (known after apply)
      + target_id        = (known after apply)
    }

  # aws_security_group.ec2_sg will be created
  + resource "aws_security_group" "ec2_sg" {
      + arn                    = (known after apply)
      + description            = "Security group for EC2 instances"
      + egress                 = [
          + {
              + cidr_blocks      = [
                  + "0.0.0.0/0",
                ]
              + from_port        = 0
              + ipv6_cidr_blocks = []
              + prefix_list_ids  = []
              + protocol         = "-1"
              + security_groups  = []
              + self             = false
              + to_port          = 0
                # (1 unchanged attribute hidden)
            },
        ]
      + id                     = (known after apply)
      + ingress                = [
          + {
              + cidr_blocks      = [
                  + "0.0.0.0/0",
                ]
              + from_port        = 80
              + ipv6_cidr_blocks = []
              + prefix_list_ids  = []
              + protocol         = "tcp"
              + security_groups  = []
              + self             = false
              + to_port          = 80
                # (1 unchanged attribute hidden)
            },
        ]
      + name                   = "ec2_security_group"
      + name_prefix            = (known after apply)
      + owner_id               = (known after apply)
      + revoke_rules_on_delete = false
      + tags_all               = (known after apply)
      + vpc_id                 = "vpc-0c68f0752df142156"
    }

Plan: 10 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + alb_dns_name          = (known after apply)
  + amazon_linux_2023_ami = "ami-0220226481c20c75f"
  + first_vpc_id          = "vpc-0c68f0752df142156"
  + subnet_ids            = [
      + "subnet-0d4cdbcfc2c29cf38",
      + "subnet-07ae9d7a2d9ae8f57",
      + "subnet-0b28c02ae27327b22",
      + "subnet-0522e9e80d209bfc6",
      + "subnet-013375bfa51beef27",
      + "subnet-0d3fde3bd7076fc92",
    ]
```

#### Apply the Terraform Plan

Execute the following command to apply the changes:

```
terraform apply
```

Terraform will now pause and wait for your approval before proceeding. If anything in the plan seems incorrect or dangerous, it is safe to abort here before Terraform modifies your infrastructure.

You will see a detailed list of actions that Terraform will perform. Type `yes` to confirm and initiate the deployment.

```
# terraform apply
data.aws_ami.amazon_linux_2023: Reading...
data.aws_vpcs.all_vpcs: Reading...
data.aws_vpcs.all_vpcs: Read complete after 0s [id=us-east-1]
data.aws_subnets.vpc_subnets: Reading...
data.aws_subnets.vpc_subnets: Read complete after 0s [id=us-east-1]
data.aws_ami.amazon_linux_2023: Read complete after 0s [id=ami-0220226481c20c75f]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_instance.app_ec2["blue"] will be created
  
<SAME AS BEFORE>

Plan: 10 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + alb_dns_name          = (known after apply)
  + amazon_linux_2023_ami = "ami-0220226481c20c75f"
  + first_vpc_id          = "vpc-0c68f0752df142156"
  + subnet_ids            = [
      + "subnet-0d4cdbcfc2c29cf38",
      + "subnet-07ae9d7a2d9ae8f57",
      + "subnet-0b28c02ae27327b22",
      + "subnet-0522e9e80d209bfc6",
      + "subnet-013375bfa51beef27",
      + "subnet-0d3fde3bd7076fc92",
    ]

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes
  
aws_security_group.ec2_sg: Creating...
aws_lb_target_group.app_tg: Creating...
aws_lb_target_group.app_tg: Creation complete after 0s [id=arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/99d1fd9b062afb92]
aws_security_group.ec2_sg: Creation complete after 2s [id=sg-001600595760f5754]
aws_lb.app_alb: Creating...
aws_instance.app_ec2["blue"]: Creating...
aws_instance.app_ec2["green"]: Creating...
aws_instance.app_ec2["red"]: Creating...
aws_lb.app_alb: Still creating... [10s elapsed]
aws_instance.app_ec2["blue"]: Still creating... [10s elapsed]
aws_instance.app_ec2["green"]: Still creating... [10s elapsed]
aws_instance.app_ec2["red"]: Still creating... [10s elapsed]
aws_instance.app_ec2["green"]: Creation complete after 13s [id=i-061614b8dc7fb4545]
aws_instance.app_ec2["blue"]: Creation complete after 13s [id=i-05a0e7d603a177cc4]
aws_lb.app_alb: Still creating... [20s elapsed]
aws_instance.app_ec2["red"]: Still creating... [20s elapsed]
aws_instance.app_ec2["red"]: Creation complete after 22s [id=i-03899ee64760e1cdd]
aws_lb_target_group_attachment.tg_attachment["green"]: Creating...
aws_lb_target_group_attachment.tg_attachment["blue"]: Creating...
aws_lb_target_group_attachment.tg_attachment["red"]: Creating...
aws_lb_target_group_attachment.tg_attachment["red"]: Creation complete after 0s [id=arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/99d1fd9b062afb92-20250116213159212100000004]
aws_lb_target_group_attachment.tg_attachment["green"]: Creation complete after 0s [id=arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/99d1fd9b062afb92-20250116213159251400000005]
aws_lb_target_group_attachment.tg_attachment["blue"]: Creation complete after 0s [id=arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/99d1fd9b062afb92-20250116213159294400000006]
aws_lb.app_alb: Still creating... [30s elapsed]
aws_lb.app_alb: Still creating... [40s elapsed]
aws_lb.app_alb: Still creating... [50s elapsed]
aws_lb.app_alb: Still creating... [1m0s elapsed]
aws_lb.app_alb: Still creating... [1m10s elapsed]
aws_lb.app_alb: Still creating... [1m20s elapsed]
aws_lb.app_alb: Still creating... [1m30s elapsed]
aws_lb.app_alb: Still creating... [1m40s elapsed]
aws_lb.app_alb: Still creating... [1m50s elapsed]
aws_lb.app_alb: Still creating... [2m0s elapsed]
aws_lb.app_alb: Still creating... [2m10s elapsed]
aws_lb.app_alb: Still creating... [2m20s elapsed]
aws_lb.app_alb: Still creating... [2m30s elapsed]
aws_lb.app_alb: Still creating... [2m40s elapsed]
aws_lb.app_alb: Still creating... [2m50s elapsed]
aws_lb.app_alb: Still creating... [3m0s elapsed]
aws_lb.app_alb: Still creating... [3m10s elapsed]
aws_lb.app_alb: Creation complete after 3m12s [id=arn:aws:elasticloadbalancing:us-east-1:879433079150:loadbalancer/app/app-alb/d80771058a6037f4]
aws_lb_listener.http_listener: Creating...
aws_lb_listener.http_listener: Creation complete after 0s [id=arn:aws:elasticloadbalancing:us-east-1:879433079150:listener/app/app-alb/d80771058a6037f4/52cc6d055b8460fc]

Apply complete! Resources: 10 added, 0 changed, 0 destroyed.

Outputs:

alb_dns_name = "app-alb-1384372476.us-east-1.elb.amazonaws.com"
amazon_linux_2023_ami = "ami-0220226481c20c75f"
first_vpc_id = "vpc-0c68f0752df142156"
subnet_ids = tolist([
  "subnet-0d4cdbcfc2c29cf38",
  "subnet-07ae9d7a2d9ae8f57",
  "subnet-0b28c02ae27327b22",
  "subnet-0522e9e80d209bfc6",
  "subnet-013375bfa51beef27",
  "subnet-0d3fde3bd7076fc92",
])
```



### Key Resources Created

1. **EC2 Instances:**
   - Three instances named "blue", "green", and "red".
   - Instances use the  Amazon Linux 2023 AMI.
   - Subnet IDs are specified for each instance.
2. **Application Load Balancer (ALB):**
   - Named `app-alb`.
   - Distributes traffic to the EC2 instances using a target group.
3. **Security Group:**
   - Allows inbound traffic on port 80 (HTTP).
   - Outbound traffic is open to all destinations.
4. **Target Group:**
   - Associated with the ALB for load balancing traffic among the instances.
5. **Outputs:**
   - `alb_dns_name`: The DNS name of the ALB.
   - `subnet_ids`: List of subnet IDs.
   - `first_vpc_id`: The ID of the primary VPC.



### Validation Steps

1. **Access the Web Application:**

   - Retrieve the DNS name of the ALB from the Terraform output:

     ```
     terraform output alb_dns_name
     ```

   - Open the URL in your browser to verify the web application is accessible.

2. **Verify EC2 Instances:**

   - Login to the AWS Management Console and navigate to the EC2 dashboard.
   - Confirm the instances "blue", "green", and "red" are running in the specified subnets.

3. **Check Load Balancer:**

   - Navigate to the Load Balancers section in the AWS Console.
   - Confirm the ALB is active and attached to the target group.

4. **Test Load Balancing:**

   - Refresh the application URL multiple times to observe traffic distribution among instances.



### Clean-Up

To avoid incurring additional costs, destroy the resources created:

```
terraform destroy
```

Type `yes` to confirm the destruction of all resources.

```
[root@ip-172-31-75-105 environment]# terraform destroy
data.aws_vpcs.all_vpcs: Reading...
data.aws_ami.amazon_linux_2023: Reading...
data.aws_vpcs.all_vpcs: Read complete after 0s [id=us-east-1]
aws_security_group.ec2_sg: Refreshing state... [id=sg-0cbd155008d612114]
data.aws_subnets.vpc_subnets: Reading...
aws_lb_target_group.app_tg: Refreshing state... [id=arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/3acf7f3e63ec56f5]
data.aws_subnets.vpc_subnets: Read complete after 0s [id=us-east-1]
aws_lb.app_alb: Refreshing state... [id=arn:aws:elasticloadbalancing:us-east-1:879433079150:loadbalancer/app/app-alb/412e1705362e63be]
aws_lb_listener.http_listener: Refreshing state... [id=arn:aws:elasticloadbalancing:us-east-1:879433079150:listener/app/app-alb/412e1705362e63be/10ef18abe2c2576f]
data.aws_ami.amazon_linux_2023: Read complete after 1s [id=ami-0220226481c20c75f]
aws_instance.app_ec2["green"]: Refreshing state... [id=i-0ee97ce094dc9582e]
aws_instance.app_ec2["red"]: Refreshing state... [id=i-09a5e5642f47ea9e1]
aws_instance.app_ec2["blue"]: Refreshing state... [id=i-0c9539b7703295cf6]
aws_lb_target_group_attachment.tg_attachment["green"]: Refreshing state... [id=arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/3acf7f3e63ec56f5-20250116210559032100000005]
aws_lb_target_group_attachment.tg_attachment["red"]: Refreshing state... [id=arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/3acf7f3e63ec56f5-20250116210559004500000004]
aws_lb_target_group_attachment.tg_attachment["blue"]: Refreshing state... [id=arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/3acf7f3e63ec56f5-20250116210559105300000006]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # aws_instance.app_ec2["blue"] will be destroyed
  - resource "aws_instance" "app_ec2" {
      - ami                                  = "ami-0220226481c20c75f" -> null
      - arn                                  = "arn:aws:ec2:us-east-1:879433079150:instance/i-0c9539b7703295cf6" -> null
      - associate_public_ip_address          = true -> null
      - availability_zone                    = "us-east-1a" -> null
      - cpu_core_count                       = 1 -> null
      - cpu_threads_per_core                 = 1 -> null
      - disable_api_stop                     = false -> null
      - disable_api_termination              = false -> null
      - ebs_optimized                        = false -> null
      - get_password_data                    = false -> null
      - hibernation                          = false -> null
      - id                                   = "i-0c9539b7703295cf6" -> null
      - instance_initiated_shutdown_behavior = "stop" -> null
      - instance_state                       = "running" -> null
      - instance_type                        = "t2.micro" -> null
      - ipv6_address_count                   = 0 -> null
      - ipv6_addresses                       = [] -> null
      - monitoring                           = false -> null
      - placement_partition_number           = 0 -> null
      - primary_network_interface_id         = "eni-058d248de7141ed66" -> null
      - private_dns                          = "ip-172-31-14-138.ec2.internal" -> null
      - private_ip                           = "172.31.14.138" -> null
      - public_dns                           = "ec2-100-26-148-160.compute-1.amazonaws.com" -> null
      - public_ip                            = "100.26.148.160" -> null
      - secondary_private_ips                = [] -> null
      - security_groups                      = [
          - "ec2_security_group",
        ] -> null
      - source_dest_check                    = true -> null
      - subnet_id                            = "subnet-0d4cdbcfc2c29cf38" -> null
      - tags                                 = {
          - "Name" = "AppServer-blue"
        } -> null
      - tags_all                             = {
          - "Name" = "AppServer-blue"
        } -> null
      - tenancy                              = "default" -> null
      - user_data                            = "e138f63a3df8481c35a14f0928beecb4f6c5ee58" -> null
      - user_data_replace_on_change          = false -> null
      - vpc_security_group_ids               = [
          - "sg-0cbd155008d612114",
        ] -> null
        # (8 unchanged attributes hidden)

      - capacity_reservation_specification {
          - capacity_reservation_preference = "open" -> null
        }

      - cpu_options {
          - core_count       = 1 -> null
          - threads_per_core = 1 -> null
            # (1 unchanged attribute hidden)
        }

      - credit_specification {
          - cpu_credits = "standard" -> null
        }

      - enclave_options {
          - enabled = false -> null
        }

      - maintenance_options {
          - auto_recovery = "default" -> null
        }

      - metadata_options {
          - http_endpoint               = "enabled" -> null
          - http_protocol_ipv6          = "disabled" -> null
          - http_put_response_hop_limit = 2 -> null
          - http_tokens                 = "required" -> null
          - instance_metadata_tags      = "disabled" -> null
        }

      - private_dns_name_options {
          - enable_resource_name_dns_a_record    = false -> null
          - enable_resource_name_dns_aaaa_record = false -> null
          - hostname_type                        = "ip-name" -> null
        }

      - root_block_device {
          - delete_on_termination = true -> null
          - device_name           = "/dev/xvda" -> null
          - encrypted             = false -> null
          - iops                  = 3000 -> null
          - tags                  = {} -> null
          - tags_all              = {} -> null
          - throughput            = 125 -> null
          - volume_id             = "vol-00bf31b06589a6c5d" -> null
          - volume_size           = 30 -> null
          - volume_type           = "gp3" -> null
            # (1 unchanged attribute hidden)
        }
    }

  # aws_instance.app_ec2["green"] will be destroyed
  - resource "aws_instance" "app_ec2" {
      - ami                                  = "ami-0220226481c20c75f" -> null
      - arn                                  = "arn:aws:ec2:us-east-1:879433079150:instance/i-0ee97ce094dc9582e" -> null
      - associate_public_ip_address          = true -> null
      - availability_zone                    = "us-east-1f" -> null
      - cpu_core_count                       = 1 -> null
      - cpu_threads_per_core                 = 1 -> null
      - disable_api_stop                     = false -> null
      - disable_api_termination              = false -> null
      - ebs_optimized                        = false -> null
      - get_password_data                    = false -> null
      - hibernation                          = false -> null
      - id                                   = "i-0ee97ce094dc9582e" -> null
      - instance_initiated_shutdown_behavior = "stop" -> null
      - instance_state                       = "running" -> null
      - instance_type                        = "t2.micro" -> null
      - ipv6_address_count                   = 0 -> null
      - ipv6_addresses                       = [] -> null
      - monitoring                           = false -> null
      - placement_partition_number           = 0 -> null
      - primary_network_interface_id         = "eni-063c0bc87ff17f490" -> null
      - private_dns                          = "ip-172-31-79-132.ec2.internal" -> null
      - private_ip                           = "172.31.79.132" -> null
      - public_dns                           = "ec2-98-84-19-187.compute-1.amazonaws.com" -> null
      - public_ip                            = "98.84.19.187" -> null
      - secondary_private_ips                = [] -> null
      - security_groups                      = [
          - "ec2_security_group",
        ] -> null
      - source_dest_check                    = true -> null
      - subnet_id                            = "subnet-07ae9d7a2d9ae8f57" -> null
      - tags                                 = {
          - "Name" = "AppServer-green"
        } -> null
      - tags_all                             = {
          - "Name" = "AppServer-green"
        } -> null
      - tenancy                              = "default" -> null
      - user_data                            = "5f6316693522159401acd62a52573c21e8c5892a" -> null
      - user_data_replace_on_change          = false -> null
      - vpc_security_group_ids               = [
          - "sg-0cbd155008d612114",
        ] -> null
        # (8 unchanged attributes hidden)

      - capacity_reservation_specification {
          - capacity_reservation_preference = "open" -> null
        }

      - cpu_options {
          - core_count       = 1 -> null
          - threads_per_core = 1 -> null
            # (1 unchanged attribute hidden)
        }

      - credit_specification {
          - cpu_credits = "standard" -> null
        }

      - enclave_options {
          - enabled = false -> null
        }

      - maintenance_options {
          - auto_recovery = "default" -> null
        }

      - metadata_options {
          - http_endpoint               = "enabled" -> null
          - http_protocol_ipv6          = "disabled" -> null
          - http_put_response_hop_limit = 2 -> null
          - http_tokens                 = "required" -> null
          - instance_metadata_tags      = "disabled" -> null
        }

      - private_dns_name_options {
          - enable_resource_name_dns_a_record    = false -> null
          - enable_resource_name_dns_aaaa_record = false -> null
          - hostname_type                        = "ip-name" -> null
        }

      - root_block_device {
          - delete_on_termination = true -> null
          - device_name           = "/dev/xvda" -> null
          - encrypted             = false -> null
          - iops                  = 3000 -> null
          - tags                  = {} -> null
          - tags_all              = {} -> null
          - throughput            = 125 -> null
          - volume_id             = "vol-0cc4904aec3eea0de" -> null
          - volume_size           = 30 -> null
          - volume_type           = "gp3" -> null
            # (1 unchanged attribute hidden)
        }
    }

  # aws_instance.app_ec2["red"] will be destroyed
  - resource "aws_instance" "app_ec2" {
      - ami                                  = "ami-0220226481c20c75f" -> null
      - arn                                  = "arn:aws:ec2:us-east-1:879433079150:instance/i-09a5e5642f47ea9e1" -> null
      - associate_public_ip_address          = true -> null
      - availability_zone                    = "us-east-1e" -> null
      - cpu_core_count                       = 1 -> null
      - cpu_threads_per_core                 = 1 -> null
      - disable_api_stop                     = false -> null
      - disable_api_termination              = false -> null
      - ebs_optimized                        = false -> null
      - get_password_data                    = false -> null
      - hibernation                          = false -> null
      - id                                   = "i-09a5e5642f47ea9e1" -> null
      - instance_initiated_shutdown_behavior = "stop" -> null
      - instance_state                       = "running" -> null
      - instance_type                        = "t2.micro" -> null
      - ipv6_address_count                   = 0 -> null
      - ipv6_addresses                       = [] -> null
      - monitoring                           = false -> null
      - placement_partition_number           = 0 -> null
      - primary_network_interface_id         = "eni-00b3f40ab44f4246a" -> null
      - private_dns                          = "ip-172-31-51-148.ec2.internal" -> null
      - private_ip                           = "172.31.51.148" -> null
      - public_dns                           = "ec2-107-22-142-9.compute-1.amazonaws.com" -> null
      - public_ip                            = "107.22.142.9" -> null
      - secondary_private_ips                = [] -> null
      - security_groups                      = [
          - "ec2_security_group",
        ] -> null
      - source_dest_check                    = true -> null
      - subnet_id                            = "subnet-0b28c02ae27327b22" -> null
      - tags                                 = {
          - "Name" = "AppServer-red"
        } -> null
      - tags_all                             = {
          - "Name" = "AppServer-red"
        } -> null
      - tenancy                              = "default" -> null
      - user_data                            = "3d25ff82455905e4fea90c79badb33b3bb1a621f" -> null
      - user_data_replace_on_change          = false -> null
      - vpc_security_group_ids               = [
          - "sg-0cbd155008d612114",
        ] -> null
        # (8 unchanged attributes hidden)

      - capacity_reservation_specification {
          - capacity_reservation_preference = "open" -> null
        }

      - cpu_options {
          - core_count       = 1 -> null
          - threads_per_core = 1 -> null
            # (1 unchanged attribute hidden)
        }

      - credit_specification {
          - cpu_credits = "standard" -> null
        }

      - enclave_options {
          - enabled = false -> null
        }

      - maintenance_options {
          - auto_recovery = "default" -> null
        }

      - metadata_options {
          - http_endpoint               = "enabled" -> null
          - http_protocol_ipv6          = "disabled" -> null
          - http_put_response_hop_limit = 2 -> null
          - http_tokens                 = "required" -> null
          - instance_metadata_tags      = "disabled" -> null
        }

      - private_dns_name_options {
          - enable_resource_name_dns_a_record    = false -> null
          - enable_resource_name_dns_aaaa_record = false -> null
          - hostname_type                        = "ip-name" -> null
        }

      - root_block_device {
          - delete_on_termination = true -> null
          - device_name           = "/dev/xvda" -> null
          - encrypted             = false -> null
          - iops                  = 3000 -> null
          - tags                  = {} -> null
          - tags_all              = {} -> null
          - throughput            = 125 -> null
          - volume_id             = "vol-0db4468783baac9fe" -> null
          - volume_size           = 30 -> null
          - volume_type           = "gp3" -> null
            # (1 unchanged attribute hidden)
        }
    }

  # aws_lb.app_alb will be destroyed
  - resource "aws_lb" "app_alb" {
      - arn                                                          = "arn:aws:elasticloadbalancing:us-east-1:879433079150:loadbalancer/app/app-alb/412e1705362e63be" -> null
      - arn_suffix                                                   = "app/app-alb/412e1705362e63be" -> null
      - client_keep_alive                                            = 3600 -> null
      - desync_mitigation_mode                                       = "defensive" -> null
      - dns_name                                                     = "app-alb-1465625505.us-east-1.elb.amazonaws.com" -> null
      - drop_invalid_header_fields                                   = false -> null
      - enable_cross_zone_load_balancing                             = true -> null
      - enable_deletion_protection                                   = false -> null
      - enable_http2                                                 = true -> null
      - enable_tls_version_and_cipher_suite_headers                  = false -> null
      - enable_waf_fail_open                                         = false -> null
      - enable_xff_client_port                                       = false -> null
      - enable_zonal_shift                                           = false -> null
      - id                                                           = "arn:aws:elasticloadbalancing:us-east-1:879433079150:loadbalancer/app/app-alb/412e1705362e63be" -> null
      - idle_timeout                                                 = 60 -> null
      - internal                                                     = false -> null
      - ip_address_type                                              = "ipv4" -> null
      - load_balancer_type                                           = "application" -> null
      - name                                                         = "app-alb" -> null
      - preserve_host_header                                         = false -> null
      - security_groups                                              = [
          - "sg-0cbd155008d612114",
        ] -> null
      - subnets                                                      = [
          - "subnet-013375bfa51beef27",
          - "subnet-0522e9e80d209bfc6",
          - "subnet-07ae9d7a2d9ae8f57",
          - "subnet-0b28c02ae27327b22",
          - "subnet-0d3fde3bd7076fc92",
          - "subnet-0d4cdbcfc2c29cf38",
        ] -> null
      - tags                                                         = {} -> null
      - tags_all                                                     = {} -> null
      - vpc_id                                                       = "vpc-0c68f0752df142156" -> null
      - xff_header_processing_mode                                   = "append" -> null
      - zone_id                                                      = "Z35SXDOTRQ7X7K" -> null
        # (3 unchanged attributes hidden)

      - access_logs {
          - enabled = false -> null
            # (2 unchanged attributes hidden)
        }

      - connection_logs {
          - enabled = false -> null
            # (2 unchanged attributes hidden)
        }

      - subnet_mapping {
          - subnet_id            = "subnet-013375bfa51beef27" -> null
            # (4 unchanged attributes hidden)
        }
      - subnet_mapping {
          - subnet_id            = "subnet-0522e9e80d209bfc6" -> null
            # (4 unchanged attributes hidden)
        }
      - subnet_mapping {
          - subnet_id            = "subnet-07ae9d7a2d9ae8f57" -> null
            # (4 unchanged attributes hidden)
        }
      - subnet_mapping {
          - subnet_id            = "subnet-0b28c02ae27327b22" -> null
            # (4 unchanged attributes hidden)
        }
      - subnet_mapping {
          - subnet_id            = "subnet-0d3fde3bd7076fc92" -> null
            # (4 unchanged attributes hidden)
        }
      - subnet_mapping {
          - subnet_id            = "subnet-0d4cdbcfc2c29cf38" -> null
            # (4 unchanged attributes hidden)
        }
    }

  # aws_lb_listener.http_listener will be destroyed
  - resource "aws_lb_listener" "http_listener" {
      - arn                                                                 = "arn:aws:elasticloadbalancing:us-east-1:879433079150:listener/app/app-alb/412e1705362e63be/10ef18abe2c2576f" -> null
      - id                                                                  = "arn:aws:elasticloadbalancing:us-east-1:879433079150:listener/app/app-alb/412e1705362e63be/10ef18abe2c2576f" -> null
      - load_balancer_arn                                                   = "arn:aws:elasticloadbalancing:us-east-1:879433079150:loadbalancer/app/app-alb/412e1705362e63be" -> null
      - port                                                                = 80 -> null
      - protocol                                                            = "HTTP" -> null
      - routing_http_response_server_enabled                                = false -> null
      - tags                                                                = {} -> null
      - tags_all                                                            = {} -> null
        # (11 unchanged attributes hidden)

      - default_action {
          - order            = 1 -> null
          - target_group_arn = "arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/3acf7f3e63ec56f5" -> null
          - type             = "forward" -> null
        }
    }

  # aws_lb_target_group.app_tg will be destroyed
  - resource "aws_lb_target_group" "app_tg" {
      - arn                                = "arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/3acf7f3e63ec56f5" -> null
      - arn_suffix                         = "targetgroup/app-target-group/3acf7f3e63ec56f5" -> null
      - deregistration_delay               = "300" -> null
      - id                                 = "arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/3acf7f3e63ec56f5" -> null
      - ip_address_type                    = "ipv4" -> null
      - lambda_multi_value_headers_enabled = false -> null
      - load_balancer_arns                 = [
          - "arn:aws:elasticloadbalancing:us-east-1:879433079150:loadbalancer/app/app-alb/412e1705362e63be",
        ] -> null
      - load_balancing_algorithm_type      = "round_robin" -> null
      - load_balancing_anomaly_mitigation  = "off" -> null
      - load_balancing_cross_zone_enabled  = "use_load_balancer_configuration" -> null
      - name                               = "app-target-group" -> null
      - port                               = 80 -> null
      - protocol                           = "HTTP" -> null
      - protocol_version                   = "HTTP1" -> null
      - proxy_protocol_v2                  = false -> null
      - slow_start                         = 0 -> null
      - tags                               = {} -> null
      - tags_all                           = {} -> null
      - target_type                        = "instance" -> null
      - vpc_id                             = "vpc-0c68f0752df142156" -> null
        # (1 unchanged attribute hidden)

      - health_check {
          - enabled             = true -> null
          - healthy_threshold   = 5 -> null
          - interval            = 30 -> null
          - matcher             = "200-399" -> null
          - path                = "/" -> null
          - port                = "traffic-port" -> null
          - protocol            = "HTTP" -> null
          - timeout             = 5 -> null
          - unhealthy_threshold = 2 -> null
        }

      - stickiness {
          - cookie_duration = 86400 -> null
          - enabled         = false -> null
          - type            = "lb_cookie" -> null
            # (1 unchanged attribute hidden)
        }

      - target_failover {}

      - target_group_health {
          - dns_failover {
              - minimum_healthy_targets_count      = "1" -> null
              - minimum_healthy_targets_percentage = "off" -> null
            }
          - unhealthy_state_routing {
              - minimum_healthy_targets_count      = 1 -> null
              - minimum_healthy_targets_percentage = "off" -> null
            }
        }

      - target_health_state {}
    }

  # aws_lb_target_group_attachment.tg_attachment["blue"] will be destroyed
  - resource "aws_lb_target_group_attachment" "tg_attachment" {
      - id               = "arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/3acf7f3e63ec56f5-20250116210559105300000006" -> null
      - port             = 80 -> null
      - target_group_arn = "arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/3acf7f3e63ec56f5" -> null
      - target_id        = "i-0c9539b7703295cf6" -> null
    }

  # aws_lb_target_group_attachment.tg_attachment["green"] will be destroyed
  - resource "aws_lb_target_group_attachment" "tg_attachment" {
      - id               = "arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/3acf7f3e63ec56f5-20250116210559032100000005" -> null
      - port             = 80 -> null
      - target_group_arn = "arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/3acf7f3e63ec56f5" -> null
      - target_id        = "i-0ee97ce094dc9582e" -> null
    }

  # aws_lb_target_group_attachment.tg_attachment["red"] will be destroyed
  - resource "aws_lb_target_group_attachment" "tg_attachment" {
      - id               = "arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/3acf7f3e63ec56f5-20250116210559004500000004" -> null
      - port             = 80 -> null
      - target_group_arn = "arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/3acf7f3e63ec56f5" -> null
      - target_id        = "i-09a5e5642f47ea9e1" -> null
    }

  # aws_security_group.ec2_sg will be destroyed
  - resource "aws_security_group" "ec2_sg" {
      - arn                    = "arn:aws:ec2:us-east-1:879433079150:security-group/sg-0cbd155008d612114" -> null
      - description            = "Security group for EC2 instances" -> null
      - egress                 = [
          - {
              - cidr_blocks      = [
                  - "0.0.0.0/0",
                ]
              - from_port        = 0
              - ipv6_cidr_blocks = []
              - prefix_list_ids  = []
              - protocol         = "-1"
              - security_groups  = []
              - self             = false
              - to_port          = 0
                # (1 unchanged attribute hidden)
            },
        ] -> null
      - id                     = "sg-0cbd155008d612114" -> null
      - ingress                = [
          - {
              - cidr_blocks      = [
                  - "0.0.0.0/0",
                ]
              - from_port        = 80
              - ipv6_cidr_blocks = []
              - prefix_list_ids  = []
              - protocol         = "tcp"
              - security_groups  = []
              - self             = false
              - to_port          = 80
                # (1 unchanged attribute hidden)
            },
        ] -> null
      - name                   = "ec2_security_group" -> null
      - owner_id               = "879433079150" -> null
      - revoke_rules_on_delete = false -> null
      - tags                   = {} -> null
      - tags_all               = {} -> null
      - vpc_id                 = "vpc-0c68f0752df142156" -> null
        # (1 unchanged attribute hidden)
    }

Plan: 0 to add, 0 to change, 10 to destroy.

Changes to Outputs:
  - alb_dns_name          = "app-alb-1465625505.us-east-1.elb.amazonaws.com" -> null
  - amazon_linux_2023_ami = "ami-0220226481c20c75f" -> null
  - first_vpc_id          = "vpc-0c68f0752df142156" -> null
  - subnet_ids            = [
      - "subnet-0d4cdbcfc2c29cf38",
      - "subnet-07ae9d7a2d9ae8f57",
      - "subnet-0b28c02ae27327b22",
      - "subnet-0522e9e80d209bfc6",
      - "subnet-013375bfa51beef27",
      - "subnet-0d3fde3bd7076fc92",
    ] -> null

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes
  
  aws_lb_target_group_attachment.tg_attachment["green"]: Destroying... [id=arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/3acf7f3e63ec56f5-20250116210559032100000005]
aws_lb_listener.http_listener: Destroying... [id=arn:aws:elasticloadbalancing:us-east-1:879433079150:listener/app/app-alb/412e1705362e63be/10ef18abe2c2576f]
aws_lb_target_group_attachment.tg_attachment["blue"]: Destroying... [id=arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/3acf7f3e63ec56f5-20250116210559105300000006]
aws_lb_target_group_attachment.tg_attachment["red"]: Destroying... [id=arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/3acf7f3e63ec56f5-20250116210559004500000004]
aws_lb_target_group_attachment.tg_attachment["green"]: Destruction complete after 0s
aws_lb_target_group_attachment.tg_attachment["blue"]: Destruction complete after 0s
aws_lb_target_group_attachment.tg_attachment["red"]: Destruction complete after 0s
aws_instance.app_ec2["green"]: Destroying... [id=i-0ee97ce094dc9582e]
aws_instance.app_ec2["blue"]: Destroying... [id=i-0c9539b7703295cf6]
aws_instance.app_ec2["red"]: Destroying... [id=i-09a5e5642f47ea9e1]
aws_lb_listener.http_listener: Destruction complete after 0s
aws_lb_target_group.app_tg: Destroying... [id=arn:aws:elasticloadbalancing:us-east-1:879433079150:targetgroup/app-target-group/3acf7f3e63ec56f5]
aws_lb.app_alb: Destroying... [id=arn:aws:elasticloadbalancing:us-east-1:879433079150:loadbalancer/app/app-alb/412e1705362e63be]
aws_lb_target_group.app_tg: Destruction complete after 0s
aws_lb.app_alb: Destruction complete after 2s
aws_instance.app_ec2["green"]: Still destroying... [id=i-0ee97ce094dc9582e, 10s elapsed]
aws_instance.app_ec2["blue"]: Still destroying... [id=i-0c9539b7703295cf6, 10s elapsed]
aws_instance.app_ec2["red"]: Still destroying... [id=i-09a5e5642f47ea9e1, 10s elapsed]
aws_instance.app_ec2["green"]: Still destroying... [id=i-0ee97ce094dc9582e, 20s elapsed]
aws_instance.app_ec2["blue"]: Still destroying... [id=i-0c9539b7703295cf6, 20s elapsed]
aws_instance.app_ec2["red"]: Still destroying... [id=i-09a5e5642f47ea9e1, 20s elapsed]
aws_instance.app_ec2["green"]: Still destroying... [id=i-0ee97ce094dc9582e, 30s elapsed]
aws_instance.app_ec2["blue"]: Still destroying... [id=i-0c9539b7703295cf6, 30s elapsed]
aws_instance.app_ec2["red"]: Still destroying... [id=i-09a5e5642f47ea9e1, 30s elapsed]
aws_instance.app_ec2["blue"]: Destruction complete after 30s
aws_instance.app_ec2["green"]: Still destroying... [id=i-0ee97ce094dc9582e, 40s elapsed]
aws_instance.app_ec2["red"]: Still destroying... [id=i-09a5e5642f47ea9e1, 40s elapsed]
aws_instance.app_ec2["red"]: Destruction complete after 41s
aws_instance.app_ec2["green"]: Still destroying... [id=i-0ee97ce094dc9582e, 50s elapsed]
aws_instance.app_ec2["green"]: Still destroying... [id=i-0ee97ce094dc9582e, 1m0s elapsed]
aws_instance.app_ec2["green"]: Still destroying... [id=i-0ee97ce094dc9582e, 1m10s elapsed]
aws_instance.app_ec2["green"]: Still destroying... [id=i-0ee97ce094dc9582e, 1m20s elapsed]
aws_instance.app_ec2["green"]: Destruction complete after 1m21s
aws_security_group.ec2_sg: Destroying... [id=sg-0cbd155008d612114]
aws_security_group.ec2_sg: Destruction complete after 0s

Destroy complete! Resources: 10 destroyed.

```



### Notes for Students

- **State Management:** Terraform maintains a state file (`terraform.tfstate`). Avoid manually editing this file.



### Challenges for Advanced Students

1. Modify the configuration to:
   - Add an HTTPS listener to the ALB.
   - Enable autoscaling for the EC2 instances.

### Conclusion

This lab demonstrates the power of Terraform for deploying and managing AWS infrastructure.
