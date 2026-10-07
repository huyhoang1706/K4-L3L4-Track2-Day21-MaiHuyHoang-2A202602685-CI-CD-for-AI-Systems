# Bước 2 trên AWS: DVC, S3, EC2 và GitHub Actions

Tài liệu này thay thế các lệnh GCP của Bước 2 bằng AWS. Pipeline trong repo
đã dùng S3 (`boto3`, `dvc[s3]`) và EC2 qua SSH.

## 1. Biến môi trường và S3 bucket

Chọn một region và tên bucket duy nhất toàn cầu. Không commit access key vào repo.

```bash
export AWS_REGION=ap-southeast-1
export BUCKET=<ten-bucket-duy-nhat>
aws sts get-caller-identity

# ap-southeast-1 cần LocationConstraint. Với us-east-1, bỏ tham số này.
aws s3api create-bucket \
  --bucket "$BUCKET" \
  --region "$AWS_REGION" \
  --create-bucket-configuration LocationConstraint="$AWS_REGION"

aws s3api put-public-access-block --bucket "$BUCKET" \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

## 2. IAM tối thiểu

Tạo một IAM user dành riêng cho GitHub Actions, tạo access key, rồi cấp policy chỉ
trên bucket lab. Thay `REGION`, `ACCOUNT_ID`, `BUCKET_NAME` trước khi tạo policy.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket", "s3:GetBucketLocation"],
      "Resource": "arn:aws:s3:::BUCKET_NAME"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": [
        "arn:aws:s3:::BUCKET_NAME/dvc/*",
        "arn:aws:s3:::BUCKET_NAME/artifacts/*"
      ]
    }
  ]
}
```

EC2 không dùng access key trên đĩa. Gắn instance profile có quyền tối thiểu chỉ đọc
`arn:aws:s3:::BUCKET_NAME/artifacts/current/model.joblib` (`s3:GetObject`) cho EC2.

## 3. DVC remote S3

```bash
dvc init
dvc remote add -d labstore "s3://$BUCKET/dvc"

dvc add data/train_batch1.csv data/holdout.csv data/train_batch2.csv
dvc push

git add .dvc/config .gitignore data/*.dvc
git commit -m "feat: track datasets with DVC on S3"
```

Máy local dùng profile AWS (`aws configure`) hoặc các biến `AWS_ACCESS_KEY_ID`,
`AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`. Xác nhận remote bằng
`aws s3 ls "s3://$BUCKET/dvc/" --recursive`.

## 4. EC2 và Security Group

Tạo key pair EC2, lấy Ubuntu AMI chính thức qua SSM, sau đó tạo một EC2 Ubuntu nhỏ
trong default VPC/subnet. Security group phải cho phép TCP 8080 từ IP công khai của
bạn. Port 22 chỉ nên mở cho IP cần SSH; nếu GitHub Actions deploy qua SSH, runner
cần truy cập port này, vì vậy dùng SSH key riêng và giới hạn rule lại sau lab.

```bash
export MY_IP="$(curl -s https://checkip.amazonaws.com)/32"
export AMI_ID="$(aws ssm get-parameter \
  --name /aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id \
  --region "$AWS_REGION" --query 'Parameter.Value' --output text)"

aws ec2 create-key-pair --key-name income-api-key \
  --query KeyMaterial --output text > income-api-key.pem
chmod 400 income-api-key.pem

VPC_ID=$(aws ec2 describe-vpcs --filters Name=is-default,Values=true \
  --query 'Vpcs[0].VpcId' --output text --region "$AWS_REGION")
SG_ID=$(aws ec2 create-security-group --group-name income-api-sg \
  --description 'Income inference API' --vpc-id "$VPC_ID" \
  --query GroupId --output text --region "$AWS_REGION")
aws ec2 authorize-security-group-ingress --group-id "$SG_ID" \
  --protocol tcp --port 8080 --cidr "$MY_IP" --region "$AWS_REGION"
aws ec2 authorize-security-group-ingress --group-id "$SG_ID" \
  --protocol tcp --port 22 --cidr "$MY_IP" --region "$AWS_REGION"

INSTANCE_ID=$(aws ec2 run-instances --image-id "$AMI_ID" --instance-type t3.micro \
  --key-name income-api-key --security-group-ids "$SG_ID" \
  --iam-instance-profile Name=<EC2_S3_READ_ROLE> \
  --query 'Instances[0].InstanceId' --output text --region "$AWS_REGION")
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID" --region "$AWS_REGION"
VM_IP=$(aws ec2 describe-instances --instance-ids "$INSTANCE_ID" --region "$AWS_REGION" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
```

## 5. Cài API và systemd trên EC2

```bash
ssh -i income-api-key.pem ubuntu@"$VM_IP"
sudo apt update && sudo apt install -y python3-pip
pip3 install fastapi uvicorn scikit-learn joblib boto3
mkdir -p ~/models ~/src
exit

scp -i income-api-key.pem src/serve.py ubuntu@"$VM_IP":~/src/serve.py
ssh -i income-api-key.pem ubuntu@"$VM_IP"
```

Trên EC2, tạo `/etc/systemd/system/income-api.service` (thay bucket):

```ini
[Unit]
Description=Income Model Inference Server
After=network-online.target
Wants=network-online.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu
Environment="ARTIFACT_BUCKET=BUCKET_NAME"
Environment="AWS_DEFAULT_REGION=ap-southeast-1"
ExecStart=/usr/bin/python3 /home/ubuntu/src/serve.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable income-api
```

Chưa start service trước lần pipeline đầu tiên vì model chưa ở S3.

## 6. GitHub Secrets và pipeline

Thêm các secrets sau vào GitHub Actions:

| Secret | Giá trị |
|---|---|
| `AWS_ACCESS_KEY_ID` | Access key của IAM user cho Actions |
| `AWS_SECRET_ACCESS_KEY` | Secret access key tương ứng |
| `AWS_REGION` | Ví dụ `ap-southeast-1` |
| `ARTIFACT_BUCKET` | Tên S3 bucket, không có `s3://` |
| `SERVER_HOST` | `VM_IP` của EC2 |
| `SERVER_USER` | `ubuntu` cho Ubuntu AMI |
| `SERVER_SSH_KEY` | Private key deploy riêng cho GitHub Actions |

Tạo SSH key deploy riêng, thêm public key vào EC2 và copy source:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/income_deploy -N "" -C "github-actions-deploy"
ssh -i income-api-key.pem ubuntu@"$VM_IP" \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && echo '$(cat ~/.ssh/income_deploy.pub)' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Sau khi DVC remote và secrets đã sẵn sàng, push lên `main`. Bốn jobs sẽ chạy theo
thứ tự Unit Test -> Train -> Quality Gate -> Release. Khi release xanh, thử:

```bash
curl "http://$VM_IP:8080/healthz"
curl -X POST "http://$VM_IP:8080/score" -H 'Content-Type: application/json' \
  -d '{"features":[60,2,5,2,4,0,1,0,0,45]}'
```
