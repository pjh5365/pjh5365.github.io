---
title: Docker의 진화와 내부 구조 (containerd, runc, containerd-shim의 역할)
author: pjh5365
date: 2024-9-25 22:11:00 +0900
categories: [DevOps, Docker]
tags: [DevOps, Docker]
image:
  path: /assets/img/dockerLogo.svg
  alt: Docker

---

## Docker의 역사

Docker는 처음 나왔을 때, **하나의 데몬 프로세스(Docker Daemon == dockerd)가 컨테이너의 생성, 관리, 실행 등 모든 작업을 처리**했다. 그러나 Docker가 인기를 얻고, 컨테이너 기술이 확장되면서 dockerd에서 모든 기능을 처리하는 것이 점점 더 복잡해졌다.

이를 해결하기 위해, Docker는 컨테이너 관리 기능을 더 작은 모듈로 분리하였고, 이때 나온 것이 containerd라는 컴포넌트이다. **containerd는 컨테이너의 실행, 관리 등 컨테이너의 라이프사이클을 관리**하기 시작했다.

이후, **OCI(Open Container Initiative) 표준을 따르는 runc라는 컨테이너 실행기**가 등장했다. 이를 통해 **containerd는 컨테이너 라이프사이클 관리에 집중하고, runc는 컨테이너의 실제 실행을 담당**하도록 역할이 나뉘었다.

추후 containerd-shim이 등장하면서, runc가 컨테이너를 실행한 후 해당 컨테이너 프로세스를 관리하고 유지하는 역할을 맡게 되었다. **containerd-shim은 containerd가 내리는 명령을 수행하며, containerd가 중단되더라도 실행 중인 컨테이너가 중단되지 않도록 보장하는 역할**을 한다.

## Docker Engine

Docker Engine은 컨테이너 기술을 구현하고 관리하는 핵심 소프트웨어로, 도커 전체 시스템을 뜻한다.

### Docker Client

사용자가 CLI 명령어를 통해 Docker와 상호작용하는 인터페이스이다. 사용자가 `docker run` 같은 명령을 입력하면, **Docker Clinet는 그 명령을 Docker Daemon에 전달**한다.

### Docker Daemon(dockerd)

Docker Engine의 핵심 구성 요소 중 하나로, 도커 클라이언트의 명령을 처리하고 컨테이너와 이미지를 관리하는 역할을 한다.

### containerd

containerd는 **OCI(Open Container Initiative) 런타임 사양과 이미지 사양을 준수하는 고수준 컨테이너 관리 시스템으로, 컨테이너 라이프사이클을 관리하는 역할**을 한다. 컨테이너를 생성하고, 실행 환경을 설정하는 과정에서 중요한 작업들을 수행한다.

#### containerd의 주요 역할

- 이미지풀링: **containerd**는 **Docker Hub**나 로컬 레지스트리에서 이미지를 가져온다.
- 파일 시스템 준비: 컨테이너 루트 파일 시스템을 준비하고, 이미지와 관련된 계층을 설정한다.
- 네트워크 설정: 컨테이너가 사용할 네트워크 인터페이스 및 namespace를 설정한다.
- 리소스 할당: 컨테이너가 사용할 CPU, 메모리 등의 리소스를 cgorups를 통해 할당한다.

### runc

Docker에서 컨테이너 기술을 위해 **OCI 런타임 사양을 구현한 컨테이너 실행기로, 실제 컨테이너 프로세스를 실행하는 저수준의 런타임**이다. runc는 containerd가 설정한 컨테이너를 전달받아 실제로 실행시키는 역할을 한다.

#### runc의 주요 역할

- 컨테이너 실행 담당
  - runc는 containerd로부터 설정된 컨테이너 환경을 전달받아 컨테이너를 실제로 실행한다. **컨테이너가 실행된 후 containerd-shim이 그 컨테이너 프로세스를 이어받아 관리**한다.
- Linux 커널 기능 사용
  - runc는 Linux 커널의 namespace와 cgroups 기능을 사용하여, 컨테이너를 격리된 상태로 실행한다. 이를 통해 컨테이너는 독립된 환경에서 안전하게 실행된다.
- 컨테이너 실행 후 종료
  - runc는 컨테이너 실행만을 담당하며, 컨테이너가 실행된 후 즉시 종료된다. 실행 이후 프로세스 관리는 **containerd-shim**이 담당한다.

### containerd-shim

컨테이너가 실행된 후 해당 컨테이너 프로세스를 독립적으로 관리하는 도구이다. **runc가 컨테이너를 실행한 후, containerd-shim이 해당 프로세스의 상태를 지속적으로 추적하고 관리**한다. 이를 통해 **containerd가 중단되더라도 실행 중인 컨테이너는 계속 유지**된다.

**containerd-shim의 주요 역할**

- 컨테이너 실행 후의 상태 관리
  - containerd는 컨테이너를 실행할 때 **runc를 호출해서 실제 컨테이너 프로세스를 실행**한다.
  - **runc는 이 프로세스를 격리된 환경에서 실행한 후 바로 종료**된다.
  - containerd-shim이 runc의 컨테이너 프로세스를 이어받아 프로세스를 계속 관리한다. 즉, 컨테이너의 상태를 지속적으로 추적하고, 필요할 때 **containerd의 명령에 따라** 컨테이너를 중지하거나 종료한다.
- containerd와의 독립성 보장
  - containerd-shim이 컨테이너를 독립적으로 관리하기 때문에 containerd가 재시작되거나 중단되더라도, containerd-shim은 컨테이너가 중단되지 않고 계속 실행되도록 보장한다.
  - containerd가 재시작 된 후, containerd와 containerd-shim이 다시 연결되어 컨테이너 상태를 함께 관리할 수 있다.
- 컨테이너 종료 및 리소스 해제
  - 컨테이너가 종료되면, containerd-shim은 컨테이너가 사용했던 리소스를 해제하고, 관련된 작업을 마무리하는 역할도 수행한다. 이를 통해 시스템 리소스가 적절하게 관리되고, 낭비되지 않도록 한다.

### containerd, runc, containerd-shim 의 역할 구분
- containerd(관리자 역할)
  - 여러 컨테이너를 생성, 실행, 중지를 지시하고, 상태를 모니터링하는 중앙 관리자 역할을 한다(전체적인 흐름을 관리한다).
  - 컨테이너를 계획하고 실행 명령을 내리며, 컨테이너 라이프사이클을 관리한다.
- runc(일꾼 역할)
  - containerd가 준비한 컨테이너 환경에서 실제 컨테이너를 실행한다.
  - 컨테이너를 실행한 후, runc 종료되며, 더 이상 해당 프로세스를 관리하지 않는다(일꾼은 일을 끝낸 후 바로 떠난다).
- containerd-shim(중간 관리자 역할/containerd의 명령을 수행한다)
  - runc가 컨테이너를 실행한 후에도, containerd-shim이 그 컨테이너 프로세스를 지속적으로 관리한다(개별 컨테이너를 관리한다).
  - 컨테이너의 상태를 계속 추적하고, 필요할 때 컨테이너를 중지하거나 종료하는 역할을 한다(containerd 가 내린 명령을 수행한다). 
  - containerd-shim은 containerd와 독립적으로 작동하므로, containerd가 중단되더라도 컨테이너 프로세스가 계속 실행될 수 있도록 보장한다.

## OCI(Open Container Initiative)

컨테이너 이미지 형식과 컨테이너 런타임을 표준화하기 위한 프로젝트로, 이 표준을 통해 서로 다른 도구들이 호환성을 가질 수 있게 되었다.

- **OCI 이미지 사양:** 컨테이너 이미지를 어떻게 정의하고 다루어야 하는지에 대한 표준이다. 이를 통해 여러 도구가 동일한 이미지 형식을 사용할 수 있다.
- **OCI 런타임 사양:** 컨테이너를 어떻게 실행하고 관리해야 하는지에 대한 표준이다. runc 같은 런타임 도구가 이 표준을 따른다.

이 표준 덕분에 **Docker, Kubernetes, Podman** 같은 서로 다른 도구들이 동일한 방식으로 컨테이너를 관리하고 실행할 수 있다. 이를 통해 컨테이너 기술의 상호 호환성이 크게 향상되었다.

