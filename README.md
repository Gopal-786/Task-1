provider "aws" {
              profile = "tfgopal"
              region = "ap-south-1"
               } 

resource "tls_private_key" "my-key"{
           algorithm = "RSA"
           rsa_bits = "4096"
            }
resource "aws_key_pair" "generate_key" {
           key_name = "mytfkey1"
           public_key = "${tls_private_key.my-key.public_key_openssh}"
 
 depends_on = [ 
                   tls_private_key.my-key
                ]
    }
 
resource "local_file" "key-file" {
         content = "${tls_private_key.my-key.private_key_pem}"
         filename = "mytfkey1.pem"

   depends_on = [
             tls_private_key.my-key
                ]
}

resource "aws_security_group" "sec_group" {
name = "sec_group"
description = "allow ssh and HTTPD"

 ingress {
           description = "SSH Port"
           from_port =22
           to_port = 22
           protocol = "tcp"
           cidr_blocks = ["0.0.0.0/0"]
        }
 ingress {
            
           description = "HTTPD"
           from_port = 80
           to_port = 80
           protocol = "tcp"
           cidr_blocks = ["0.0.0.0/0"]
          } 
 egress {
     
           from_port = 0
           to_port = 0
           protocol = "-1"
           cidr_blocks = ["0.0.0.0/0"]
       } 
  tags = {
            Name = "sec_group"
          }
 }
 
resource "aws_instance" "web" {
 ami = "ami-0447a12f28fddb066"
 instance_type = "t2.micro"
 key_name = aws_key_pair.generate_key.key_name
 security_groups = [ "sec_group"]

provisioner "remote-exec" {
    connection {
    type = "ssh"
    user = "ec2-user"
    private_key = "${tls_private_key.my-key.private_key_pem}"
    host = "${aws_instance.web.public_ip}"
}

  inline = [
            "sudo yum install httpd git -y",
            "sudo systemctl restart httpd" ,
            "sudo systemctl enable httpd" ,
            ]
}
  tags = {
          Name = "Gopal_OS"
  }
}
 
 resource "aws_ebs_volume" "ebs11" {
  availability_zone = aws_instance.web.availability_zone
  size = 1
   tags = {
    Name = "vol_web"
          }
}
 
 resource "aws_volume_attachment" "ebs_attach"{
  device_name = "/dev/sdh"
  volume_id = aws_ebs_volume.ebs11.id
  instance_id = aws_instance.web.id
  force_detach = true
  }
  
 output "myip" {
    value = aws_instance.web.public_ip
  
}

resource "null_resource" "nullip" {
  provisioner "local-exec" {
  command = "echo ${aws_instance.web.public_ip} > publicip.txt "
    }
}

 resource "null_resource" "nullmount" {
   depends_on = [
                 aws_volume_attachment.ebs_attach,
]

 connection {
    type = "ssh"
    user = "ec2-user"
    private_key = "${tls_private_key.my-key.private_key_pem}"
    host = "${aws_instance.web.public_ip}"
   }
 
  provisioner "remote-exec" {
   inline = [ 
    "sudo mkfs.ext4 /dev/xvdh" ,
    "sudo mount /dev/xvdh /var/www/html",
    "sudo rm -rf /var/www/html/*",
    "sudo git clone https://github.com/Gopal-786/Gopal-Jaiswal.git"
   ]
      }
}
 
/*
resource "aws_s3_bucket"  "terra-bucket1" {
   bucket = "Gopalsbucket"
   acl = "public-read"

  versioning { 
         enabled = true 
      }
   
  tags = { 
        Name = "my-terra-bucket1" 
        Environment = "Dev"
         }
}

   
 resource "aws_cloudfront_distribution" "terra-cloudfront1"
   origin {
            domain_name = "Gopalsbucket.s3.amazonaws.com"
            origin_id = "S3-Gopalsbucket"

          custom_origin_config {
           http_port = 80
           origin_protocol_policy = "match-viewer"
           origin_ssl_protocols = ["TLSv1", "TLSv1.1", "TLSv1.2"]
         }
}

 enabled = true
   
 default_cache_behavior {
     allowed_methods = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "PUT"]
     cached-methods = ["GET" , "HEAD" ]
     target_origin-id = "S3-Gopalsbucket"
     
   forwarded_values {
            query_string = false 

            cookies {
                  forward = "none"
             }
}
 
 viewer_protocol_policy = "allow-all"
 min_ttl = 0
 default_ttl = 3600
 max_ttl = 86400

 }

 restrictions { 
         geo_restriction {
                      restriction_type = "none"
              } 
 }

 viewer_certificate {
              cloudfront_default_certification = true
}

resource "null_resource" "nullremote" {
   depends_on = [ 
    null_resource.nullmount
     ]

}
 */
