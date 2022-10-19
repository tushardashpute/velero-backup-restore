# EKS cluster backup and restor using Velero

**Install jq:**

    yum install jq -y

**Provide AWS_REGION=**

    export AWS_REGION=us-east-2

**Create an S3 bucket to backup cluster:**

    export VELERO_BUCKET=$(aws s3api create-bucket \
    --bucket eksworkshop-backup-$(date +%s)-$RANDOM \
    --region $AWS_REGION \
    --create-bucket-configuration LocationConstraint=$AWS_REGION \
    --| jq -r '.Location' \
    --| cut -d'/' -f3 \
    --| cut -d'.' -f1)

For us-east-1, use the command below to create S3 bucket.

    export VELERO_BUCKET=$(aws s3api create-bucket \
    --bucket eksworkshop-backup-$(date +%s)-$RANDOM \
    --region $AWS_REGION \
    --| jq -r '.Location' \
    --| tr -d /)  

Now, let’s save the VELERO_BUCKET environment variable into the bash_profile

    echo "export VELERO_BUCKET=${VELERO_BUCKET}" | tee -a ~/.bash_profile

**Create policies to give velero the necessary permissions:**

    cat > velero-policy.json <<EOF
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "ec2:DescribeVolumes",
                    "ec2:DescribeSnapshots",
                    "ec2:CreateTags",
                    "ec2:CreateVolume",
                    "ec2:CreateSnapshot",
                    "ec2:DeleteSnapshot"
                ],
                "Resource": "*"
            },
            {
                "Effect": "Allow",
                "Action": [
                    "s3:GetObject",
                    "s3:DeleteObject",
                    "s3:PutObject",
                    "s3:AbortMultipartUpload",
                    "s3:ListMultipartUploadParts"
                ],
                "Resource": [
                    "arn:aws:s3:::${VELERO_BUCKET}/*"
                ]
            },
            {
                "Effect": "Allow",
                "Action": [
                    "s3:ListBucket"
                ],
                "Resource": [
                    "arn:aws:s3:::${VELERO_BUCKET}"
                ]
            }
        ]
      }
    EOF
                               
**Create policy to velero IAM User:**

    aws iam create-policy \ 
        --policy-name VeleroAccessPolicy \
        --policy-document file://velero_policy.json

**Create Service Accounts for Velero**

    CLUSTER=eksdemo
    ACCOUNT=$(aws sts get-caller-identity --query Account --output text)

    eksctl create iamserviceaccount \
    --cluster=$CLUSTER \
    --name=velero-server \
    --namespace=velero \
    --role-name=eks-velero-backup \
    --role-only \
    --attach-policy-arn=arn:aws:iam::$ACCOUNT:policy/VeleroAccessPolicy \
    --approve
    
The --namespace=velero flag ensures that only an application running in the velero namespace will be able to access the IAM Policy created in the previous step.
  
**Install Velero on EKS:** 

    helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts

    cat > values.yaml <<EOF
    configuration:
      backupStorageLocation:
        bucket: $VELERO_BUCKET
      provider: aws
      volumeSnapshotLocation:
        config:
          region: $AWS_REGION
    credentials:
      useSecret: false
    initContainers:
    - name: velero-plugin-for-aws
      image: velero/velero-plugin-for-aws:v1.2.0
      volumeMounts:
      - mountPath: /target
        name: plugins
    serviceAccount:
      server:
        annotations:
          eks.amazonaws.com/role-arn: "arn:aws:iam::${ACCOUNT}:role/eks-velero-backup"
    EOF

**Now install velero using helm:**

     helm install velero vmware-tanzu/velero \
        --create-namespace \
        --namespace velero \
        -f values.yaml

kubectl get pods –n velero

     kubectl get pods -n velero
    NAME                     READY   STATUS    RESTARTS   AGE
    velero-67c78d584-94n46   1/1     Running   0          39s
    
**Install the Velero CLI**

**Mac:**

    brew install velero

**Linux:**

    wget https://github.com/vmware-tanzu/velero/releases/download/v1.8.1/velero-v1.8.1-linux-amd64.tar.gz
    tar -xvf velero-v1.8.1-linux-amd64.tar.gz
    mv velero /usr/local/bin/velero

**Backup & Restore**

Before performing manual backup and restore let’s create a deployment on the cluster:

    kubectl create ns nginx
    kubectl create deployment nginx --image=nginx -n nginx
    kubectl get deploy -n nginx

<img width="647" alt="image" src="https://user-images.githubusercontent.com/68885738/196746088-bea7cfb4-1f3f-4b87-a222-97c99f801780.png">

    kubectl create ns nginx1
    kubectl create deployment nginx --image=nginx -n nginx1
    kubectl get deploy -n nginx1
    
 <img width="678" alt="image" src="https://user-images.githubusercontent.com/68885738/196746367-a213e7d7-9000-4170-800a-7345927c3993.png">


**Now let’s perform a manual backup:**

velero backup create demo1

<img width="444" alt="image" src="https://user-images.githubusercontent.com/68885738/196746682-0a7a7ff4-1c6b-4775-ae3a-722189836555.png">

Let’s confirm if the backup is complete:

    # velero backup get
    NAME    STATUS      CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
    demo1    Completed   2022-10-19 14:59:27 +0000 UTC   29d       default            <none>
    
Let’s also confirm if the backup is stored in S3.

<img width="1344" alt="image" src="https://user-images.githubusercontent.com/68885738/196747090-af3c372a-855c-4a06-a480-27479cb7933f.png">

Now let’s delete the nginx and nginx1 namesapce we created earlier and then perform a restore to confirm if we will be safe during uncertain times.

Lets first try to restore only one ns "nginx1" from backup.

    kubectl delete ns nginx nginx1

<img width="460" alt="image" src="https://user-images.githubusercontent.com/68885738/196747575-998f3cde-c506-4f07-a5a3-807c9fbfa1e7.png">

**Restoring the backup:**

velero restore create --from-backup demo1 --include-namespaces nginx1

<img width="835" alt="image" src="https://user-images.githubusercontent.com/68885738/196747838-1586dc7b-ef7f-4517-9e8c-cbe94083d386.png">

If we run same command without "--include-namespace" filter, it will restore all data.

velero restore create --from-backup demo1

<img width="811" alt="image" src="https://user-images.githubusercontent.com/68885738/196748644-c04ce1c4-a75e-49a6-b322-a330a90dc27d.png">

Reference: https://velero.io/docs/v1.4/resource-filtering/ 
