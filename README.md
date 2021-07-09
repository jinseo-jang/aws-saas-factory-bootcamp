# Building SaaS Solutions on AWS

![SaaSAWS](images/SaaS-Factory.png "SaaS Factory")

# 소개

SaaS 아키텍팅은 개발자에게 멀티 테넌시, 온보딩, 보안, 데이터 분할, 테넌트 격리 및 Identity와 같은 것에 대한 고민을 하도록 만듭니다.

이 부트캠프 워크숍에서는 샘플 SaaS 솔루션의 아키텍처를 살펴봄으로써 SaaS 아키텍처의 핵심 개념에 대해 설명합니다.

이 워크숍은 참석여자가 SaaS 아키텍처의 핵심 개념을 직접 구성해보는 실습을 제공합니다. 이를 통해 모범 사례 SaaS 아키텍쳐에 대한 아이디어를 얻을 수 있습니다.

# 누가 참여 해야 하나?

이 부트캠프의 내용은 SaaS를 처음 접하는 사람들을 대상으로 합니다. 그러나 SaaS에 대한 배경 지식이 있는 사람들이더라도, 이 부트캠프 내용을 통해 SaaS를 AWS에서 구현하는 경험을 해볼 수 있습니다.

# 워크샵을 어떻게 시작 하나요?

만약 AWS에서 호스팅 하는 이벤트를 통해 참여 하는 분들이라면 호스트의 안내를 따르시면 됩니다.

만약 개인적으로 실습을 실행하려면 AWS 계정에서 [workshop.yml](https://github.com/jinseo-jang/aws-saas-factory-bootcamp/blob/master/resources/workshop.yml) Cloudformation 템플릿을 시작하기만 하면 됩니다. 그리고 이어서 아래 Lab 1 아이콘을 클릭해 랩 가이드를 따르면 됩니다.

이때 주의 사항은 **이 워크샵은 프리 티어 범위 밖의 AWS 서비스를 사용하므로 비용을 최소화하기 위해 완료되면 CloudFormation 스택을 삭제해야 하셔야 합니다.**

# Lab 가이드

### Lab 1 - Identity and Onboarding

[![Lab1](images/lab1.png)](Lab1.md)

### Lab 2 - Multi-Tenant Security & User Roles

[![Lab2](images/lab2.png)](Lab2.md)

### Lab 3 - Data Isolation

[![Lab3](images/lab3.png)](Lab3.md)

# License

This workshop is licensed under the Apache 2.0 License. See the LICENSE file.
