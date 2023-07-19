# IaaC
Zaliczenie_35138
variable "vpc_cidr_block" {
  description = "CIDR bloku VPC"
}

variable "private_subnet_cidr_blocks" {
  type        = list(string)
  description = "Lista CIDR bloków dla podsieci prywatnych"
}

variable "public_subnet_cidr_blocks" {
  type        = list(string)
  description = "Lista CIDR bloków dla podsieci publicznych"
}

resource "aws_vpc" "my_vpc" {
  cidr_block = var.vpc_cidr_block
}

resource "aws_subnet" "private_subnets" {
  count             = length(var.private_subnet_cidr_blocks)
  vpc_id            = aws_vpc.my_vpc.id
  cidr_block        = var.private_subnet_cidr_blocks[count.index]
  availability_zone = "us-east-1a"  # Zmień to na odpowiednią strefę dostępności
}

resource "aws_subnet" "public_subnets" {
  count             = length(var.public_subnet_cidr_blocks)
  vpc_id            = aws_vpc.my_vpc.id
  cidr_block        = var.public_subnet_cidr_blocks[count.index]
  availability_zone = "us-east-1a"  # Zmień to na odpowiednią strefę dostępności
}

resource "aws_eip" "nat_gateway_eip" {
  count = length(var.public_subnet_cidr_blocks)
}

resource "aws_nat_gateway" "nat_gateway" {
  count          = length(var.public_subnet_cidr_blocks)
  subnet_id      = aws_subnet.public_subnets[count.index].id
  allocation_id  = aws_eip.nat_gateway_eip[count.index].id
}

resource "aws_route_table" "private_route_table" {
  vpc_id = aws_vpc.my_vpc.id

  dynamic "route" {
    for_each = aws_subnet.private_subnets
    content {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.nat_gateway[0].id
      subnet_id      = route.value.id
    }
  }
}

resource "aws_route_table_association" "private_subnet_association" {
  count          = length(aws_subnet.private_subnets)
  subnet_id      = aws_subnet.private_subnets[count.index].id
  route_table_id = aws_route_table.private_route_table.id
}

output "vpc_id" {
  value = aws_vpc.my_vpc.id
}

output "private_subnet_ids" {
  value = aws_subnet.private_subnets[*].id
}

output "public_subnet_ids" {
  value = aws_subnet.public_subnets[*].id
}
