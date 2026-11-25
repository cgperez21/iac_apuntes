## outputs 2
```` hcl
output "nlb_dns" { value = aws_lb.nlb.dns_name }
output "rds_endpoint" { value = aws_db_instance.db.endpoint }
output "bucket_name" { value = aws_s3_bucket.material.bucket }
output "ips_servidores" { value = { for k, s in aws_instance.servidor : k => s.public_ip } }
````
