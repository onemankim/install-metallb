
# MetalLB 설치 가이드
    * https://metallb.universe.tf/

## 구성 요소 및 버전
* metallb/controller ([metallb/controller:v0.8.2](https://hub.docker.com/layers/metallb/controller/v0.8.2/images/sha256-d1fe971bdb986915cafe339444329d8ef64cb59b11aaf7b22aeb167fdbd67aad?context=explore))
* metallb/speaker ([metallb/speaker:v0.8.2](https://hub.docker.com/layers/metallb/speaker/v0.8.2/images/sha256-a9c822e640fa5aed6f244a47bf7a66e5d1dac765479af44b954f4ae86072943d?context=explore))

## 폐쇄망 설치 가이드
설치를 진행하기 전 아래의 과정을 통해 필요한 이미지 및 yaml 파일을 준비한다.
1. **폐쇄망에서 설치하는 경우** 사용하는 image repository에 metallb 설치 시 필요한 이미지를 push한다. 

    * 작업 디렉토리 생성 및 환경 설정
    ```bash
    $ mkdir -p ~/metallb-install
    $ export METALLB_HOME=~/metallb-install
    $ export METALLB_VERSION=v0.8.2
    $ export REGISTRY=172.22.8.106:5000
    $ cd $METALLB_HOME
    ```

    * 외부 네트워크 통신이 가능한 환경에서 필요한 이미지를 다운받는다.
    ```bash
    $ sudo docker pull metallb/controller:${METALLB_VERSION}
    $ sudo docker save metallb/controller:${METALLB_VERSION} > metallb-controller_${METALLB_VERSION}.tar
    $ sudo docker pull metallb/speaker:${METALLB_VERSION}
    $ sudo docker save metallb/speaker:${METALLB_VERSION} > metallb-speaker_${METALLB_VERSION}.tar
    ```

    * metallb yaml을 다운로드한다. 
    ```bash
    $ curl https://raw.githubusercontent.com/google/metallb/v0.8.2/manifests/metallb.yaml > metallb.yaml
    ```

    * metallb yaml을 공식 홈페이지에서 다운로드가 불가능한 경우 아래의 링크를 이용한다.
    ```bash
    $ curl https://raw.githubusercontent.com/tmax-cloud/hypercloud-install-guide/master/MetalLB/metallb_v0.8.2.yaml > metallb.yaml
    ```

    * metallb_cidr yaml을 다운로드한다.
    ```bash
    $ curl https://raw.githubusercontent.com/tmax-cloud/hypercloud-install-guide/master/MetalLB/metallb_cidr.yaml > metallb_cidr.yaml
    ```

2. 위의 과정에서 생성한 tar 파일들을 폐쇄망 환경으로 이동시킨 뒤 사용하려는 registry에 이미지를 push한다.
    ```bash
    $ sudo docker load < metallb-controller_${METALLB_VERSION}.tar
    $ sudo docker load < metallb-speaker_${METALLB_VERSION}.tar

    $ sudo docker tag metallb/controller:${METALLB_VERSION} ${REGISTRY}/metallb/controller:${METALLB_VERSION}
    $ sudo docker tag metallb/speaker:${METALLB_VERSION} ${REGISTRY}/metallb/speaker:${METALLB_VERSION}

    $ sudo docker push ${REGISTRY}/metallb/controller:${METALLB_VERSION}
    $ sudo docker push ${REGISTRY}/metallb/speaker:${METALLB_VERSION}
    ```

## 설치 가이드
0. [metallb.yaml 수정](#step0 "step0")
1. [metallb 설치](#step1 "step1")
2. [metallb 대역 설정](#step2 "step2")

<h2 id="step0"> Step0. metallb yaml 수정 </h2>

* 목적 : `metallb yaml에 이미지 registry, 버전 정보 수정`
* 생성 순서 : 
    * 아래의 command를 수정하여 사용하고자 하는 image 버전 정보를 수정한다. (기본 설정 버전은 v0.8.2)
	```bash
            sed -i 's/v0.8.2/'${METALLB_VERSION}'/g' metallb.yaml
	```

* 비고 :
    * `폐쇄망에서 설치를 진행하여 별도의 image registry를 사용하는 경우 registry 정보를 추가로 설정해준다.`
	```bash
            sed -i 's/metallb\/speaker/'${REGISTRY}'\/metallb\/speaker/g' metallb.yaml 
            sed -i 's/metallb\/controller/'${REGISTRY}'\/metallb\/controller/g' metallb.yaml 
	```

<h2 id="step1"> Step 1. metallb </h2>

* 목적 : `metallb 설치`
* 생성 순서: metallb.yaml 설치  `ex) kubectl apply -f metallb.yaml`
* 비고 : 
    * metallb-system 네임스페이스 사용
    * controller-xxxxxxxxxx-xxxxx (1개의 pod)
    * speaker-xxxxx (모든 노드에 pod)


<h2 id="step2"> Step 2. metallb 대역 설정 </h2>

* 목적 : `metallb에서 사용할 대역 설정 (호스트와 동일한 대역 사용)`
* 생성 순서: metallb_cidr.yaml 적용  `ex) kubectl apply -f metallb_cidr.yaml`
* 비고 :
    ```bash
    apiVersion: v1
    kind: ConfigMap
    metadata:
      namespace: metallb-system
      name: config
    data:
      config: |
        address-pools:
        - name: default
          protocol: layer2
          addresses:
          - 172.22.8.160-172.22.8.180
          - 172.22.8.184-172.22.8.190
    ```
    
## 삭제 가이드