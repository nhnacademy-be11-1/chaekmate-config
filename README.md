# Chaekmate Config Server

## 프로젝트 소개

이 프로젝트는 [Chaekmate](https://github.com/nhnacademy-be11-1) MSA 프로젝트를 구성하는 여러 마이크로서비스들의 `application.yml` 설정 파일을 한 곳에서 관리하기 위한 **중앙 설정 서버**입니다.

Spring Cloud Config Server를 기반으로, 각 서비스의 설정을 외부 Git 저장소에서 가져와 제공합니다. 이를 통해 서비스 코드의 변경이나 재배포 없이 설정 파일만 수정하여 애플리케이션의 동작을 변경할 수 있습니다.

## 주요 기능

- **`yml` 파일 중앙 관리:** 각 마이크로서비스에서 사용하는 `application.yml` 파일을 단일 Git 저장소에서 통합 관리합니다.

- **동적 설정 업데이트:** Git 저장소의 `yml` 파일이 변경되었을 때, 각 서비스가 재시작 없이 변경된 설정값을 애플리케이션에 반영할 수 있는 기능을 제공합니다. 이 과정은 아래와 같이 동작합니다.
    1.  Config Server가 Git 저장소에서 변경된 `yml` 내용을 가져옵니다.
    2.  설정 변경이 필요한 각 마이크로서비스의 `POST /actuator/refresh` 엔드포인트로 요청을 보냅니다. (이 과정은 Webhook을 통해 자동화할 수 있습니다.)
    3.  해당 요청을 받은 서비스는 Config Server로부터 새로운 설정값을 받아옵니다.
    4.  서비스 내에서 `@RefreshScope` 어노테이션이 붙은 컴포넌트(Bean)들만 새로 만들어져 변경된 설정값이 주입됩니다.
    
- **서비스 디스커버리:** Eureka 서버에 자신을 등록하여 다른 서비스들이 설정 서버를 쉽게 찾을 수 있도록 합니다.

## `yml` 파일 저장소

모든 `yml` 설정 파일은 아래의 Git 저장소에서 관리됩니다.

- **Git Repository:** [https://github.com/nhnacademy-be11-1/chaekmate-config-repo.git](https://github.com/nhnacademy-be11-1/chaekmate-config-repo.git)

## 실행 방법

### 환경 변수

애플리케이션을 실행하기 전에 다음 환경 변수를 설정해야 합니다.

- `GIT_USERNAME`: `yml` 파일이 저장된 Git 저장소에 접근하기 위한 사용자 이름
- `GIT_PASSWORD`: Git 저장소 접근을 위한 비밀번호 또는 Personal Access Token
- `EUREKA_URL`: (운영 환경) 서비스 등록을 위한 Eureka 서버의 URL

### 빌드 및 실행

Maven을 사용하여 애플리케이션을 빌드하고 실행할 수 있습니다.

```bash
./mvnw spring-boot:run
```

애플리케이션은 기본적으로 `8888` 포트에서 실행됩니다.

## API (확인 및 디버깅용)

**참고:** 이 API는 개발자가 직접 호출하기보다는, 각 마이크로서비스가 시작될 때 필요한 `yml` 설정 파일을 자동으로 가져가기 위해 사용됩니다. 아래 API는 설정이 올바르게 제공되는지 확인하거나 디버깅할 목적으로만 사용해주세요.

`yml` 파일의 내용은 다음 엔드포인트를 통해 JSON 형식으로 확인할 수 있습니다.

`/{application}/{profile}`

- `{application}`: `yml` 파일을 사용할 마이크로서비스의 이름 (e.g., `book-service`)
- `{profile}`: 조회할 환경 (e.g., `dev`, `prod`)

### 예시

`book-service`의 `dev` 환경 `yml` 설정을 확인하고 싶을 경우:

```bash
curl http://localhost:8888/book-service/dev
```

---

## 마이크로서비스 설정 예시

Config Server를 사용하는 각 마이크로서비스(e.g., `book-service`)는 자신의 설정 대부분을 Git 저장소에 위임하므로, 프로젝트 내부에는 최소한의 설정만 남게 됩니다.

특히 **Eureka**와 같은 서비스 디스커버리를 사용하면, Config Server의 네트워크 주소(`uri`)를 직접 명시할 필요 없이 서비스 이름(`service-id`)으로 동적으로 찾아갈 수 있어 더욱 유연한 구성이 가능합니다.

> **💡 왜 `application.yml`이 아닌 `bootstrap.yml` 인가요?**
> `bootstrap.yml`은 `application.yml`보다 먼저 로드되는 파일입니다. 애플리케이션이 본격적으로 시작되기 전에 Config Server로부터 설정 정보를 먼저 가져와야 하므로, 가장 먼저 로드되는 `bootstrap.yml`에 Config Server 접속 정보를 설정하는 것입니다.

```yaml
# 예시: book-service의 /src/main/resources/bootstrap.yml

spring:
  application:
    # Config Server에서 설정 파일을 찾기 위한 자신의 서비스 이름
    name: book-service
  profiles:
    # 사용할 프로파일 (dev, prod 등)
    # 이 값에 따라 Config Server는 book-service-dev.yml 또는 book-service-prod.yml을 찾음
    active: dev
  cloud:
    config:
      # Config Server의 주소를 직접 명시하는 대신 Eureka를 통해 찾도록 설정
      discovery:
        enabled: true
        # Eureka에 등록된 Config Server의 서비스 ID
        service-id: config-server

eureka:
  client:
    service-url:
      # Eureka 서버의 주소
      defaultZone: ${EUREKA_URL:http://localhost:8761/eureka/}
```

이제 `book-service`가 시작되면, 위 `bootstrap.yml` 설정을 읽어 Eureka 서버(`defaultZone`)에 접속합니다. 그리고 Eureka에게 `config-server`라는 서비스의 주소를 물어본 뒤, 응답받은 주소로 접속하여 자신의 설정(`book-service-dev.yml`)을 가져오게 됩니다. 이렇게 하면 Config Server의 IP나 포트가 변경되어도 각 서비스의 설정을 수정할 필요가 없어집니다.

---

## 자주 묻는 질문 (FAQ)

**Q1: `yml` 파일을 수정하고 `git push`만 했는데 어떻게 모든 서비스에 설정이 자동으로 반영되나요?**

A: **Webhook** 기능을 사용하기 때문입니다. 전체 과정은 다음과 같습니다.
1.  **`git push`**: 개발자가 `yml` 파일 수정 후 Git 저장소에 push합니다.
2.  **Webhook 알림**: Git 저장소(e.g., GitHub)는 push 이벤트를 감지하고 지정된 URL로 "변경사항 발생" 알림을 보냅니다.
3.  **자동 새로고침**: 이 알림을 받은 CI/CD 서버 등이 미리 준비된 스크립트를 실행하여, 관련된 모든 마이크로서비스의 `/actuator/refresh` API를 호출해줍니다.

이러한 자동화 덕분에 개발자는 `git push`만 하면 됩니다.

**Q2: 각 서비스는 `/actuator/refresh` API를 어떻게 가지고 있나요?**

A: **Spring Boot Actuator** 라이브러리 덕분입니다.
1.  각 마이크로서비스 프로젝트에 `spring-boot-starter-actuator` 의존성을 추가하면, `/actuator/health`처럼 애플리케이션을 관리하고 모니터링하는 여러 API가 자동으로 생성됩니다.
2.  `/actuator/refresh`는 그 중 하나로, Spring Cloud와 함께 사용될 때 활성화되는 동적 리프레시 전용 API입니다.
3.  보안상 기본으로 비활성화되어 있어, 각 서비스의 `application.yml`에 `management.endpoints.web.exposure.include: refresh`와 같이 직접 노출 설정을 해주어야 사용할 수 있습니다.