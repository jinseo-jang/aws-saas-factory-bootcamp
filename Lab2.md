# Lab 2 – Building Multi-Tenant Microservices

### Overview

첫 번째 실습에서는 테넌트를 온보딩하고 사용자 Identity가 테넌트 Identity에 결합 된 **SaaS Identity**를 만들었습니다. 이제 여러분은 멀티 테넌트 방식의 애플리케이션에 이 SaaS Identity를 적용 하는 모습을 살펴볼 것 입니다.

지금까지 우리가 만든 서비스(tenant registration, user manager 등)는 모두 온보딩의 기본 서비스들 입니다. 이제 애플리케이션의 실제 기능(예, 데이터 조회, 생성 등)을 제공할 서비스를 살펴 보겠습니다. 이를 위해 매우 기본적인 주문 관리 시스템 시나리오를 활용할 겁니다. 제품 카탈로그를 만들고 해당 제품에 대해 주문을 생성 할겁니다. <span style="color:blue">멀티 테넌트 SaaS 솔루션으로서 이 기능을 구현하는 데 사용되는 컴퓨팅 및 스토리지 리소스가 모든 테넌트들에 의해 공유 되고 있다는 점을 주목해야 합니다.</span>

아래는 만들게 될 서비스 레벨의 다이어그램 입니다.

<p align="center"><img src="./images/lab2/arch_overview.png" alt="Lab 2 Architecture Overview"/></p>

이러한 서비스는 **API Gateway**를 통해 웹 애플리케이션에 연결되어 기본 **CRUD** 작업을 노출하여 제품과 주문을 만들고, 읽고, 업데이트하고, 삭제할 수 있습니다.

물론 이러한 서비스를 구현할 때 이러한 서비스에 어떻게 멀티 테넌시 개념을 적용 할지 고려해야합니다. 이러한 서비스는 어떤 테넌트의 요청인지를 인식 한 상태에서 데이터를 저장하고 메트릭을 기록하고 user identity를 획득 해야합니다. 따라서 이런 부분이 서비스에 어떻게 적용될 수 있을지 고민해야 합니다.

다음은 다이어그램은 멀티 테넌트 관점의 마이크로 서비스 구축하는 것이 무엇을 의미하는지 보여줍니다.

<p align="center"><img src="./images/lab2/multitenant_diagram.png" alt="Lab 2 Multi-Tenant Overview"/></p>

이 다소 단순화 된 다이어그램은 우리가 배포 할 멀티 테넌트 마이크로 서비스가 갖춰야 하는 요소를 강조해 보여줍니다. 다이어그램의 중앙에는 기능을 제공하는 서비스(이 경우 product 및 order manager)가 있습니다. 그리고 그 주변에는 멀티 테넌시에 포함되어야 하는 요소가 있습니다. 이 요소들은 다음과 같습니다:

- **Identity and Tenant Context** – 저희 서비스에는 테넌트 컨텍스트와 함께 현재 사용자의 역할 및 권한을 획득하기 위한 표준 방법이 필요합니다. 서비스 안의 거의 모든 작업은 테넌트 컨텍스트 안에서 발생하므로 해당 컨텍스트를 획득하고 적용하는 방법에 대해 생각해야합니다.
- **Multi-Tenant Data Partioning** – 두 서비스(product, order)는 데이터를 저장해야합니다. 즉, 스토리지 CRUD 작업은 모두 멀티 테넌트 모델에서 데이터를 분할, 유지 및 수집 하기 위해 테넌트 컨텍스트를 주입 해야 합니다.
- **Tenant Aware Logging, Metering, and Analytics** – 시스템의 활동에 대한 로그 및 메트릭을 기록 할 때 이러한 활동어 어떤 테넌트의 것인지 식별할 수 있어야 합니다. 따라서 서비스들은 향후 문제 해결 또는 분석을 위해 모든 이벤트/활동 메시지 에 이 테넌트 컨텍스트를 주입 해야합니다.

이런 요소들이 어떻게 만들어지고 동작하는지 이번 실습을 통해 살펴 볼겁니다. 물론 이것들을 모두 처음 부터 개발 하지는 않고 실습을 위해 미리 만들어진 것을 사용 할겁니다.

### 실습에서 만드는 것들

- <span style="color:blue">**Part1-단일 테넌트 제품 관리자 마이크로 서비스를 배포**</span> - 이것은 위에서 언급한 세 가지 멀티 테넌시의 요소를 반영 하기 전에 서비스가 어떻게 동작하 하는지 살펴 보는 단계 입니다. 즉 멀티 테넌트 관점이 반영되지 않은 기본적인 CRUD 서비스가 될 것입니다.
- <span style="color:blue">**Part2-멀티 테넌트 데이터 파티셔닝 도입**</span> – 멀티 테넌트 관점 도입의 첫 번째 단계는, 입수되는 요청에 포함된 테넌트 식별자를 기반으로 데이터를 파티셔닝하는 기능을 추가하는 것입니다.
- <span style="color:blue">**Part3-User identity 에서 Tenant Context 추출**</span> – 데이터 파티셔닝을 위해 사용될 Tenant identity를 추출 하고 적용하기 위해서 HTTP 호출에 포함된 보안 컨텍스트 사용하는 기능을 추가 합니다.
- <span style="color:blue">**Part4-파티셔닝 동작 확인을 위한 두 번째 테넌트 온보딩**</span> – 새 테넌트를 등록하고 해당 테넌트 컨텍스트를 통해 제품을 관리하여 시스템이 애플리케이션에서 데이터를 성공적으로 분할 하는 방법을 보여줍니다.
- <span style="color:blue">**Part5-테넌트 컨텍스트를 사용하여 데이터 로깅**</span> – 마지막 단계에서는 테넌트 중심 분석을 지원하기 위해 테넌트 컨텍스트를 로그 및 측정에 삽입하는 방법을 시연합니다.

이 프로세스가 마지막으로 완료되면 데이터 파티셔닝, User Identity 에서 테넌트 컨텍스트 추출 및 로그에 Tenant identity 삽입 기능을 모두 지원하는 완전한 멀티 테넌트 인식 제품 관리 서비스가 만들어져 있을겁니다.

## Part 1 - Deploying a Single-Tenant Product Manager Service

멀티 테넌트 애플리케이션을 만들기 전에, Product Manager 서비스를 먼저 싱글 테넌트 형태로 만들어 동작을 점검해보겠습니다. 이를 통해 나중에 멀티 테넌트 형태일 때 이 서비스가 어떻게 동작할지 가늠할 수 있을겁니다.

실습을 위해 마련된 배포 스크립트를 실행하여 서비스를 배포한 다음, 이 서비스가 사용할 **DynamoDB** 테이블을 만들 겁니다. 그리고 cURL을 활용해 이 서비스를 실행하여 동작을 확인 할겁니다. 참고로 이 서비스는 API Gateway를 통해 노출 되는데, 이 설정은 배포 스크립트에 의해 자동으로 완료 될겁니다.

**Step 1** - 먼저 배포할 product manager 서비스를 살펴보겠습니다. 이 서비스는 REST 인터페이스를 통해 일련의 CRUD 작업을 노출합니다. 이 파일의 소스 코드는`Lab2/Part1/app/source/product-manager/src/server.js`에서 확인할 수 있습니다.

코드를 보면 REST 메소드 (app.get (...), app.post (...) 등)에 해당하는 여러 진입 점이 표시됩니다. 이러한 각 함수에는 해당 REST 작업이 구현되어 있습니다. 여기서 무슨 일이 일어나고 있는지 이해하기 위해 메소드 중 하나를 자세히 살펴 보겠습니다.

`/product` 패스와 매핑된 GET 메소드를 살펴 보면 요청에 포함된 `params`으로 id를 추출한 다음 이를 함수의 나머지 부분에 포함된 DynamoDBHelper 호출에 실어 DynamoDB 테이블에서 item을 가져 옵니다.

```javascript
app.get("/product/:id", function (req, res) {
  winston.debug("Fetching product: " + req.params.id);
  // init params structure with request params
  var params = {
    productId: req.params.id,
  };
  tokenManager.getSystemCredentials(function (credentials) {
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

**Step 2** - 이제 Cloud9 IDE를 통해 싱글 테넌트 형태의 product manager를 배포해 보겠습니다. `Lab2/Part1/scripts` 디렉토리로 이동하여`product-manager-v1.sh`를 마우스 오른쪽 버튼으로 클릭 한 다음 **Run**을 클릭하여 셸 스크립트를 실행합니다.

<p align="center"><img src="./images/lab2/part1/cloud9_run_script.png" alt="Lab 2 Part 1 Step 2 Cloud9 Run Script"/></p>

**Step 3** - **Process exited with code : 0** 메시지 다음에 **STACK CREATE COMPLETE** 이 표시되며 `product-manager-v1.sh` 셸 스크립트가 성공적으로 완료될 때까지 기다립니다.

<p align="center"><img src="./images/lab2/part1/cloud9_run_script_complete.png" alt="Lab 2 Part 1 Step 3 Cloud9 Script Finished"/></p>

**Step 4** - 서비스가 정상적으로 배포 되었으니, 제품 정보를 저장하는 데 사용할 DynamoDB 테이블을 생성하겠습니다. 먼저 AWS 콘솔에서 DynamoDB 서비스로 이동하고 페이지 왼쪽 상단의 탐색 목록에서 **Tables** 옵션을 선택합니다.

<p align="center"><img src="./images/lab2/part1/dynamo_menu_tables.png" alt="Lab 2 Part 1 Step 4 DynamoDB Menu Tables"/></p>

**Step 5** - 이제 페이지 상단의 **Create table** 버튼을 클릭합니다. **ProductBootcamp**라는 테이블 이름을 입력하고 **productId**를 기본 키로 입력합니다. 테이블 및 키 이름은 DynamoDB에서 대소 문자를 구분합니다. 따라서 값을 올바르게 입력했는지 확인하세요. 값을 모두 입력한 후 페이지 오른쪽 하단의 **Create** 버튼을 선택하여 새 테이블을 만듭니다.

<p align="center"><img src="./images/lab2/part1/dynamo_create_table.png" alt="Lab 2 Part 1 Step 5 DynamoDB Create Table"/></p>

**Step 6** - 이제 테이블이 생성 되었고 서비스가 실행되고 **Amazon API Gateway**를 통해 액세스 할 수 있어야합니다. 서비스에 액세스 하기 위해서 API Gateway URL이 필요합니다. AWS 콘솔에서 API Gateway로 이동하고 **saas-bootcamp-api** API를 선택합니다. 이 API는 부트 캠프 환경에서 자동으로 생성 되었으며 실습에서 사용할 SaaS 애플리케이션을 실행하는 데 필요한 모든 리소스와 메소드를 포함합니다.

<p align="center"><img src="./images/lab2/part1/apigw_select_api.png" alt="Lab 2 Part 1 Step 6 API Gateway Select API"/></p>

**Step 7** - 왼쪽 메뉴에서 **Stages**를 선택한 다음 **v1** (version 1) 단계를 클릭합니다. 호출 URL이 표시됩니다. 모든 마이크로 서비스에 액세스 할 수있는 기본 URL (_ 끝에 **/v1** 포함_)입니다.

<p align="center"><img src="./images/lab2/part1/apigw_invoke_url.png" alt="Lab 2 Part 1 Step 6 API Gateway Select API"/></p>

**Step 8** - 이제 서비스를 직접 호출해보겠습니다. 이를 위해 **cURL** 또는 Postman (또는 원하는 도구)을 사용할 수 있습니다. 서비스가 실행 중인지 확인하기 위해`/product/health`에서 GET 메서드를 호출 해 보겠습니다. 다음 명령을 복사하여 붙여넣고 **INVOKE-URL**을 API Gateway 설정에서 캡처 한 URL 바꿉니다.

```bash
curl -w "\n" --header "Content-Type: application/json" INVOKE-URL/product/health
```

**URL 끝에 /product/health 그리고 앞에 API stage 이름을 포함했는지 확인합니다.** cURL \*\* 요청이 성공적으로 처리 되었으면 서비스가 요청을 처리 할 준비가되었음을 나타내는 JSON 형식의 성공 메시지를 받아야합니다.

**Step 9** - 이제 REST API를 통해 카탈로그에 새 제품을 추가 해보겠습니다. 다음 REST 호출을 통해 첫 ​​번째 제품을 생성합니다. 다음 명령을 복사하여 붙여 넣습니다.(스크롤하여 전체 명령을 선택해야 함). **INVOKE-URL**을 API Gateway 설정에서 캡처 한 URL로 바꿉니다.

```bash
curl -w "\n" --header "Content-Type: application/json" --request POST --data '{"sku": "1234", "title": "My Product", "description": "A Great Product", "condition": "Brand New", "conditionDescription": "New", "numberInStock": "1"}' INVOKE-URL/product
```

**Step 10** - 이제 데이터가 DynamoDB 테이블에 성공적으로 생성 되었는지 확인하겠습니다. AWS 콘솔에서 DynamoDB 서비스로 이동하고 페이지 왼쪽 상단에있는 옵션 목록에서 **Tables**을 선택합니다. 이제 페이지 중앙에 테이블 목록이 표시됩니다. **ProductBootcamp** 테이블을 찾아 테이블 이름에 있는 링크를 선택합니다. 테이블에 대한 기본 정보가 표시됩니다. 화면 상단에서 **Items** 탭을 선택하면 방금 추가 한 데이터가 포함 된 제품 테이블의 item 목록이 표시됩니다.

<p align="center"><img src="./images/lab2/part1/dynamo_table_items.png" alt="Lab 2 Part 1 Step 10 DynamoDB Table Items"/></p>

**Recap**: 이번 실습에서는 싱글 테넌트 용 product manager 서비스를 만들었습니다. 아직은 서비스에 내장 된 Identity 또는 테넌트 컨텍스트가 반영되지 않았습니다. 이제 이걸 바탕으로 멀티 테넌트 개념을 반영하여 서비스를 진화해 나가보겠습니다.

## Part 2 - Adding Multi-Tenant Data Partitioning

우리 서비스를 멀티 테넌트를 인식하도록 만드는 첫 번째 단계는 단일 DynamoDB 데이터베이스에서 여러 테넌트의 데이터를 유지할 수있도록 파티셔닝 모델을 구현하는 것입니다. 또한 REST 요청에 테넌트 컨텍스트를 주입하고 각 CRUD 작업에 이 테넌트 컨텍스트를 활용해야합니다.

이를 위해서 DynamoDB 데이터베이스의 파티션 키로 테넌트 식별자를 도입하는 설정이 필요합니다. 또한 각 REST 메서드에서 테넌트 식별자를 받아, DynamoDB 테이블에 액세스 할 때 이를 사용 하는 새로운 버전의 서비스가 필요합니다.

이어지는 단계들에서 product manager 서비스에 이러한 기능을 추가하겠습니다.

**Step 1** - 먼저 새로운 버전의 서비스가 배포가 필요합니다. 여기서 코드를 직접 수정하지는 않겠지만, 데이터 파티셔닝을 지원하기 위해 코드가 어떻게 변경되는지 간단히 살펴 보겠습니다. Cloud9 IDE에서 product manager server.js 파일을 엽니다. Cloud9에서`Lab2/Part2/product-manager/`로 이동하고`server.js`를 마우스 오른쪽 버튼으로 클릭 한 다음 **Open**를 클릭합니다.

<p align="center"><img src="./images/lab2/part2/cloud9_open_script.png" alt="Lab 2 Part 2 Step 1 Cloud9 Open Script"/></p>

이 파일은 이전 버전과 크게 다르지 않습니다. 실제로 여기서 유일한 변경 사항은 DynamoDBHelper에 제공하는 파라미터에 테넌트 식별자를 추가했다는 것입니다. 다음은 그 코드의 일부 입니다.

```javascript
app.post("/product", function (req, res) {
  var product = req.body;
  var guid = uuidv4();
  product.product_id = guid;
  product.tenant_id = req.body.tenant_id;
  winston.debug(JSON.stringify(product));
  // construct the helper object
  tokenManager.getSystemCredentials(function (credentials) {
    var dynamoHelper = new DynamoDBHelper(
      productSchema,
      credentials,
      configuration
    );
    dynamoHelper.putItem(product, credentials, function (err, product) {
      if (err) {
        winston.error("Error creating new product: " + err.message);
        res.status(400).send('{"Error": "Error creating product"}');
      } else {
        winston.debug("Product " + req.body.title + " created");
        res.status(200).send({ status: "success" });
      }
    });
  });
});
```

`product.tenant_id = req.body.tenant_id;`줄은 새로운 버전의 유일한 변경 사항 입니다. 들어오는 요청에서 테넌트 식별자를 추출하여 제품 객체에 추가합니다. 이것은 물론, 이 메서드에 대한 REST 호출 할 때마다 테넌트 식별자를 제공해야 함을 의미합니다.

**Step 2** - 지금까지는 **DynamoDBHelper**가 DynamoDB 테이블에서 item을 어떻게 가져 오는지 살펴보지 않았습니다. 이 모듈은 AWS 제공 DynamoDB 클라이언트의 래퍼에 isolation을 지원 하기 위한 몇 가지 요소가 추가된 형태 입니다. 사실 이 테넌트 식별자 모델을 도입더라도 DynamoDBHelper가이 요청을 처리하는 방식은 변경되지 않습니다. 다음은`getItem ()`메서드에 대한 DynamoDBHelper의 코드입니다.

```javascript
DynamoDBHelper.prototype.getItem = function (keyParams, credentials, callback) {
  this.getDynamoDBDocumentClient(
    credentials,
    function (error, docClient) {
      var fetchParams = {
        TableName: this.tableDefinition.TableName,
        Key: keyParams,
      };
      docClient.get(fetchParams, function (err, data) {
        if (err) {
          winston.debug(JSON.stringify(docClient.response));
          callback(err);
        } else {
          callback(null, data.Item);
        }
      });
    }.bind(this)
  );
};
```

제품 관리자 서비스에서 모든 매개 변수를`fetchParams` 구조의`keyParams` 값으로 전달하고 있음을 알 수 있습니다. 클라이언트는 단순히 이 매개 변수를 사용하여 테이블의 파티션 키를 일치시킵니다. **여기서 요점은 테넌트 식별자에 의한 분할을 지원하기 위해 코드에서 고유 한 작업이 수행되지 않는다는 것입니다. DynamoDB 테이블의 또 다른 키일뿐입니다.**

**Step 3** - 이제 데이터 파티셔닝을 구현하기 위해 이 서비스가 어떻게 변경되는지 이해 했으므로 Cloud9 IDE 내에서 product manager 버전 2를 배포 해 보겠습니다. `Lab2/Part2/product-manager` 디렉토리로 이동하여`deploy.sh`를 마우스 오른쪽 버튼으로 클릭 한 다음 **Run**을 클릭하여 셸 스크립트를 실행합니다.

<p align="center"><img src="./images/lab2/part2/cloud9_run_script.png" alt="Lab 2 Part 2 Step 3 Cloud9 Run Script"/></p>

**Step 4** - `deploy.sh`이 성공적으로 완료 될때 까지 기다립니다.

<p align="center"><img src="./images/lab2/part2/cloud9_run_script_complete.png" alt="Lab 2 Part 2 Step 4 Cloud9 Script Finished"/></p>

**Step 5** - 이 새로운 파티션 체계를 사용하여 DynamoDB 테이블의 구성도 변경 해야합니다. 기억 하겠지만, 이 테이블은 **product_id**를 파티션 키로 사용했습니다. 이제 **tenant_id**를 파티션 키로 지정하고 **product_id**를 보조 인덱스로 사용해야합니다(여전히 해당 값을 기준으로 정렬 할 수 있으므로). 이 변경 사항을 적용하는 가장 쉬운 방법은 기존 **ProductBootcamp** 테이블을 **_delete_**하고 올바른 구성으로 새 테이블을 만드는 것입니다.

AWS 콘솔에서 DynamoDB 서비스로 이동하고 페이지 왼쪽 상단에있는 메뉴에서 **Tables** 옵션을 선택합니다. **ProductBootcamp** 테이블의 라디오 버튼을 선택합니다. 제품 표를 선택한 후 **Delete table** 버튼을 선택합니다. 프로세스를 완료하려면 CloudWatch 경보 제거를 확인하라는 메시지가 표시됩니다.

<p align="center"><img src="./images/lab2/part2/dynamo_delete_table.png" alt="Lab 2 Part 2 Step 5 DynamoDB Delete Table"/></p>

**Step 6** - 이제 다시 새로운 테이블을 만듭니다. 페이지 상단에서 **Create table** 버튼을 선택합니다. 이전과 마찬가지로 테이블 이름으로 **ProductBootcamp**를 입력 하지만 이번에는 파티션 키로 **tenant_id**를 입력합니다. 이제 **Add sort key** 확인란을 클릭하고 정렬 키로 **product_id**를 입력합니다. **Create** 버튼을 클릭합니다.

<p align="center"><img src="./images/lab2/part2/dynamo_create_table.png" alt="Lab 2 Part 2 Step 6 DynamoDB Create Table"/></p>

**Step 7** - 앞에 까지, 서비스가 테넌트 식별자를 반영할 수 있도록 수정 했고, 이 테넌트 식별자로 데이터를 파티셔닝 하도록 DynamoDB 테이블도 수정했습니다. 이제 새 버전의 서비스가 실행 중인지 확인할 때입니다. 다음 cURL 명령을 실행하여 서비스 health 체크를 호출하십시오. API Gateway URL을 찾아야하는 경우 Part 1를 다시 참조하십시오. 다음 명령을 복사하여 붙여넣고 **INVOKE-URL**을 API Gateway 설정에서 캡처 한 URL로 바꿉니다.

```bash
curl -w "\n" --header "Content-Type: application/json" INVOKE-URL/product/health
```

**Step 8** - 이제 서비스가 실행 중임을 확인 했으면, REST API를 통해 카탈로그에 새 제품을 추가 할 수 있습니다. 이전 REST 호출과 달리 이 호출은 요청의 일부로 테넌트 식별자를 제공해야합니다. 다음 REST 명령을 제출하여 "**123**"테넌트에 대한 제품을 생성 합니다. 아래 명령을 복사하여 붙여 넣습니다(스크롤하여 전체 명령을 선택해야 함). **INVOKE-URL**을 API Gateway 설정에서 캡처 한 URL로 바꿉니다.

```bash
curl -w "\n" --header "Content-Type: application/json" --request POST --data '{"tenant_id": "123", "sku": "1234", "title": "My Product", "description": "A Great Product", "condition": "Brand New", "conditionDescription": "New", "numberInStock": "1"}' INVOKE-URL/product
```

앞선 제품 생성 테스트와 유사해 보이지만 전달하는 body에 `tenant_id` ("123") 이 포함되어 있습니다.

**Step 9** - 이 데이터가 성공적으로 생성 되었는지 확인하기 전에 다른 테넌트의 제품도 생성하겠습니다. 이것은 저희가 만든 파티션 체계가 각 테넌트 별로 데이터를 저장할 수 있는지 확인하기 위해서 입니다. 다른 테넌트에 해당 하는 제품을 추가하려면 다른 테넌트에 대해 다른 POST 명령을 실행하기 만하면됩니다. 테넌트 "**456**"에 대해 다음 POST를 제출합니다. 다음 명령을 복사하여 붙여 넣습니다 (스크롤하여 전체 명령을 선택해야 함). **INVOKE-URL**을 API Gateway 설정에서 캡처 한 URL로 바꿉니다.

```bash
curl -w "\n" --header "Content-Type: application/json" --request POST --data '{"tenant_id": "456", "sku": "1234", "title": "My Product", "description": "A Great Product", "condition": "Brand New", "conditionDescription": "New", "numberInStock": "1"}' INVOKE-URL/product
```

**Step 19** - 제출 한 데이터가 생성 한 DynamoDB 테이블에 성공적으로 생성되었는지 확인 하겠습니다. AWS 콘솔에서 DynamoDB 서비스로 이동하고 페이지 왼쪽 상단에있는 옵션 목록에서 **Tables**을 선택합니다. 이제 페이지 중앙에 테이블 목록이 표시됩니다. **ProductBootcamp** 테이블을 찾아 테이블 이름이있는 링크를 선택합니다. 테이블에 대한 기본 정보가 표시됩니다. 이제 화면 상단에서 **Items** 탭을 선택하면 방금 추가 한 두 데이터를 포함하는 item 목록이 표에 표시됩니다. 이 두 item이 존재하고 사용자가 제공 한 두 개의 테넌트 식별자 ( "123"및 "456")를 기반으로 정상적으로 파티셔닝 되어 저장 되었는지 확인합니다.

<p align="center"><img src="./images/lab2/part2/dynamo_scan_table.png" alt="Lab 2 Part 2 Step 10 DynamoDB Scan Table"/></p>

**Recap** : 이제 서비스에 데이터 파티셔닝을 성공적으로 도입했습니다. product manager 서비스에 테넌트 식별자 매개 변수를 추가하고 제품 테이블에 **tenant_id**를 파티션 키로 추가 했습니다. 이제 모든 CRUD 작업이 멀티 테넌트 개념을 인식 할겁니다.

## Part 3 - Applying Tenant Context

테넌트 식별자가 product manager 서비스에 대한 모든 호출에 매개 변수로 잘 전달 될 것이라고 기대하는 것은 실용적이거나 안전하지 않습니다. 이 대신에, 테넌트 컨텍스트는 이전 랩에서 설정 한 인증 프로세스의 일부인 토큰에서 가져오도록 하는게 효과적입니다.

**따라서 다음 단계는 서비스가 이러한 보안 토큰을 인식하고 이러한 토큰에서 테넌트 컨텍스트를 추출 할 수 있도록 만드는 것입니다. 그런 다음 방금 설정 한 파티셔닝은, 온보딩 과정에서 생성되어 각 HTTP request header로 제품 관리자 서비스로 전달 된 테넌트 식별자를 통해 달성할 수 있습니다.**

이 섹션에서는 HTTP 요청에서 이러한 토큰을 추출하여 보안 및 데이터 파티셔닝 모델에 적용하기 위해 product manager 서비스가 새 코드로 어떻게 개조 되는지 살펴 보겠습니다.

**Step 1** - 진행을 위해 새로운 버전의 서비스가 필요합니다. 코드를 직접 수정하지는 않겠지 만, Identity 토큰에서 테넌트 컨텍스트 획득을 지원하기 위해 코드가 어떻게 변경되는지 간단히 살펴 보겠습니다. `Lab2/Part3/product-manager/server.js`를 열어 Cloud9에서 새로윤 버전의 Product Manager 서비스를 확인합니다.

<p align="center"><img src="./images/lab2/part3/cloud9_open_script.png" alt="Lab 2 Part 3 Step 1 Cloud9 Open Script"/></p>

Product manager 서비스의 버전 3에는 토큰 처리 작업을 추상화하는 새로운 **TokenManager** helper 개체가 추가되었습니다. 잠깐 이 업데이트 된 버전의 코드를 살펴보고 사용자의 Identity에서 테넌트 컨텍스트를 얻는 방법을 살펴 보겠습니다:

```javascript
app.use(function (req, res, next) {
  res.header("Access-Control-Allow-Origin", "*");
  res.header(
    "Access-Control-Allow-Methods",
    "GET, POST, OPTIONS, PUT, PATCH, DELETE"
  );
  res.header(
    "Access-Control-Allow-Headers",
    "Content-Type, Access-Control-Allow-Headers, Authorization, X-Requested-With"
  );
  bearerToken = req.get("Authorization");
  if (bearerToken) {
    tenantId = tokenManager.getTenantId(req);
  }
  next();
});
```

여기에 표시된 TokenManager의 `getTenantId`함수는 저희 애플리케이션 대부분의 서비스에 나타나면서 각 REST 요청에 대해 **HTTP headers**에서 테넌트 식별자를 획득하는 메커니즘을 제공합니다. 저희가 사용하는 샘플 애플리케이션은 Express 프레임 워크의 미들웨어 구성을 사용하여 이를 구현 했습니다. 이 미들웨어를 사용하면 각 REST 메서드에 대한 함수가 처리하기 전에 각 HTTP 요청을 가로 채서 사전 처리하는 함수를 도입 할 수 있습니다.

응답 헤더가 설정된 후 **HTTP Authorization** 헤더에서 **bearer token** 이 추출되는 것을 볼 수 있습니다. 이 토큰은 테넌트 컨텍스트를 획득 하기 위한 데이터를 보유 하고 있습니다. 그런 다음 **TokenManager** helper를 사용하여 request에서 테넌트 식별자를 가져옵니다. 그러면 이어서 이 함수에 대한 호출은 테넌트 식별자를 반환하고 이를`tenantId` 변수에 할당합니다. 이 변수는 테넌트 컨텍스트를 획득하기 위해 서비스 전체에서 참조됩니다.

**Step 2** - 이제 테넌트 식별자를 얻었으므로 이를 적용하는 것은 비교적 간단합니다. 아래 코드 일부를 보듯이, 1 단계에서 미들웨어의 bearer token 에서 추출한 **tenantId**를 참조하여 테넌트 식별자를 획득하는 방식을 적용한걸 알수 있습니다. (수신 요청에서 가져 오는 대신).

```javascript
app.get("/product/:id", function (req, res) {
  winston.debug("Fetching product: " + req.params.id);
  // init params structure with request params
  var params = {
    tenant_id: tenantId,
    product_id: req.params.id,
  };
  tokenManager.getSystemCredentials(function (credentials) {
    // construct the helper object
    var dynamoHelper = new DynamoDBHelper(
      productSchema,
      credentials,
      configuration
    );
    dynamoHelper.getItem(params, credentials, function (err, product) {
      if (err) {
        winston.error("Error getting product: " + err.message);
        res.status(400).send('{"Error": "Error getting product"}');
      } else {
        winston.debug("Product " + req.params.id + " retrieved");
        res.status(200).send(product);
      }
    });
  });
});
```

**Step 3** - 여기에서 대부분의 토큰 처리 작업은 의도적으로 helper class에 포함되어 있습니다. 토큰에서 이 컨텍스트를 추출하는 방법을 알아보기 위해 해당 클래스의 내용을 간략히 살펴 보겠습니다.

다음은 토큰을 추출하기 위해 호출되는 TokenManager의 코드 일부 입니다. 이 함수는 HTTP 요청의 Authorization 헤더에서 보안 토큰을 추출하여 디코딩 한 다음 디코딩 된 토큰에서 tenantId를 획득합니다. _프로덕션 환경에서 이 언패킹은 보안을 위해 서명 된 인증서를 사용하여 토큰 내용이 전송 중에 수정되지 않았음을 컨펌 합니다_.

```javascript
module.exports.getTenantId = function (req) {
  var tenantId = "";
  var bearerToken = req.get("Authorization");
  if (bearerToken) {
    bearerToken = bearerToken.substring(bearerToken.indexOf(" ") + 1);
    var decodedIdToken = jwtDecode(bearerToken);
    if (decodedIdToken) {
      tenantId = decodedIdToken["custom:tenant_id"];
    }
  }
  return tenantId;
};
```

**Step 4** - 이제 HTTP 헤더를 통해 테넌트 컨텍스트를 도입 하는 방법을 이해 했으므로 Cloud9 IDE 내에서 product manager 버전 3을 배포 합니다. `Lab2/Part3/product-manager/`디렉토리로 이동하여`deploy.sh`를 마우스 오른쪽 버튼으로 클릭 한 다음 **Run**을 클릭하여 셸 스크립트를 실행합니다.

<p align="center"><img src="./images/lab2/part3/cloud9_run_script.png" alt="Lab 2 Part 3 Step 4 Cloud9 Run Script"/></p>

**Step 5** - `deploy.sh` 이 성공적으로 완료될때 까지 기다립니다.

<p align="center"><img src="./images/lab2/part3/cloud9_run_script_complete.png" alt="Lab 2 Part 3 Step 5 Cloud9 Script Finished"/></p>

**Step 6** - 이제 애플리케이션이 배포 되었으므로 이 새로운 테넌트 및 보안 컨텍스트가 처리되는 모습을 확인하겠습니다. 이를 위해 최소 두 명의 tenant가 등록 되어 있는지 확인해야합니다.

이미 Lab 1에서 테넌트를 하나 등록 했습니다. Lab 1과 동일한 단계에 따라 두 번째 테넌트를 추가하겠습니다. 애플리케이션의 URL(Lab 1에서 생성됨)을 입력하고 로그인 화면이 나타나면 **Register** 버튼을 선택합니다. **CloudFront** 서비스에서 애플리케이션의 URL을 캡처 해야하는 경우 Lab 1을 참조하십시오.

**Step 7** - 새 테넌트에 대한 데이터로 가입을 완료 합니다. 두 개의 테넌트를 생성하므로 **두 개의 별도** 이메일 주소가 필요합니다. 두 개가 없는 경우 lab 1에 설명 된대로 **@** 기호 앞의 사용자 이름에 더하기 (**+**) 기호를 사용하는 트릭을 사용할 수 있습니다. 양식에서 **Register** 버튼을 선택합니다.

**Step 8** - lab 1과 마찬가지로 이제 이메일에서 Cognito에서 보낸 등록 확인 메시지를 확인합니다. 받은 편지함에서 임시 암호 (Cognito에서 생성)와 함께 사용자 이름 (이메일 주소)이 포함 된 메시지를 찾아야합니다. 메시지는 다음과 유사합니다.

<p align="center"><img src="./images/lab2/part3/cognito_email.png" alt="Lab 2 Part 3 Step 8 Cognito Validation Email"/></p>

**Step 9** - 이제 애플리케이션에 로그인 할 수 있습니다. 공개 URL을 사용하여 애플리케이션으로 돌아갑니다 (실습 1에서 생성됨). 이메일에 제공된 임시 자격 증명을 입력하고 **Login** 버튼을 선택합니다.

<p align="center"><img src="./images/lab2/part3/login.png" alt="Lab 2 Part 3 Step 9 Login"/></p>

**Step 10** - 새 비밀번호를 만들고 **Confirm** 버튼을 선택합니다.

<p align="center"><img src="./images/lab2/part3/change_password.png" alt="Lab 2 Part 3 Step 10 Reset Password"/></p>

암호를 성공적으로 변경하면 애플리케이션에 로그인되고 홈 화면으로 이동될 겁니다.

**Step 11** - 실습을 완료 하려면 두 명의 테넌트가 있어야합니다. 테넌트가 하나만 등록 된 경우 6-10 단계를 다시 반복하여 테넌트에 대해 다른 이메일 주소를 제공하여 다른 테넌트를 만듭니다.

**Step 12** - 이제 온보딩을 통해 새로운 테넌트들이 생성되었으므로 실제로 애플리케이션을 통해 일부 제품을 생성 해 보겠습니다. 첫 번째 테넌트로 애플리케이션에 로그인하고 페이지 상단의 **Catalog** 메뉴 항목으로 이동합니다.

<p align="center"><img src="./images/lab2/part3/catalog.png" alt="Lab 2 Part 3 Step 12 Catalog Page"/></p>

**Step 13** - **Catalog** 페이지가 열린 상태에서 페이지 오른쪽 상단의 **Add Product** 버튼을 선택합니다. 그리고 세부 정보를 입력하십시오. 그러나 **SKU**의 경우 테넌트 별로 구분을 위해서 "**TENANTONE-ABC**"와 같이 **TENANTONE**을 앞에 추가합니다.

<p align="center"><img src="./images/lab3/part1/add_product1.png" alt="Lab 3 Part 1 Step 7 Add Product"/></p>

**Step 14** - 테넌트 중 하나에 대해 두 가지 제품을 추가 했으면 화면 오른쪽 상단에서 테넌트 이름이있는 드롭 다운 메뉴를 선택하고 **Logout**을 선택합니다. 로그인 페이지로 돌아갑니다.

<p align="center"><img src="./images/lab2/part3/logout.png" alt="Lab 2 Part 3 Step 14 Logout"/></p>

**Step 15** - 생성 한 다른 테넌트의 자격 증명을 입력하고 **Login** 버튼을 선택합니다. 이제 다른 테넌트로 로그인 했으며 화면 오른쪽 상단의 프로필 메뉴 선택에 다른 이름이 표시되어야 합니다.

**Step 16** - 이제 **Catalog** 보기로 다시 이동합니다. 제품 목록이 비어 있습니다. 이전에 생성 한 제품은 다른 테넌트와 연결 되었으므로 여기에 표시되지 않습니다. **이는 파티셔닝이 작동하고 있음을 보여줍니다**.

**Step 17** - 이전과 마찬가지로 **Add Product** 버튼을 클릭하여 제품 세부 정보를 입력합니다. SKU의 경우 앞에 **TENANTTWO**를 추가합니다. 이는 테넌트에 속하는 제품을 명확하게 식별하도록 SKU 앞에 _구분자_ 값을 추가하는 것입니다.

<p align="center"><img src="./images/lab3/part1/add_product2.png" alt="Lab 3 Part 1 Step 13 Add Product"/></p>

**Step 18** - 두 개의 개별 테넌트에 대해 온보딩을 완료하고 제품을 추가 한 후, 이제 이 데이터가 DynamoDB 테이블에 어떻게 저장되었는지 확인 합니다.

AWS 콘솔에서 **DynamoDB** 서비스로 이동하고 페이지 왼쪽 상단의 옵션 목록에서 **Tables**을 선택합니다. **ProductBootcamp** 테이블을 선택한 다음 **Items** 탭을 선택합니다. 테이블은 `tenant_id`로 분할되어 있습니다. 두 개의 서로 다른 테넌트로 로그인 한 상태에서 웹 애플리케이션을 통해 입력 한 제품을 볼 수 있어야합니다 (실습의 앞부분에서 cURL을 통해 REST API를 호출하여 생성한 제품과는 별개).

<p align="center"><img src="./images/lab2/part3/product_table.png" alt="Lab 2 Part 3 Step 18 Product Table"/></p>

**Step 19** - **TenantBootcamp** 테이블을 살펴보면, 온보딩 한 테넌트의 item들이 표시 되어야 하며 **ProductBootcamp** 테이블의 **tenant_id** 필드에 자동으로 생성 된 GUID가 **tenant_id**로 들어가 있어야 합니다.

**Recap** : 이제 Authorization HTTP 헤더에 전달 된 보안 토큰에서 **"custom claim"** 을 추출하여 테넌트 컨텍스트를 획득하는 메커니즘을 만들었습니다. 또한 request를 가로 테넌트 컨텍스트를 획득하는 TokenManager helper 클래스를 만들어 복잡성을 줄였습니다.

[Continue to Lab 3](Lab3.md)
