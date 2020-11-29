# 인프라 구축

# 1. EKS 클러스터 구축

![%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/Untitled.png](%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/Untitled.png)

### Prerequisites

- **bastion server**
    - AWS 계정 권한을 가지고 CLI 환경으로 AWS Application들을 사용하기 위한 역할
    - 네트워크가 지원되는 어떤 컴퓨터든 상관 없고, 리소스도 최소한도만 있어도 됨. 다만, 공동 작업이 필요한 경우 다중접속이 가능해야 함
    - AWS의 모든 서비스 생성 권한을 가지고 있기 때문에 보안이 매우 중요
    - Amazon Linux AMI, Seoul region ap-northeast-2
- **Kubectl**
    - CLI환경에서 Kubernetes 클러스터를 구성 및 application 배포, 검사, 리소스 관리, 로그 확인 등의 기능
    - 노드 간 마이너 버전 차이가 0.2이상 나면 오류 발생할 수 있음
    - 예를 들어, master 노드에서 v1.2를 사용하면, 다른 노드에서는 v1.1 ~ v1.3 사용 가능
    1. [최신 Stable 버전](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl)을 다운로드한다.

    ```bash
    curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.17.0/bin/linux/amd64/kubectl
    ```

    2. kubectl binary 파일에 실행권한을 부여한다.

    ```bash
    chmod +x ./kubectl
    ```

    3. kubectl binary 파일을 바이너리 폴더로 이동시킨다.

    ```bash
    sudo mv ./kubectl /usr/local/bin/kubectl
    ```

    4. kubectl 명령어로 버전을 확인한다.

    ```bash
    kubectl version --client
    ```

- **AWS CLI 설정**
    1. AWS CLI2를 설치한다.

    ```bash
    cd ~

    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

    unzip awscliv2.zip
    ```

    2.  AWS CLI를 실행한다. 

    ```bash
    aws configure
    ```

    3. Access Keys를 입력한다.

    - AWS Management Console에 root 권한으로 로그인 한다.
    - 우측 상단의 네비게이션 바 - 계정 - 내 보안자격 증명 - 액세스 키 - 새 액세스 키 만들기(최초)
    - 생성된 ID:PW가 저장된 Key File.csv 파일 저장

    ```bash
    AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
    AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
    Default region name [None]: ap-northeast-2
    Default output format [None]: json
    ```

- **aws-iam-authenticator**
    - AWS EKS가 IAM을 사용하여 aws-iam-authenticator를 통해 Kubernetes 클러스터에 인증을 제공
    1. AWS S3로부터 aws-iam-authenticator binary 파일을 설치한다.

    ```bash
    curl -o aws-iam-authenticator https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.9/2020-11-02/bin/linux/amd64/aws-iam-authenticator
    ```

    2. aws-iam-authenticator binary 파일에 실행권한을 부여한다.

    ```bash
    chmod +x ./aws-iam-authenticator
    ```

    3. aws-iam-authenticator binary 파일을 $HOME/bin에 위치시킨다.

    ```bash
    mkdir -p $HOME/bin && cp ./aws-iam-authenticator $HOME/bin/aws-iam-authenticator && export PATH=$PATH:$HOME/bin
    ```

    4. .bash_profile에 환경변수를 추가한다.

    ```bash
    echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
    ```

    5. aws-iam-authenticator가 실행되는지 확인한다.

    ```bash
    aws-iam-authenticator help
    ```

- **eksctl**
    - EKS에 클러스터를 구축하는데 필요한 CLI tool
    - CloudFormation에 사용
    1. 최신 버전을 설치한다.

    ```bash
    curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    ```

    2. eksctl binary 파일을 /usr/local/bin으로 위치시킨다.

    ```bash
    sudo mv /tmp/eksctl /usr/local/bin
    ```

---

# EKS Cluster 구성

### EC2 vs. Fargate

- EC2

    →  자체 관리형 클러스터

    → 인스턴스 타입에 따라 가격이 결정

    → 각 인스턴스가 남아도는 리소스 없이 효율적 자원을 모두 사용 되어야 낭비를 줄일 수 있음

- Fargate

    → EC2 인스턴스 없이 컨테이너로 직접 실행 됨

    → 컨테이너를 구동하는 데 사용한 리소스만큼(CPU, Memory) 초단위로 요금이 청구된다. 즉, 실제 리소스를 사용한 만큼만 비용이 청구되며, EC2처럼 사용하지 않은 요금에 대해 돈을 지불할 일이 줄어듬

    → small workload에 적합

    → 컨테이너를 실행하기 위해 가상 머신 그룹을 프로비저닝, 구성 또는 조정할 필요 없음

1. 아래 내용을 cluster.yaml로 생성한다.

```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: yourname
  region: ap-northeast-2
  version: '1.17'

# NodeGroup holds all configuration attributes that are specific to a nodegroup
# You can have several node group in your cluster.
nodeGroups:
  - name: cpu-nodegroup
    instanceType: m5.xlarge
    desiredCapacity: 2
    minSize: 0
    maxSize: 4
    volumeSize: 50
    ssh:
      allow: true
      publicKeyPath: '~/.ssh/id_rsa.pub'

  # Example of GPU node group
  - name: Tesla-V100
    instanceType: p3.8xlarge
    # Make sure the availability zone here is one of cluster availability zones.
    availabilityZones: ["us-west-2b"]
    desiredCapacity: 0
    minSize: 0
    maxSize: 4
    volumeSize: 50
    ssh:
      allow: true
      publicKeyPath: '~/.ssh/id_rsa.pub'
```

2. 아래 명령어로 클러스터를 구성을 명령하면 수 분 내로 손쉽게 EKS(k8s) 클러스터가 구성된다.

```bash
eksctl create cluster -f cluster.yaml
```

3. 아래 명령어로 노드의 상태를 확인한다.

```bash
kubectl get nodes
```

4. (Optional) 노드 갯수 축소 및 확장 하려면 [아래 명령어](https://aws.amazon.com/ko/premiumsupport/knowledge-center/eks-worker-node-actions/)를 참고한다.

```bash
## 노드 갯수 조정
eksctl scale nodegroup --cluster=**클러스터 이름** --nodes=노드 개수 --name=**노드그룹 이름**
eksctl scale nodegroup --cluster=eksworkshop-eksctl --nodes=0 --name=nodegroup
****eksctl scale nodegroup --cluster=eksworkshop-eksctl-inkyu --nodes=0 --name=nodegroup

## 노드 Min, Max 값 조정
eksctl scale nodegroup --cluster=eksworkshop-eksctl-inkyu --name=nodegroup --nodes-min=0 # --nodes-max=5

```

- cluster.yaml 에서 클러스터 이름, 노드그룹 이름 확인

```bash
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: **yourname  // 클러스터 이름**
  region: ap-northeast-2
  version: '1.17'

nodeGroups:
  - name: **cpu-nodegroup // 노드그룹 이름**
    instanceType: m5.xlarge
    desiredCapacity: 2
    minSize: 0
    maxSize: 4
    volumeSize: 50
```

5. (Optional) EKS 클러스터 및 노드를 삭제 하려면 [아래 명령어](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/delete-cluster.html)를 참고한다.

- eksctl ≥ 0.31.0-rc.0 이상인지 확인

```bash
eksctl version
```

- 클러스터에서 실행중인 모든 서비스 나열

```bash
kubectl get svc --all-namespaces
```

- EXTERNAL-IP 값과 연결된 모든 서비스를 삭제

```bash
kubectl delete svc **서비스 이름**
```

- 클러스터 및 연결된 노드 삭제

```bash
eksctl delete cluster --name **클러스터 이름**
```

- 현재 정의된 Cluster Autoscaler Group을 확인하는 방법

```bash
aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='클러스터 이름']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]" --output table
```

### Kubernetes 의 용어 학습

![%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/1605961423251.jpg](%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/1605961423251.jpg)

![%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/1605961411872.jpg](%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/1605961411872.jpg)

![%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/20201121_175630.jpg](%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/20201121_175630.jpg)

# 2. Kubeflow 설치

1. [다운로드 링크](https://github.com/kubeflow/kfctl/releases/tag/v1.1.0)에서 kfctl v1.1.0 release를 다운로드 받는다.

```bash
wget https://github.com/kubeflow/kfctl/releases/download/v1.1.0/kfctl_v1.1.0-0-g9a3621e_linux.tar.gz
```

2. tar 압축 파일을 해제한다.

```bash
tar -xvf kfctl_v1.1.0-0-g9a3621e_linux.tar.gz

```

3. kubectl을 /usr/local/bin 폴더로 위치시킨다.

```bash
sudo mv ./kfctl /usr/local/bin 
```

```bash
sudo mv ./kfctl /usr/local/bin
```

- 4. 임시 환경변수 파일에 aws 계정 정보를 입력한다.

```bash
export AWS_REGION=ap-northeast-2

export AWS_CLUSTER_NAME=yourname

export KF_NAME=${AWS_CLUSTER_NAME}

export BASE_DIR=`pwd`

export KF_DIR=${BASE_DIR}/${KF_NAME}

export CONFIG_URI="https://raw.githubusercontent.com/kubeflow/manifests/v1.1-branch/kfdef/kfctl_aws.v1.1.0.yaml"

export CONFIG_FILE=${KF_DIR}/kfctl_aws.yaml
```

5. Kubeflow 다운로드 전 create_kubeflow.sh을 작성한다.

```bash
#!/bin/bash

source .env

mkdir -p ${KF_DIR}

cd ${KF_DIR}

wget -O kfctl_aws.yaml $CONFIG_URI

sed -i '/region: us-west-2/ a \      enablePodIamPolicy: true' ${CONFIG_FILE}

sed -i -e 's/kubeflow-aws/'"$AWS_CLUSTER_NAME"'/' ${CONFIG_FILE}

sed -i "s@us-west-2@$AWS_REGION@" ${CONFIG_FILE}

#sed -i "s@roles:@#roles:@" ${CONFIG_FILE}

aws iam list-roles \
    | jq -r ".Roles[] \
    | select(.RoleName \
    | startswith(\"eksctl-$AWS_CLUSTER_NAME\") and contains(\"NodeInstanceRole\")) \
    .RoleName"
```

6. Kubeflow 설치 전 Kubeflow 다운로드 전 deploy_kubeflow.sh을 작성한다.

```bash
#!/bin/bash

source .env

cd ${KF_DIR}
kfctl apply -f kfctl_aws.yaml
```

7. 앞서 작성한 create_kubeflow.sh와 deploy_kubeflow.sh를 bash로 실행한다.

```bash
bash create_kubeflow.sh
bash deploy_kubeflow.sh
```

# Kubernetes Terminology

- Namespaces
    - Kubernetes(EKS) Cluster 내의 논리적인 가상 클러스터
    - Pod, Service가 Namespace별로 생성이나 관리 될 수 있음
    - 사용자 권한도 Namespace별로 나눠서 부여할 수 있음
    - 리소스 할당량을 지정할 수 있음
    - 네트워크 정책을 이용하므로 다른 Namespace 간에 pod끼리 통신 가능하며, 반대로 차단도 가능

- 배포 방식
    - 롤링 업데이트란 ?

    **Rolling Update은** 애플리케이션 버전 업이 모두 한꺼번에 업데이트 되는 것이 아닌 순서대로 조금씩 업데이트하는 방법으로 똑같은 애플리케이션이 여러 개 병렬로 움직이는 경우 가능

    - Blue/Green(구버전/신버전) Deployment
        - 버전이 다른 두 애플리케이션을 동시에 가동하고 네트워크 설정을 사용해 별도의 공간에서 동작시킨다.
        - 업데이트 버전의 애플리케이션 테스트 완료 후 서비스는 전환시켜 업데이트를 완료하는 방법
        - 블루 (구버전)에서 장애 발생 시 블루로 바로 복구 가능한 장점
    - 카날리 배포

- Workload Resources
    - Container
    - Pod (Container 1개 이상)
    - ReplicaSet (RS)
        - Desired State를 유지하기 위해 두는 안전 장치
        - 레플리카 수 유지를 위해 생성하는 신규 파드에 대한 데이터를 명시하는 파드 템플릿을 포함
        - 명시된 동일한 Replica(Pod)의 복제 개수에 대한 보증

    - Replica Controller(RC)
        - **RC를 이용한** **v1 → v2  롤링 업데이트 시나리오**

        ![%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/Untitled%201.png](%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/Untitled%201.png)

        - RC (v1) replica 3 → 2, RC (v2) replica 0 → 1

        ![%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/Untitled%202.png](%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/Untitled%202.png)

        - RC (v1) replica 2 → 1, RC (v2) replica 1 → 2

        ![%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/Untitled%203.png](%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/Untitled%203.png)

        - RC (v1) replica 1 → 0, RC (v2) replica 2 → 3

        ![%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/Untitled%204.png](%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/Untitled%204.png)

    - Statefulset
        - 스토리지 볼륨을 사용해서 워크로드에 지속성을 제공
        - 겉으로 볼 땐 고요하게 Replica 수 만큼 유지하는 것처럼 보이지만 실제로 내부에서는 Pod들이 소멸과 생성을 반복
        - RS의 한계점인 Pod-PV 관리 편의성을 보완하기 위한  Controller
        - Statefulset은 미리 만들어둔 PVC관련 temp의 정보를 바탕으로 생성되는 pod에 PV(persistence volume)를 매칭시켜 Pod생성에 민첩성을 높임. 기존 RS에서는 처음 RS의 pod들을 하나의 PVC-PV를 바라보게 하는데, 이후에 만들어지는 New Pod는 새롭게 PV 매칭을 시켜줘야 함. 이는 편의성이 하락하는 문제를 야기시키므로 Statefulset controller 도입 >> 스토리지 볼륨을 사용해서 워크로드에 지속성을 제공
        - 애플리케이션이 안정적인 식별자 또는 순차적인 배포, 삭제 또는 스케일링이 필요한 경우에 사용하는 것이 적합함. Statefulset은 동일한 컨테이너 스펙을 기반으로 둔 파드들을 독자적으로 관리할 수 있음.

            ![%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/_.png](%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/_.png)

    - Deployment
        - Replica Controller가 **롤링 업데이트**하는 방식을 자동화해서 추상화한 개념이 Deployment
        - 과거 배포 이력 유지 및 이전 버전으로 Revision이 관리되어 쉽게 **롤백** 가능
        - 가장 보편적인 배포 단위
        - Desired 상태를 위해 Deployment내의 Replica Set, Pod들의 상태가 정해지게 됨.
        - create 커맨드에 –record 플래그 붙이면 디플로이먼트 생성이 롤아웃 히스토리에 기록

        ```bash
        kubectl create -f node-js-deploy.yaml --record
        ```

    - Deamonset
        - 클러스터 내 단일 Pod 복사본이 어느 노드 집합에서든 잘 작동하게 함
        - 클러스터 스토리지 데몬 실행
        - 로그 수집, 노드 모니터링 데몬 실행
        - 클러스터에 새로운 노드가 설치되면 데몬셋이 동작하여 자동으로 해당 노드에 파드를 실행
        - 클러스터에서 노드가 제거 될 경우 해당 노드에서 실행중이던 데몬셋 파드는 다른 노드로 이동하지 않고 그대로 사라짐
    - Job
        - 잡은 실행된 후에 종료해야 하는 성격의 작업을 실행할 때 사용되는 컨트롤러
        - 단일 잡, (워크 큐가 있는) 병렬 잡 등이 있음

# 3. AWS Aurora (PostgreSQL Engine)구축

- AWS에서 제공하는 MySQL 및 PostgreSQL과 호환되는 완전 관리형 관계형 데이터베이스 엔진
- 따릉이 실시간 데이터와 미세먼지 실시간 데이터를 적재하기 위한 용도로 일시적으로 사용하기 위해 사용. 이후에는 Kubernetes환경에서 PostgreSQL을 사용할 예정
- 스토리지를 인스턴스 당 최대 64TB 자동 확장

# 4. 데이터 수집 자동화 (feat.crontab)

### crontab을 이용하여 .py코드를 자동으로 실행시켜 업데이트된 데이터 수집

![%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/Untitled%205.png](%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/Untitled%205.png)

- 실시간 따릉이 대여정보 수집.py

```python
import pandas as pd
import json
import os
import glob
import requests
import psycopg2
import sqlalchemy
import datetime as dt

from sqlalchemy import create_engine
from datetime import datetime
from urllib import parse

print('==================')
url = 'http://openapi.seoul.go.kr:8088/485877647173646136306255674449/json/bikeList/{}/{}/'

total = pd.DataFrame() # 빈 데이터 프레임 선언
row = list(range(1, 4000, 1000)) # 호출 범위 지정 1
row_range = [[row[x], row[x + 1] - 1] for x in range(len(row) - 1)] # 호출 범>위 지정 2
time = datetime.today() + dt.timedelta(hours = 9)
url = 'http://openapi.seoul.go.kr:8088/485877647173646136306255674449/json/bikeList/{}/{}/' # 호출 범위 지정 3
print('##{}##'.format(time))

columns = ['거치대수', '대여소이름', '잔여대수', '거치율', '위도', '경도', '대여소id', '일시']

print('DATA CREATE . . .')

# 1000회씩 나눠서 호출
for r in row_range:
    api_url = url.format(r[0], r[1]) # 호출 범위 지정 4
    data = requests.get(api_url).json() # GET 방식으로 호출
    station = len(data['rentBikeStatus']['row']) # 대여소 수

    # 대여소 수 만큼 호출
    for idx in range(station):
        value = list(data['rentBikeStatus']['row'][idx].values()) # 대여소별 >실시간 값
        value.append(0) # 날짜 추가
        total = total.append(pd.DataFrame(columns = columns, data = [value])) # 데이터프레임 생성
        total = total.reset_index(drop = True) # 인덱스 정렬

total['일시'] = total['일시'].apply(lambda x: time)
print('DATA CREATE COMPLETE!')

print('DATA INSERT START!')
engine = create_engine("postgresql://postgres:password@aws aurora endpoint:5432/DB")
total.to_sql(name = 'bike',
          con = engine,
          schema = 'public',
          if_exists = 'append',
          index = False)

del total
print('DATA INSERT COMPLETE!')
```

- 실시간 따릉이 대여 정보 5분 간격으로 수집 결과

![%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/Untitled%206.png](%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/Untitled%206.png)

- 미세먼지 데이터 수집.py

```python
import json
import requests
import pandas as pd
import datetime as dt
import psycopg2
import sqlalchemy

from datetime import datetime
from pandas.io.json import json_normalize
from sqlalchemy import create_engine

print('==================')
print('##미세먼지 데이터 수집##')
print('DATA INSERT START!')
key = '75476a53717a6f7338346e6a664341' # 주화님 key
print('##{}##'.format(datetime.today() + dt.timedelta(hours = 9)))
api_url = "http://openAPI.seoul.go.kr:8088/{}/json/RealtimeCityAir/1/1000/".format(key)

print('DATA CREATE . . .')
r = requests.get(api_url)
time = datetime.today() + dt.timedelta(hours = 9)
air = r.json()['RealtimeCityAir']['row']
df = json_normalize(air)

df.rename(columns={'MSRDT': '일시', 'MSRRGN_NM': '권역명', 'MSRSTE_NM': '측정>소명', 'PM10': '미세먼지', 'PM25': '초미세먼지', 'O3': '오존', 'NO2': '이산화>질소', 'CO': '일산화탄소','SO2': '아황산가스', 'IDEX_NM': '통합대기환경등급', 'IDEX_MVL': '통합대기환경지수', 'ARPLT_MAIN': '지수결정물질' }, inplace=True)

df['일시'] = df['일시'].apply(lambda x: time)

print('DATA CREATE COMPLETE!')
engine = create_engine("postgresql://postgres:6team123!@database-aurora-instance-1.cj92narf3bwn.ap-northeast-2.rds.amazonaws.com:5432/final_project")
df.to_sql(name = 'dust',
          con = engine,
          schema = 'public',
          if_exists = 'append',
          index = False)

del df
print('DATA INSERT COMPLETE!')
```

- 실시간 미세먼지 정보 5분 간격으로 수집

![%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/Untitled%207.png](%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%91%E1%85%B3%E1%84%85%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%20ea4ebf31d01e42d1b3af5a9891cbe4e2/Untitled%207.png)

- 자전거 대여 정보 + 날씨 정보  JOIN문

SELECT b.*, w.* \

FROM weather w INNER JOIN weather_join wj ON w.권역명=wj.권역명 \

INNER JOIN bike b ON b.대여소이름=wj.대여소이름 \

WHERE b.일시>='2020-11-24 16:30';
