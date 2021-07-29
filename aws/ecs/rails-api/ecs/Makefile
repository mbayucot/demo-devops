# VPC
REGION = us-east-2
SUBNETS = subnet-xxx,subnet-xxx,subnet-xxx
VPC_SUBNETS = subnet-xxx subnet-xxx subnet-xxx
VPC = vpc-xxx

# IAM
INSTANCE_ROLE = ecsInstanceRole

# ECR
ECR_REPO = xxx.dkr.ecr.us-east-2.amazonaws.com

# EC2
SECURITY_GROUP = sg-xxx
ELB_NAME = xxx-demo-web-https
ELB = arn:aws:elasticloadbalancing:us-east-2:xxx:loadbalancer/app/xxx-demo-web-https/xxx
TARGET_GROUP_NAME = xxx-demo-web-https-tg
TARGET_GROUP = arn:aws:elasticloadbalancing:us-east-2:xxx:targetgroup/xxx-demo-web-https-tg/xxx

# ECS
CLUSTER = xxx-app
PROJECT_NAME = demo-web
WORK_DIR = ~/xxx
CONTAINER_NAME = nginx
DESIRED_COUNT = 2

# ACM
PROD_ACM = arn:aws:acm:us-east-2:xxx:certificate/xxx


# ECR
push-images:
	docker login -u AWS -p $(aws ecr get-login-password --region us-east-2) xxx.dkr.ecr.us-east-2.amazonaws.com
	make push-web
	make push-nginx

push-web:
	make build-web
	docker push $(ECR_REPO)/xxx-app/web

push-nginx:
	make build-nginx
	docker push $(ECR_REPO)/xxx-app/nginx

build-images:
	make build-web
	make build-nginx

build-web:
	cd $(WORK_DIR)/xxx && docker build -t xxx-demo-web .
	docker tag xxx-demo-web:latest $(ECR_REPO)/xxx-app/web:latest

build-nginx:
	cd $(WORK_DIR)/xxx/nginx && docker build -t xxx-app-nginx .
	docker tag xxx-app-nginx:latest $(ECR_REPO)/xxx-app/nginx:latest

# ELB
create-elb:
	aws elbv2 create-load-balancer --name $(ELB_NAME) --subnets $(VPC_SUBNETS) --security-groups $(SECURITY_GROUP)

create-elb-listener:
	aws elbv2 create-listener --load-balancer-arn $(ELB) --protocol HTTP --port 80 --default-actions Type=redirect,RedirectConfig={Protocol=HTTPS,Port=443,Host='#{host}',Query='#{query}',Path='/#{path}',StatusCode=HTTP_301}
	aws elbv2 create-listener --load-balancer-arn $(ELB) --protocol HTTPS --port 443 --default-actions Type=forward,TargetGroupArn=$(TARGET_GROUP) --certificates CertificateArn=$(PROD_ACM)
	aws elbv2 modify-load-balancer-attributes --load-balancer-arn $(ELB) --attributes Key=idle_timeout.timeout_seconds,Value=5

INSTANCE_IDS = `aws ec2 describe-instances --query 'Reservations[].Instances[].InstanceId' --filters 'Name=tag:Name,Values=*xxx-app' 'Name=instance-state-name,Values=running' --output text`

register-elb:
	for INSTANCE_ID in $(INSTANCE_IDS) ; do \
  		aws elbv2 register-targets --target-group-arn $(TARGET_GROUP) --targets Id=$$INSTANCE_ID; \
  	done

# Target Group
create-target-group:
	aws elbv2 create-target-group --name $(TARGET_GROUP_NAME) --protocol HTTPS --port 443 --target-type instance --vpc-id $(VPC) --health-check-path=/health_check --health-check-port=443 --health-check-interval-seconds=30 --health-check-timeout-seconds=5 --healthy-threshold-count=10 --unhealthy-threshold-count=2 --matcher HttpCode=200,301

# ECS
configure-cluster:
	ecs-cli configure --cluster $(CLUSTER) --region $(REGION) --default-launch-type EC2 --config-name $(CLUSTER)

create-cluster:
	ecs-cli up --keypair aws-xxx --subnets $(SUBNETS) --security-group $(SECURITY_GROUP) --vpc $(VPC) --instance-role $(INSTANCE_ROLE) --size $(DESIRED_COUNT) --instance-type t2.small --cluster-config $(CLUSTER)

start-cluster:
	ecs-cli compose --project-name $(PROJECT_NAME) service up --create-log-groups --cluster-config $(CLUSTER) --target-group-arn $(TARGET_GROUP) --deployment-max-percent 100 --deployment-min-healthy-percent 50 --container-name $(CONTAINER_NAME) --container-port 443
	ecs-cli compose --project-name $(PROJECT_NAME) --cluster-config $(CLUSTER) service scale $(DESIRED_COUNT)

deploy:
	aws ecs update-service --cluster $(CLUSTER) --service $(PROJECT_NAME) --force-new-deployment

down-cluster:
	aws ecs update-service --cluster $(CLUSTER) --service $(PROJECT_NAME) --desired-count 0
	aws ecs delete-service --cluster $(CLUSTER) --service $(PROJECT_NAME)
	ecs-cli down --force --cluster-config $(CLUSTER)