# Github Action

Jenkins를 사용하면 로컬 git을 포함하여 다양한 git 외부저장소들을 사용해서 CI/CD pipeline을 구축할 수 있다.

Jenkins를 사용하지 않아도 Github, GitLab, Gitea 등 다양한 외부저장소들에서 자체적으로 `action`혹은 `runner`을 제공하여 CI/CD pipeline을 구축할 수 있게 도와준다.

그 중 가장 많이 사용한다고 여겨지는 `Github`에서 제공하는 `Github Action`을 이용하여 Jenkins에서 image를 build하고 외부 repository에 push하는 방법을 알아본다.

## Github Action 구성

Github Action은 repository의 `.github/workflows` 내부에 지정되어있는 `yaml파일`을 사용하여 수행한다.

> 각 외부저장소마다 문법은 비슷하나 제공하는 action의 버전 및 사용방법은 다를 수 있으니 공식문서를 참고해서 작성한다.

```yaml
name: Deploy to ECR

# main branch에 push가 발생하면 동작한다.
on:
  push:
    branches: [ main ]

jobs:
  build:
    # action이 ubuntu위에서 동작하도록 설정한다.
    name: Build and Push Image to ECR
    runs-on: ubuntu-latest

    steps:
    # 현재 repository로 부터 checkout한다.
    - name: Checkout code
      uses: actions/checkout@v4

    # secrets에 저장한 env 값을 .env파일로 생성하기 위해 작성
    - name: Create .env file
      run: |
        echo "MYSQL_HOST=${{ secrets.MYSQL_HOST }}" >> .env
        echo "MYSQL_DB=${{ secrets.MYSQL_DB }}" >> .env
        echo "MYSQL_USER=${{ secrets.MYSQL_USER }}" >> .env
        echo "MYSQL_PW=${{ secrets.MYSQL_PW }}" >> .env

    # AWS 서비스에 접근하기 위한 credential을 설정
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ap-northeast-1

    # ECR에 로그인하기 위한 패스워드를 가져와서 도커 로그인
    # ECR에서 푸시명령을 확인한다.
    - name: Get ECR login password
      run: |
        echo "$(aws ecr get-login-password --region ap-northeast-1)" | docker login --username AWS --password-stdin ${{ secrets.ECR_REGISTRY }}

    - name: Build, tag, and push image to Amazon ECR
      run: |
        IMAGE_TAG=${{ github.run_number }}
        LATEST_TAG=latest
        IMAGE_REPO=${{ secrets.IMAGE_REPO }}
        ECR_REGISTRY=${{ secrets.ECR_REGISTRY }}
        docker build -t $IMAGE_REPO:$IMAGE_TAG .
        docker tag $IMAGE_REPO:$IMAGE_TAG $ECR_REGISTRY/$IMAGE_REPO:$IMAGE_TAG
        docker push $ECR_REGISTRY/$IMAGE_REPO:$IMAGE_TAG

    # 최신 이미지 참조 하기 위해 동일한 이미지에 "latest" 태그를 추가   
        docker tag $IMAGE_REPO:$IMAGE_TAG $ECR_REGISTRY/$IMAGE_REPO:$LATEST_TAG
        docker push $ECR_REGISTRY/$IMAGE_REPO:$LATEST_TAG 
```