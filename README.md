
# Create and connect to Public RDS with AWS 



#### Host a network on the cloud that has a database resource that can be accessed through a public ip address
#### Author @elijahizelllogan@gmail.com



## Deployment

To deploy this project run

```bash
 terraform init 
 terraform plan 
 terraform apply --auto-approve
```


## Resources

## VPC: 

	Computer network all resources will exist within 
## Subnet 
	Segmentation of network to relieve congestion and improve performance + security
	Using subset of network CIDR 
## Route table:
	Directs traffic within VPC between resources and from the internet 

## Internet Gateway (IGW) 
	Makes possible communication to the internet from within a VPC 

## Security Groups:
	A Virtual firewall for individual components  

# PROCESS 
	

### Create vpc 
Set up name and ipv4.   
Name: demo   
Ipv4: /22    < - just over a thousand   
192.168.0.0/22. 

```terraform
resource "aws_vpc" "demo" {
 cidr_block       = "192.168.0.0/22"
 instance_tenancy = "default"
 enable_dns_hostnames = true
 enable_dns_support = true
 tags = {
   Name = "demo"
 }
}
```

### create two subnets 
**two are required to create an rds instance**

```terraform
resource "aws_subnet" "demo-pub-a" {
 vpc_id = aws_vpc.demo.id
 cidr_block = "192.168.0.0/25"
 availability_zone = "us-east-1a"
 tags = {
   Name = "demo-pub-a"
 }
}




resource "aws_subnet" "demo-pub-b" {
 vpc_id = aws_vpc.demo.id
 cidr_block = "192.168.1.0/25"
 availability_zone = "us-east-1b"
 tags = {
   Name = "demo-pub-a"
 }
}
```


### Create internet Gateway and connect to VPC

```terraform
resource "aws_internet_gateway" "demo-gw" {
 vpc_id = aws_vpc.demo.id


 tags = {
   Name = "demo-gw"
 }
}
```
### Create route table and connect it to public subnets 

```terraform
resource "aws_route_table" "pub" {
 vpc_id = aws_vpc.demo.id
 route {
 cidr_block = "0.0.0.0/0"
 gateway_id = aws_internet_gateway.demo-gw.id
}
   tags = {
   Name = "pub"
 }
}
```

### Associate the public subnets to  the route table that points to the IGW 

```terraform
resource "aws_route_table_association" "public_association-a" {
 subnet_id      = aws_subnet.demo-pub-a.id
 route_table_id = aws_route_table.pub.id
}


resource "aws_route_table_association" "public_association-b" {
 subnet_id      = aws_subnet.demo-pub-b.id
 route_table_id = aws_route_table.pub.id
}
```

### create security Group for RDS instance

```terraform
resource "aws_security_group" "sports" {
 vpc_id = "${aws_vpc.demo.id}"
 name = "sports"
 description = "Allow all inbound for instance"


 ingress {
   from_port = 3306
   to_port = 3306
   protocol = "tcp"
   cidr_blocks = ["0.0.0.0/0"]
 }


   egress {
   from_port        = 0
   to_port          = 0
   protocol         = "-1"
   cidr_blocks      = ["0.0.0.0/0"]
   }
 # Allow all outbound traffic.
}

```


### create Subnet Group
** necessary to create an rds instance



```terraform
resource "aws_db_subnet_group" "default" {
 name       = "main"
 subnet_ids = [aws_subnet.demo-pub-a.id, aws_subnet.demo-pub-b.id]


 tags = {
   Name = "My DB subnet group"
 }
}
```

### Create  mysql instance in rds 


```terraform
resource aws_db_instance "db-name" {
   identifier =  "db-name"
   allocated_storage = 5
   db_name =  "data-name"
   engine = "mysql"
   engine_version = "5.7"
   instance_class = "db.t3.micro"
   username = "db-user"
   publicly_accessible = true
   password = "db-password"
   skip_final_snapshot = true
   multi_az               = false
   vpc_security_group_ids =  [aws_security_group.sports.id]
   db_subnet_group_name = aws_db_subnet_group.default.id
   apply_immediately = true
   port = 3306
}

```