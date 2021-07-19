# Lab 3 – Isolating Tenant Data

### Overview

앞서 SaaS 아키텍처의 많은 핵심 요소를 다루었습니다. 하지만 아직 다루지 않은 부분이 바로 Tenant isolation 입니다. SaaS 공급자는 각 테넌트의 리소스가 모든 종류의 교차 테넌트 액세스로부터 보호되도록 해야합니다. 그런데 이 부분이, 모든 테넌트가 인프라 리소스를 모두 공유 하는 상황에서는 달성하기 어렵습니다.

이를 해결하려면 기본적인 인증 이상의 것이 필요 합니다. 즉, 테넌트 환경을 격리하고 보호 한다는 것을 보장 할 수 있는 정책과 액세스 제어를 도입해야합니다. 이건 심지어, 리소스가 공유되지 않는 SaaS 환경에서도 교차 테넌트 액세스에 대한 노출을 최소화하기 위해 추가 조치의 하나로 꼭 필요 합니다.

이번 실습 에서는 DynamoDB 테이블에 있는 데이터를 분리하는 방법을 이해 하는데 초점을 맞출 것입니다. 특히 제품과 주문 테이블을 예시로, 어떻게 테넌트 데이터를 격리하고 분리 할 수 ​​있는지 살펴 보겠습니다. 이를 위해 먼저 데이터를 분할하는 방법을 고려해야합니다. 아래는 제품 및 주문 테이블의 파티션을 나타낸 그림 입니다.

<p align="center"><img src="./images/lab3/table_partitioning.png" alt="Lab 3 Isolating Tenant Data Overview Table Partitioning"/></p>

그림을 보면 여러 테넌트의 데이터(item)가 한 테이블에 함께 위치 합니다. 따라서 이러한 테이블 중 하나에 액세스하면 모든 테넌트의 데이터에 액세스 할 수 있는 상황 입니다.

저희의 목표는 **이러한 테이블에 대한 액세스 정책의 범위를 테이블의 item 레벨 까지 지정할 수있는 보안 모델을 구현하는 것입니다. 즉, 기본적으로 테넌트 자기 자신에게만 해당되는 item에 액세스 할 수 있도록 제한 하는 겁니다.**

이를 위해 이번 실습에서, Amazon Cognito, Amazon Identity and Access Management (IAM), AWS Security Token Service (STS)의 조합을 활용해 테이블 데이터(item)접근을 제한하는 장치를 만들겁니다. 즉 이 서비스 조합을 활용해, 앞에서 언급한 SaaS Identity에 사용자의 접근 정책을 바인딩 하여 이를 기준으로 데이터(item)접근을 제한할겁니다.

이 Isolation 모델을 구현하는 데는 두 가지 주요 단계가 있습니다. **1️⃣ 첫번째 단계는**, 테넌트가 처음 프로비저닝 될 때 각 테넌트에 대한 IAM Role 세트를 생성 해야 합니다. 테넌트의 환경에 존재하는 모든 역할들에 대해 해당 테넌트의 시스템 리소스에 대한 액세스 범위를 지정하는 정책(IAM Policy)을 만들어야합니다. 아래 그림 에서는 이 온보딩 프로세스의 개념적 표현과 각 테넌트에 대해 이러한 Role을 만드는 방법을 볼 수 있습니다.

<p align="center"><img src="./images/lab3/onboarding.png" alt="Lab 3 Isolating Tenant Data Overview Onboarding"/></p>

그림 왼쪽에는 Lab 1에서 구축 한 Registration 프로세스가 있습니다. 오른쪽에는 각 역할에 대해 생성되는 정책 모음이 있습니다. 개별 User 마다 별도의 정책을 만들어 적용 하는게 아니라 역할을 기준으로 그룹핑 하여 테넌트에 해당 되는 사용자에게 적용 됩니다.

**Isolation 모델 구현의 2️⃣ 두 번째 단계는**, 테넌트가 리소스에 액세스 할 때 작동합니다. 아래 다이어그램은 이 프로세스의 기본 적인 흐름을 보여 줍니다.

<p align="center"><img src="./images/lab3/authentication.png" alt="Lab 3 Isolating Tenant Data Overview Authentication"/></p>

위 다이어그램을 보면, 제품 목록을 가져 오기 위한 요청과 함께 UI에서 Product Manager 서비스가 호출되는 것을 볼 수 있습니다. JWT 토큰 (인증 중에 획득)은 HTTP 요청의 Authorization 헤더로 전달됩니다. 이 토큰에는 User identity, 역할 및 Tenant identity에 대한 데이터가 포함됩니다. <span style="color:blue">**이 토큰은 사용자 및 테넌트 속성을 담아 전달하는데 유용 하지만 사실 리소스에 대한 테넌트의 액세스를 제어하는 ​​데는 아무일도 하지 않습니다. 따라서,이 토큰의 데이터를 사용하여 DynamoDB 테이블에 액세스하는 데 필요한 범위가 지정된 자격 증명(Credential)을 획득 해야합니다**</span>.

액세스 범위가 지정된 자격 증명(Credential)은 어떻게 획득할까요?

1️⃣ `GET` 요청이 Product Manager 서비스로 들어 오면 Cognito로 `getCredentialsForIdentity ()`호출을 통해 토큰(idToken)을 전달합니다. 2️⃣ **그런 다음 Cognito는 해당 토큰을 열어서 테넌트 식별자와 사용자 역할을 검사하고 프로비저닝 중에 생성 된 정책 중 하나와 매핑 합니다.** 3️⃣ 그런 다음 STS를 통해 **임시** 자격 증명(Credentials)세트 (다이그램 하단에 표시됨)를 생성하고 이를 Product Manager 서비스로 반환합니다. 4️⃣ Product Manager 서비스는 이러한 임시 자격 증명(Credentials)을 사용하여 이 자격 증명이 _tenan id_ 별 액세스 범위를 지정한다는 확신을 가지고 DynamoDB 테이블에 액세스합니다.

### 실습에서 만드는 것들

- <span style="color:blue">**테넌트간 교차 액세스 가능한 상황**</span> – 먼저 지정된 정책과 scope 없이 테넌트 간 데이터에 교차 접근 하는 상황을 만들어 봅니다.
- <span style="color:blue">**미리 프로비전된 IAM 정책 구성**</span> – 이제 교차 테넌트 액세스 하는 상황을 보았으므로 교차 테넌트 액세스(의도된 또는 의도하지 않은)를 방지하는 데 사용할 수 있는 정책을 만듭니다. 다양한 역할/리소스 조합에 대한 정책을 생성 하고, 이러한 정책이 DynamoDB 테이블에 대한 액세스 범위를 지정하는 데 어떻게 사용되는지 이해할 수 있게 될겁니다. 그런 다음 새 테넌트를 프로비저닝하고 이러한 정책이 새 테넌트의 IAM에 어떻게 적용되는지 확인합니다.
- <span style="color:blue">**사용자의 역할에 IAM 정책 매핑**</span> – Cognito를 사용하여 사용자 역할을, 우리가 생성한 IAM 정책에 매핑 하는 규칙을 생성할 수 있습니다. 이 부분에서는 이러한 IAM 정책이 테넌트 및 사용자의 역할에 대해 어떻게 설정 되는지 확인할 수 있습니다.
- <span style="color:blue">**Tenant-scoped 자격 증명(credentials) 획득**</span> – 마지막으로 위에 설명된 IAM 정책에 따라 테넌트가 접근 가능한 범위(scope)가 지정된 자격 증명(credential)을 획득하는 방법을 확인 합니다. 이 자격 증명(credentials)은 데이터에 대한 액세스를 제어합니다. 이것이 어떻게 명시적으로 테넌트 간 범위 지정을 적용하는지 알 수 있습니다.

## Part 1 - 테넌트간 교차 액세스 가능한 상황

테넌트간 교차 액세를 제한 하는 **정책**을 도입하기 전에, 테넌트간 교차 액세스가 가능한 상황을 만들어 보겠습니다.

**Step 1** - [Lab 2](Lab2.md)에서 두 테넌트의 제품 카탈로그에 제품을 추가했습니다. 만약 현재 두개의 테넌트 그리고 각 테넌트 별로 최소 1개의 제품이 카탈로그에 등록되어 있지 않은 상태라면 Lab 2 단계를 따라서 제품 등록을 완료 하세요.

교차 테넌트 액세스가 가능한 상황을 인위적으로 만드려면 먼저, 고유한 테넌트 식별자가 필요 합니다. 서로 다른 두 테넌트의 테넌트 ID를 찾기 위해 **AWS 콘솔**에서 **DynamoDB** 서비스로 이동하고 페이지 왼쪽 상단에 있는 **Tables** 옵션을 선택합니다. **TenantBootcamp** 테이블을 선택한 다음 **Items** 탭을 선택합니다.

<p align="center"><img src="./images/lab3/part1/dynamo_tenant.png" alt="Lab 3 Part 1 Step 14 Dynamo TenantBootcamp Table"/></p>

**Step 2** - 테넌트 생성에 사용한 사용자 이름/이메일을 매칭하여 **두 테넌트에 대한 tenant_id 값을 확인하고 따로 메모 해 둡니다**. 이후 단계에서 이 값들이 필요합니다.

**Step 3** - 이제 Product Manager 서비스 코드로 돌아가서 코드를 수정해 보겠습니다.Cloud9에서 `Lab3/Part1/product-manager/`로 이동합니다. `server.js`를 두 번 클릭하거나 마우스 오른쪽 버튼으로 클릭하고 **Open**를 클릭하여 편집기에서 파일을 엽니다.

<p align="center"><img src="./images/lab3/part1/open_server.js.png" alt="Lab 3 Part 1 Step 16 Open server.js"/></p>

**Step 4** - 아래 코드와 같이 모든 제품을 조회 하는 `GET` 함수를 확인 합니다.

```javascript
app.get("/products", function (req, res) {
  var searchParams = {
    TableName: productSchema.TableName,
    KeyConditionExpression: "tenant_id = :tenant_id",
    ExpressionAttributeValues: {
      ":tenant_id": tenantId,
      //":tenant_id": "<INSERT TENANT TW0 GUID HERE>"
    },
  };
  // construct the helper object
  tokenManager.getSystemCredentials(function (credentials) {
    var dynamoHelper = new DynamoDBHelper(
      productSchema,
      credentials,
      configuration
    );
    dynamoHelper.query(searchParams, credentials, function (error, products) {
      if (error) {
        winston.error("Error retrieving products: " + error.message);
        res.status(400).send('{"Error": "Error retrieving products"}');
      } else {
        winston.debug("Products successfully retrieved");
        res.status(200).send(products);
      }
    });
  });
});
```

이 함수는 제품 목록을 조회 하는데 사용됩니다. 전달된 사용자 요청의 보안 토큰에서 추출한 `tenant_id`를 참조하는 것을 확인할 수 있습니다. 인위적으로 테넌트간 교차 액세스가 가능한 상황을 만들기 위해서 이 `tenant_id`를 전 단계에서 얻은 **TenantTwo** 의 `tenant_id`값으로 **교체** 합니다. 교체한 코드는 아래와 같아야 합니다.

```javascript
app.get("/products", function (req, res) {
  winston.debug("Fetching Products for Tenant Id: " + tenantId);
  var searchParams = {
    TableName: productSchema.TableName,
    KeyConditionExpression: "tenant_id = :tenant_id",
    ExpressionAttributeValues: {
      ":tenant_id": "TENANT4c33c2eae9974615951e3dc04c7b9057",
    },
  };
  // construct the helper object
  tokenManager.getSystemCredentials(function (credentials) {
    var dynamoHelper = new DynamoDBHelper(
      productSchema,
      credentials,
      configuration
    );
    dynamoHelper.query(searchParams, credentials, function (error, products) {
      if (error) {
        winston.error("Error retrieving products: " + error.message);
        res.status(400).send('{"Error" : "Error retrieving products"}');
      } else {
        winston.debug("Products successfully retrieved");
        res.status(200).send(products);
      }
    });
  });
});
```

**Step 5** - 이제 업데이트된 Product Manager 마이크로서비스를 배포해야 합니다. 먼저 툴바에서 **File**을 클릭한 다음 **Save**를 클릭하여 편집한 `server.js` 파일을 Cloud9에 저장합니다.

<p align="center"><img src="./images/lab3/part1/cloud9_save.png" alt="Lab 3 Part 1 Step 18 Save server.js"/></p>

**Step 6** - 수정된 서비스 배포를 위해 `Lab3/Part1/product-manager/` 디렉토리로 이동하여 `deploy.sh`를 마우스 오른쪽 버튼으로 클릭하고 **Run**을 클릭하여 쉘 스크립트를 실행합니다.

<p align="center"><img src="./images/lab3/part1/cloud9_run.png" alt="Lab 3 Part 1 Step 19 Cloud9 Run"/></p>

**Step 7** - `deploy.sh` 이 성공적으로 완료 될 때까지 기다립니다.

<p align="center"><img src="./images/lab3/part1/cloud9_run_script_complete.png" alt="Lab 3 Part 1 Step 20 Cloud9 Script Finished"/></p>

**Step 8** - 이제 새로운 버전의 코드가 애플리케이션에 어떤 영향을 주는지 확인해보겠습니다. 위에서 생성한 **TenantOne**에 대한 자격 증명을 사용하여 시스템에 다시 로그인합니다(**TenantTwo**가 여전히 로그인되어 있으면 페이지 오른쪽 상단의 드롭다운을 사용하여 로그아웃합니다).

**Step 9** - 페이지 상단의 **Catalog** 메뉴 옵션을 선택합니다. 방금 인증해 접속한 **TenantOne** 사용자의 제품 리스트만 표시되야 합니다. **그러나 실제로 _목록에는 실제로 TenantTwo_ 의 제품이 포함**되어 있습니다. **Step 4 에서 하드코딩으로 TenantTwo 의 `tenant_id`를 강제로 반영**했기 때문에 TenantOne이 조회 하더라도 TenantTwo의 데이터가 조회 되는, 즉 **테넌트간 교차 액세스 상황이 된것입니다.**

**Recap**: 여기 까지의 과정의 주요 사항은, 단순 인증 만으로는 SaaS 시스템을 보호하기에 충분하지 않다는 것입니다. 즉, 추가 정책 및 권한 부여 장치가 없다면 언제든 의도하지 않게 다른 테넌트의 데이터에 액세스하는 상황이 발생 할 수 있습니다.

## Part 2 - Part2-미리 프로비전된 IAM 정책 구성

교차 테넌트 액세스로부터 시스템을 더 잘 보호하기 위한 정책이 필요하다는 것은 이제 이해되었을 겁니다. 문제는 이것을 위해 어떤 솔루션이 있느냐 입니다. 솔루션이 될 수 있는 것은 **IAM 정책**입니다. IAM 정책을 사용하여 테넌트 리소스에 대한 사용자의 액세스 수준을 제어하는 ​​규칙을 생성할 수 있습니다.

처음부터 새 정책을 만드는 대신 실습을 위해 미리 프로비저닝된 정책을 편집해 사용하겠습니다.

**Step 1** - 편집할 정책을 찾기 위해 AWS 콘솔에서 IAM 서비스로 이동하고 페이지 왼쪽 상단의 옵션 목록에서 **Policies**을 선택합니다.

**Step 2** - 이제 저희가 만든 두 테넌트(**TenantOne** 및 **TenantTwo**)와 연결된 정책을 찾아야 합니다. TenantOne부터 시작하겠습니다. 화면 상단의 검색 상자에 TenantOne에 대한 테넌트의 GUID를 입력합니다. (\*앞선 Step 2에서 이 값을 메모 했습니다)

<p align="center"><img src="./images/lab3/part2/iam_search_policies.png" alt="Lab 3 Part 2 Step 2 IAM Search Policies"/></p>

**Step 3** - TenantOne의 **admin**에 대한 정책과 테넌트 **user**에 대한 두 번째 정책이 조회 됩니다. **TenantAdmin** 정책 이름 앞의 열에서 **삼각형/화살표**를 선택하여 정책을 자세히 살펴봅니다. 그런 다음 페이지 중앙 근처에 있는 **Edit policy** 버튼을 선택합니다.

<p align="center"><img src="./images/lab3/part2/iam_edit_policy.png" alt="Lab 3 Part 2 Step 3 IAM Edit Policy"/></p>

**Step 4** - 콘솔에 DynamoDB 정책 및 Cognito User pool 정책 목록이 표시됩니다. 저희는 **ProductBootcamp** 테이블에 대한 정책을 편집하는 데 관심을 두기 때문에 목록의 왼쪽 가장자리에 있는 **화살표를 선택**하여 이 목록에서 축소된 각 DynamoDB 항목을 엽니다. 확장된 각 정책 항목의 맨 아래에서 **Resources** 섹션을 찾을 수 있습니다. **ProductBootcamp** 테이블을 참조하는 정책 집합을 찾습니다. ARN은 다음과 유사합니다.

<p align="center"><img src="./images/lab3/part2/iam_dynamo_arn.png" alt="Lab 3 Part 2 Step 4 IAM Dynamo ARN"/></p>

**Step 5** - 저희는 이 정책에 대한 **Request conditions**을 수정 할겁니다. 왜냐하면 여기서 설정한 조건이 사용자가 DynamoDB 테이블 내에서 액세스할 수 있는 항목을 제어/제한하는 ​​기능의 핵심이 되기 때문 입니다. <span style="color:blue">**즉, 저희가 원하는 건 정책에서 **TenantOne**의 테넌트 식별자와 일치하는 파티션 키 값을 가진 사용자만 테이블의 해당 항목에 액세스할 수 있도록 하는 겁니다.**</span> 이를 위해 **Request conditions** 값 위로 마우스를 가져간 다음 **조건에 대한 텍스트를 선택**하면 조건에 대한 편집 모드로 전환됩니다.

<p align="center"><img src="./images/lab3/part2/iam_request_conditions.png" alt="Lab 3 Part 2 Step 5 IAM Policy Request Conditions"/></p>

**Step 6** - 목록 하단에서 **Add condition** 옵션을 선택합니다. **Condition key**에 대해 **dynamodb:LeadingKeys**를 선택합니다. **Qualifier**에 대해 **For all values ​​in request**를 선택합니다. **Operator**에 대해 **StringEquals**를 선택합니다. 마지막으로 **Value** 텍스트 상자에 **TenantOne**의 GUID를 입력합니다. **Add** 버튼을 클릭합니다. **Review policy** 버튼을 선택한 다음 **Save Changes** 버튼을 선택하여 이 변경 사항을 정책에 저장합니다.

<p align="center"><img src="./images/lab3/part2/iam_add_request_condition.png" alt="Lab 3 Part 2 Step 6 IAM Policy Add Request Condition"/></p>

**Step 7** - 이제 **TenantTwo**에 대해 동일한 프로세스를 반복하려고 합니다. TenantOne에 대한 모든 참조를 **TenantTwo**로 교체하여 2-6단계를 다시 완료합니다. 이렇게 하면 TenantTwo도 테넌트 교차 접근으로 부터 보호됩니다.

**Recap**: 이번 파트에서는 사용자가 테이블의 항목에 액세스를 시도할 때 DynamoDB 테이블의 파티션 키 값이 테넌트 식별자와 일치해야 한다는 **조건을 정책에 추가하는** 과정을 진행 했습니다. 이를 통해 테넌트간 교차 액세스를 제한 할 수 있게 되었습니다.

## Part 3 - 사용자의 역할에 IAM 정책 매핑

이제 IAM 정책이 정의 되었으므로 이러한 정책을 특정 사용자 역할과 연결할 수 있는 방법이 필요합니다. 궁극적으로 사용자의 역할과 테넌트의 접근 범위를 _특정_ IAM 정책 집합에 일치시키는 방법이 필요합니다. 이 시나리오에서는 **Cognito이 제공하는 Role Matching 기능**을 활용할 것입니다. Cognito를 사용하면 이 매칭 조건 집합을 정의할 수 있으며 최종 적으로 매칭 된 IAM 정책을 기반으로 하는 **Credentials**을 내보낼 수 있습니다. 이것이 바로 우리가 테넌트 격리 모델을 구현 하는데 필요한 중요한 요소 입니다.

이러한 Role Matching 조건은 이미 생성되었습니다. **Cognito 콘솔**에서 살펴보겠습니다.

**Step 1** - AWS 콘솔에서 Cognito 서비스로 이동합니다. 랜딩 페이지에서 **Manage Identiy Pools** 버튼을 선택하여 identity pool 목록을 확인합니다. 여기에는 온보딩한 각 테넌트에 대한 **별도의 identity pool**이 포함되어 있을 겁니다.

**TenantOne** 및 **TenantTwo**에 대한 자격 identity pool을 찾습니다. 테넌트의 GUID가 이름에 들어가 있을 겁니다. **TenantOne**과 연결된 자격 identity pool을 클릭합니다.

<p align="center"><img src="./images/lab3/part3/cognito_identity_pools.png" alt="Lab 3 Part 3 Step 1 Cognito Identity Pools"/></p>

**Step 2** - 이제 페이지 오른쪽 상단의 **Edit identity pool** 링크를 선택합니다.

<p align="center"><img src="./images/lab3/part3/cognito_identity_pool_details.png" alt="Lab 3 Part 3 Step 2 Cognito Identity Pool Details"/></p>

**Step 3** - identity pool 편집 페이지를 아래로 스크롤하면 **Authentication Providers**라는 제목이 표시됩니다. 이 섹션을 펼치면 인증 공급자 구성이 있는 페이지가 표시됩니다.

이제 두개의 테넌트 역할에 대한 역할 매핑을 볼 수 있습니다. 관리자를 나타내는 **TenantAdmin** 역할이 있고 SaaS 시스템의 개별 비관리자 사용자에게 매핑되는 **TenantUser** 역할이 있습니다. 당연히 이들은 시스템과 리소스에 대한 액세스 수준이 다릅니다.

클레임 열에는 실습 1에서 Cognito에서 구성한 사용자 지정 **role** 속성과 일치하는 값(URL 인코딩됨)이 있습니다. 즉, 이를 통해 해당 **custome claim 이름이 테넌트의 역할 이름과 일치**하면 IAM 정책(DynamoDB 제한 사항)은 **STS에서 반환된 임시 보안 토큰**에 실려 전달 되게 됩니다.

<p align="center"><img src="./images/lab3/part3/cognito_role_matching.png" alt="Lab 3 Part 3 Step 3 Cognito Role Matching"/></p>

**Recap**: 이제 테넌트 격리의 두 번째 단계 구축을 완료 했습니다. 이 연습을 통해 Cognito identity pool 에서 역할 매핑 규칙을 활용하는 모습도 확인했습니다. 이러한 매핑 규칙을 바탕으로 테넌트의 역할(TenantAdmin 및 TenantUser)에 따라 사전에 정의된 IAM policy가(\*이 실습의 첫 번째 부분에서 구성한) 연결 되고, 이 policy가 적용된 STS가 생성 됩니다.

## Part 4 - Acquiring Tenant-Scoped Credentials

앞의 과정까지, 테넌트간 isolation을 만드는데 필요한 필수 요소를 모두 갖추었습니다. 즉 DynamoDB 테이블에 대한 액세스 범위를 지정하는 IAM 정책을 만들었고, Cognito를 통해 인증된 사용자의 역할에 따라 해당 IAM 정책과 연결하는 Role mapping 조건도 Cognito에 구성했습니다. 이제 남은 것은 이러한 요소를 실행하고 테넌트 리소스에 대한 액세스 범위 적용한 IAM 정책이 적용된 Credential(STS credential)을 획득하는 코드를 서비스에 반영 하는 겁니다.

**Step 1** - 먼저 Product Manager 서비스가 테넌트 격리를 지원하도록 수정 하겠습니다. Cloud9에서 `Lab3/Part4/product-manager/`로 이동하고 편집기에서 `server.js`를 두 번 클릭하거나 마우스 오른쪽 버튼으로 클릭하고 **Open**를 선택하여 엽니다.

<p align="center"><img src="./images/lab3/part4/cloud9_open_server.js.png" alt="Lab 3 Part 4 Step 1 Cloud9 Open server.js"/></p>

아래 코드는 테넌트 격리를 위한 마지막 핵심 부분입니다. 인증된 사용자의 보안 토큰에서 자격 증명(credential)을 획득하는 `tokenManager`에 대한 호출이 추가되어 있습니다. `getCredentialsFromToken()` 메서드는 HTTP 요청을 받아 **테넌트별로 범위가 지정된** `Credential`을 반환합니다. 이러한 자격 증명(credential)은 `dynamoHelper`에 대한 호출에 사용 되어 **테넌트간 데이터에 교차 접근 할 수 없다는 걸** 보장 합니다.

```javascript
app.get("/product/:id", function (req, res) {
  winston.debug("Fetching product: " + req.params.id);
  tokenManager.getCredentialsFromToken(req, function (credentials) {
    // init params structure with request params
    var params = {
      tenant_id: tenantId,
      product_id: req.params.id,
    };
    // construct the helper object
    var dynamoHelper = new DynamoDBHelper(
      productSchema,
      credentials,
      configuration
    );
    dynamoHelper.getItem(params, credentials, function (err, product) {
      if (err) {
        winston.error("Error getting product: " + err.message);
        res.status(400).send('{"Error" : "Error getting product"}');
      } else {
        winston.debug("Product " + req.params.id + " retrieved");
        res.status(200).send(product);
      }
    });
  });
});
```

**Step 2** - 위에서 설명한 `getCredentialsFromToken()` 함수는 identity token를 적절한 IAM 정책에 매핑하고 credential 형식으로 반환하는 작업이 일어나는 곳입니다. 이 과정을 다음 `getCredentialsFromToken()` 함수를 구현하는 `TokenManager`의 코드를 통해 보다 자세히 살펴보겠습니다.

```javascript
module.exports.getCredentialsFromToken = function (req, updateCredentials) {
  var bearerToken = req.get("Authorization");
  if (bearerToken) {
    var tokenValue = bearerToken.substring(bearerToken.indexOf(" ") + 1);
    if (!(tokenValue in tokenCache)) {
      var decodedIdToken = jwtDecode(tokenValue);
      var userName = decodedIdToken["cognito:username"];
      async.waterfall(
        [
          function (callback) {
            getUserPoolWithParams(userName, callback);
          },
          function (userPool, callback) {
            authenticateUserInPool(userPool, tokenValue, callback);
          },
        ],
        function (error, results) {
          if (error) {
            winston.error("Error fetching credentials for user");
            updateCredentials(null);
          } else {
            tokenCache[tokenValue] = results;
            updateCredentials(results);
          }
        }
      );
    } else if (tokenValue in tokenCache) {
      winston.debug("Getting credentials from cache");
      updateCredentials(tokenCache[tokenValue]);
    }
  }
};
```

이 함수의 동작 흐름을 살펴보겠습니다.

- 첫 번째 작업은 HTTP 요청에서 보안 `bearerToken`을 추출하는 것입니다. 이것은 사용자 인증 후 Cognito에서 받은 토큰입니다.
- 그런 다음 토큰을 디코딩하고 `userName` 속성을 추출합니다.
- 다음으로 일련의 호출이 순서대로 실행됩니다. 현재 사용자의 `userPool`을 찾는 것으로 시작합니다. 그런 다음 `authenticateUserInPool()`을 호출합니다. `TokenManager` 라는 클래스의 일부인 이 함수는 궁극적으로 Cognito의 `getCredentialsForIdentity()` 메서드를 호출하여 이 때 사용자로부터 받은 토큰을 전달합니다.

이 과정이 이전에 구성한 **Role mapping 작업을 트리거** 하기 위해 Cognito을 호출 하는 겁니다. Cognito는 제공된 토큰에서 테넌트의 역할을 추출하여 IAM 정책과 일치시킨 다음 호출하는 함수에 반환되는 **범위가 지정된 자격 증명의 임시 집합**을 구성합니다.

**Step 2** - 이제 이 새 버전의 Product Manager를 배포하여 작동하는 모습을 살펴보겠습니다. Cloud9에서 `Lab3/Part4/product-manager` 디렉토리로 이동하여 `deploy.sh`를 마우스 오른쪽 버튼으로 클릭하고 **Run**을 클릭하여 셸 스크립트를 실행합니다.

<p align="center"><img src="./images/lab3/part4/cloud9_run.png" alt="Lab 3 Part 4 Step 2 Cloud9 Run"/></p>

**Step 3** - `deploy.sh` 이 성공적으로 완료될 때 까지 기다립니다.

<p align="center"><img src="./images/lab3/part4/cloud9_run_script_complete.png" alt="Lab 3 Part 4 Step 3 Cloud9 Script Finished"/></p>

**Step 4** - 변경 사항을 테스트 해보겠습니다. **TenantTwo**가 계속 로그인되어 있으면 애플리케이션 탐색 모음의 왼쪽 상단에 있는 드롭다운을 사용하여 로그아웃 합니다. 이제 **TenantOne**으로 로그인하고 **Catalog** 메뉴 항목을 선택하고 **TenantOne의** 제품을 확인하여 데이터에 액세스합니다.

그런데 단순히 이렇게 조회 하는 것 만으로 변경 사항이 정상적으로 적용 되어 테넌트간 교차 접근을 방지 하고 있는지 확인하기 어렵습니다. 따라서 다음 Part 5 에서 다소 강제적인 작업을 통해 테스트 해보겠습니다.

**Recap**: 이번 파트에서 저희는 HTTP 헤더로 부터 얻은 JWT **bearer token**, 사전에 세팅한 **custom claim** 그리고 **Cognito가 제공하는 Role mapping**을 활용해 테넌트간 교차 접근을 막을 수 있는 **STS 임시 credential**을 만드는 프로세스를 Product manager 서비스에 반영 했습니다.

## Part 5 - Tenant-scoped 자격 증명(credentials)을 활용한 isolation 동작 검증

이번 파트에서는 Part 4 까지 진행 만든 tenant-scoped credential에 의해 교차 테넌트 액세스가 제한 되는 것을 테스트 해볼겁니다. 이를 위해 강제로 데이터에 접근하려는 테넌트와는 다른 **테넌트의 식별 정보를** 코드에 반영하여 이 tenant-scope credential이 작동 하는걸 지켜 볼겁니다.

**Step 1** - 이전과 마찬가지로 최신 Product manager 서비스의 소스 코드를 수정하여 특정 테넌트 식별자를 하드코딩 삽입합니다. Cloud9에서 `Lab3/Part5/product-manager/` 폴더로 이동하고 편집기에서 `server.js`를 두 번 클릭하거나 마우스 오른쪽 버튼으로 클릭하고 **Open**를 선택하여 엽니다.

<p align="center"><img src="./images/lab3/part5/cloud9_open_server.js.png" alt="Lab 3 Part 5 Step 1 Cloud9 Open server.js"/></p>

**Step 2** - 해당되는 테넌트의 모든 제품을 조회 하는 `GET` 함수가 아래와 같이 있을 겁니다:

```javascript
app.get("/products", function (req, res) {
  winston.debug("Fetching Products for Tenant Id: " + tenantId);
  tokenManager.getCredentialsFromToken(req, function (credentials) {
    var searchParams = {
      TableName: productSchema.TableName,
      KeyConditionExpression: "tenant_id = :tenant_id",
      ExpressionAttributeValues: {
        ":tenant_id": tenantId,
        //":tenant_id": "<INSERT TENANTTWO GUID HERE>"
      },
    };
    // construct the helper object
    var dynamoHelper = new DynamoDBHelper(
      productSchema,
      credentials,
      configuration
    );
    dynamoHelper.query(searchParams, credentials, function (error, products) {
      if (error) {
        winston.error("Error retrieving products: " + error.message);
        res.status(400).send('{"Error": "Error retrieving products"}');
      } else {
        winston.debug("Products successfully retrieved");
        res.status(200).send(products);
      }
    });
  });
});
```

**TenantTwo**에 대한 `tenant_id`를 다시 **하드코딩으로 삽입**하여 새 코드가 교차 테넌트 액세스를 방지하는지 확인합니다. DynamoDB에서 찾아 이전에 메모 해둔 **TenantTwo**의 `tenant_id` 이 값으로 `tenantId`를 **_대체_**합니다. 변경 완료된 코드는 아래와 같아야 합니다.

```javascript
app.get("/products", function (req, res) {
  winston.debug("Fetching Products for Tenant Id: " + tenantId);
  tokenManager.getCredentialsFromToken(req, function (credentials) {
    var searchParams = {
      TableName: productSchema.TableName,
      KeyConditionExpression: "tenant_id = :tenant_id",
      ExpressionAttributeValues: {
        ":tenant_id": "TENANT4c33c2eae9974615951e3dc04c7b9057",
      },
    };
    // construct the helper object
    var dynamoHelper = new DynamoDBHelper(
      productSchema,
      credentials,
      configuration
    );
    dynamoHelper.query(searchParams, credentials, function (error, products) {
      if (error) {
        winston.error("Error retrieving products: " + error.message);
        res.status(400).send('{"Error" : "Error retrieving products"}');
      } else {
        winston.debug("Products successfully retrieved");
        res.status(200).send(products);
      }
    });
  });
});
```

**Step 3** - 이제 위 변경 사항이 반영된 Product manager 마이크로서비스를 배포해야 합니다. 먼저 툴바에서 **File**을 클릭한 다음 **Save**를 클릭하여 편집한 `server.js` 파일을 Cloud9에 저장합니다.

<p align="center"><img src="./images/lab3/part5/cloud9_save.png" alt="Lab 3 Part 5 Step 3 Save server.js"/></p>

**Step 4** - 수정된 서비스를 배포하려면 `Lab3/Part5/product-manager/` 디렉토리로 이동하여 `deploy.sh`를 마우스 오른쪽 버튼으로 클릭하고 **Run**을 클릭하여 셸 스크립트를 실행합니다.

<p align="center"><img src="./images/lab3/part5/cloud9_run.png" alt="Lab 3 Part 5 Step 4 Cloud9 Run"/></p>

**Step 5** - `deploy.sh` 이 성공적으로 실행 완료 될때 까지 기다립니다.

<p align="center"><img src="./images/lab3/part5/cloud9_run_script_complete.png" alt="Lab 3 Part 5 Step 5 Cloud9 Script Finished"/></p>

**Step 6** - 새 버전의 서비스가 배포되었으니 이제 애플리케이션에 어떤 영향을 미치는지 확인할 수 있습니다. 위에서 생성한 **TenantOne**에 대한 자격 증명을 사용하여 시스템에 다시 로그인합니다.(**TenantTwo**가 여전히 로그인되어 있으면 페이지 오른쪽 상단의 드롭다운을 사용하여 로그아웃합니다).

**Step 7** - 페이지 상단의 **Catalog** 메뉴 옵션을 선택합니다. 이렇게 하면 방금 인증한 **TenantOne** 사용자의 카탈로그가 표시됩니다. 그런데 실제로 **표시된 제품이 없습니다**. 실제로 JavaScript 콘솔 로그를 보면(브라우저의 개발자 도구 사용) 오류가 발생했음을 알 수 있습니다. 이는 **TenantOne**으로 로그인 했지만 서비스에는 정작 **TenantTwo**가 하드 코딩되어 있기 때문입니다. 이는 <span style="color:blue">**TenantTwo의 식별 정보를 하드 코딩으로 넣었음에도 불구하고, TenantOne의 Credential로는 TenantTwo에 대한 데이터 액세스를 할 수 없기 때문 입니다.**</span>.

**Recap**: 이 마지막 단계에서 우리는 Product manager 서비스의 코드에 **테넌트 격리**의 모든 개념을 반영 했습니다. 인증된 토큰을 **테넌트 별 정책이 반영된 credential**로 교환하기 위한 함수를 추가한 다음 이를 DynamoDB 데이터 저장소에 액세스하는 데 사용했습니다. 이를 통해 **테넌트간 교차 액세스**를 방지 했습니다. 즉 공용 자원에서 IAM 정책을 통한 isolation이 동작하는 모습을 확인했습니다.
