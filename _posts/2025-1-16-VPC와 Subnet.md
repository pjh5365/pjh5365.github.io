---
title: VPC와 Subnet
author: pjh5365
date: 2025-1-16 21:35:00 +0900
categories: [DevOps, Cloud]
tags: [devops]
image:
  path: /assets/img/devops.png
  alt: DevOps
---

## VPC?

**VPC(Virtual Private Cloud)**는 여러 클라우드 플랫폼에서 제공하는 **논리적으로 격리된 네트워크 공간**이다. 이를 통해 사용자만의 사설망을 만들 수 있고, 외부 네트워크와의 연결도 사용자가 직접 제어할 수 있다.

## Subnet?

Subnet은 VPC 내부의 네트워크를 더 세분화한 것이다. 하나의 VPC안에서 여러 개의 Subnet을 만들 수 있으며, 각 Subnet은 특정 가용 영역(AZ, Availability Zone) 내에 존재한다.

Subnet은 다시 네트워크의 특성에 따라 **Public Subnet**과 **Private Subnet**으로 나눌 수 있다.

### Public Subnet

외부 인터넷과 직접 통신할 수 있는 Subnet으로 **공인 IP 주소**를 가질 수 있으며, **IGW(Internet Gateway)**를 통해 외부와의 양방향 통신이 가능하다.

주로 웹 서버나 API 서버 등 외부와 통신이 필요한 리소스를 해당 Subnet에 둔다.

### Private Subnet

외부 인터넷에서 직접 접근할 수 없는 Subnet으로 공인 IP 주소를 가질 수 없으며, 인터넷을 사용하려면 **NAT Gateway**를 설정해야 한다.

주로 데이터베이스 서버, 백엔드 서버 등 외부 접근이 불필요한 리소스를 해당 Subnet에 둔다.

### NAT Gateway와 Internet Gateway

#### Internet Gateway(IGW)

VPC의 **Public Subnet에 배치된 리소스가 외부 인터넷과 양방향 통신할 수 있도록 해주는 장치**로, VPC에 연결된 모든 Public Subnet은 IGW를 통해 외부와 연결된다. 공용 IP를 가진 리소스에 대해 기본적으로 작동하며, 별도의 추가 설정 없이 외부와의 통신이 가능하다.

#### NAT Gateway

**Private Subnet의 리소스가 인터넷을 사용하기 위해 내부에서 외부로의 트래픽만 접근할 수 있도록 해주는 장치**로, Public Subnet에 배치되며, Private Subnet의 Routing Table에 NAT Gateway를 기본 게이트웨이로 설정해야 한다.

NAT Gateway는 Private Subnet에 속한 **여러 개의 리소스가 하나의 공용 IP(NAT)를 통해 인터넷에 접근**하도록 지원하며 외부에서 Private Subnet으로의 접근은 차단한다.

> **NAT Gateway 없이 Private Subnet에서 인터넷 사용하기**
>
> NAT Gateway 없이도 **Public Subnet의 리소스를 프록시로 활용하여 NAT의 역할을 수행하도록 설정**하면 Private Subnet의 리소스가 인터넷을 사용할 수 있다. 비용을 절감할 수 있지만, 프록시 서버의 관리 및 고가용성 설정이 추가적으로 필요하다.
{: .prompt-tip}