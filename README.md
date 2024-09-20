# kubernetes에서 kafka 활용하기

## Table of Contents
- [Overview](#overview)
- [kafka 설치](#kafka-설치)
- [관리 도구 설치](#관리-도구-설치)
  - [postgresql](#postgresql) (optional)
  - [odd-platform](#odd-platform)
  - [kafka ui](#kafka-ui)

## Overview

본 프로젝트는 Cloud Native Korea Community Day 2024에서 시연을 하기 위해서 작성되었다.   
프로젝트에서는 다음과 같은 동작의 예제를 보여준다.   
> strimzi operator를 활용한 kafka, kafka connect 설치   
> kafka 관리 주변 도구 설치(helm chart)   
> kafka 모니터링 통합(prometheus, kafka exporter, grafana)   

<img src="./static/strimzi.jpg">

## kafka 설치

strimzi operator는 다음 링크를 참고한다.   
[strimzi - operator component overview](https://strimzi.io/docs/operators/latest/overview#overview-components_str)   
[strimzi - deploy kakfa](https://strimzi.io/docs/operators/latest/deploying#con-deploy-paths-str)   
[operatorhub](https://operatorhub.io/operator/strimzi-kafka-operator)   

```yaml
kubectl apply -f ./operators/strimzi
kubectl apply -f ./crds/kafka/kafka.yaml
```
### 설명
kafka, kafka connect, connector 등을 설치하기 위해서는 먼저 strimzi operator를 설치한다.   
이후 crd(custom resource definition)을 생성하면 operator가 동작하여 kafka를 설치한다.

## 관리 도구 설치 
> kafka-ui, odd-platform, apicurio
```yaml
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add opendatadiscovery https://opendatadiscovery.github.io/charts
helm repo add kafbat-ui https://kafbat.github.io/helm-charts

kubectl apply -f ./operators/apicurio-registry.yaml

helm install my-postgres ./helm/postgresql/ -f ./helm/postgresql/values.yaml -n kafka-manager
helm upgrade -i odd-platform ./helm/odd-platform/ -f ./helm/odd-platform/values.yaml -n kafka-manager
helm upgrade -i kafka-ui ./helm/kafka-ui/ -f ./helm/kafka-ui/values.yaml -n kafka-manager

kubectl apply -f ./crds/apicurio/apicurio.yaml -n kafka-manager
```
### postgresql
postgresql이 없는 경우 bitnami postgresql을 helm chart로 설치한다.


### odd-platform 
odd-platform은 database가 필요하며 postgresql을 지원한다.   
설치된(또는 bitnami로 설치한) postgresql의 정보를 helm chart의 value값에 셋팅한다.   
(또는 환경변수로 등록)   

```yaml
config:
 yaml:
   spring:
     datasource:
       url: jdbc:postgresql://my-postgres-postgresql-hl:5432/odd-platform
       username: myodd
       password: myodd
```

### apicurio-registry
apicurio registry는 세 가지 타입의 저장방식을 지원하며 각각의 이미지가 존재한다.   
- in-memory: quay.io/apicurio/apicurio-registry-mem
- postgresql: quay.io/apicurio/apicurio-registry-sql
- kafkasql: quay.io/apicurio/apicurio-registry-kafkasql

apicurio는 operator를 통해 간단히 설치가 가능하나, 커스텀 옵션이 부족한 편이다(예: ingress disable)   
helm chart가 없어서 기본 생성하였다. 이를 통해 배포가 가능하며, 환경변수를 아래와 같이 등록할 수 있다.
```yaml
env:
  # for kafkasql
  KAFKA_BOOTSTRAP_SERVERS: "my-cluster-kafka-bootstrap.kafka:9092"
  # for postgresql
  # REGISTRY_DATASOURCE_URL: "jdbc:postgresql://postgres/apicurio-registry"
  # REGISTRY_DATASOURCE_USERNAME: "apicurio-registry"
  # REGISTRY_DATASOURCE_PASSWORD: "password"
```

### kafka ui
kafka ui에 broker 목록을 추가한다.   
그리고 opendatadiscovery와 연계하기 위하여 odd에서 토큰을 발행하여 등록한다.   
management -> collectors -> add collector    
<img src="./static/odd_collecor_config.png">
```yaml
yamlApplicationConfig:
  # {}
  kafka:
    clusters:
      - name: my-broker
        bootstrapServers: my-cluster-kafka-bootstrap.kafka:9092
  integration:
    odd:
      url: http://apicurio-apicurio-registry.kafka-manager:8080
      token: 8UPwAqr2lzcycMh3FOr8UxXoKqNpsaWntjkSSxWR
```