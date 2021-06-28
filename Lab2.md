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

- **단일 테넌트 제품 관리자 마이크로 서비스를 배포** - 이것은 위에서 언급한 세 가지 멀티 테넌시의 요소를 반영 하기 전에 서비스가 어떻게 동작하 하는지 살펴 보는 단계 입니다. 즉 멀티 테넌트 관점이 반영되지 않은 기본적인 CRUD 서비스가 될 것입니다.
- **멀티 테넌트 데이터 파티셔닝 도입** – 멀티 테넌트 관점 도입의 첫 번째 단계는, 입수되는 요청에 포함된 테넌트 식별자를 기반으로 데이터를 파티셔닝하는 기능을 추가하는 것입니다.
- **User identity 에서 Tenant Context 추출** – 데이터 파티셔닝을 위해 사용될 Tenant identity를 추출 하고 적용하기 위해서 HTTP 호출에 포함된 보안 컨텍스트 사용하는 기능을 추가 합니다.
- **파티셔닝 동작 확인을 위한 두 번째 테넌트 온보딩** – 새 테넌트를 등록하고 해당 테넌트 컨텍스트를 통해 제품을 관리하여 시스템이 애플리케이션에서 데이터를 성공적으로 분할 하는 방법을 보여줍니다.
- **테넌트 컨텍스트를 사용하여 데이터 로깅** – 마지막 단계에서는 테넌트 중심 분석을 지원하기 위해 테넌트 컨텍스트를 로그 및 측정에 삽입하는 방법을 시연합니다.

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

**Be sure you've included the API stage name at the end of the URL _before_ /product/health**. You should get a JSON formatted success message from the **cURL** command indicating that the request was successfully processed and the service is ready to process requests.

**Step 9** - Now that we know the service is up-and-running, we can add a new product to the catalog via the REST API. Submit the following REST command to create your first product. Copy and paste the following command (be sure to scroll to select the entire command), replacing **INVOKE-URL** with the URL and trailing stage name you captured from the API Gateway settings.

```bash
curl -w "\n" --header "Content-Type: application/json" --request POST --data '{"sku": "1234", "title": "My Product", "description": "A Great Product", "condition": "Brand New", "conditionDescription": "New", "numberInStock": "1"}' INVOKE-URL/product
```

**Step 10** - Let's now go verify that the data we submitted landed successfully in the DynamoDB table we created. Navigate to the DynamoDB service in the AWS console and select **Tables** from the list of options at the upper left-hand side of the page. The center of the page should now display a list of tables. Find your **ProductBootcamp** table and select the link with the table name. This will display basic information about the table. Select the **Items** tab from the top of the screen, you'll see the list of items in your product table, which should include the item you just added.

<p align="center"><img src="./images/lab2/part1/dynamo_table_items.png" alt="Lab 2 Part 1 Step 10 DynamoDB Table Items"/></p>

**Recap**: This initial exercise illustrates a single-tenant version of the product manager service. It does not have identity or tenant context built into the service. In many respects, this represents the flavor of service you'd see in many non-SaaS environments. It gives us a good base for thinking about how we can now evolve the service to incorporate multi-tenant concepts.

## Part 2 - Adding Multi-Tenant Data Partitioning

The first step in making our service multi-tenant aware is to implement a partitioning model where we can persist data from multiple tenants in our single DynamoDB database. We'll also need to inject tenant context into our REST requests and leverage this context for each of our CRUD operations.

To make this work, will need a different configuration for our DynamoDB database, introducing a tenant identifier as the partition key. We'll also need a new version of our service that accepts a tenant identifier in each of the REST methods and applies it as it accesses DynamoDB tables.

The steps that follow will take you through the process of adding these capabilities to the product manager service:

**Step 1** - For this iteration, we'll need a new version of our service. While we won't modify the code directly, we'll take a quick look at how the code changes to support data partitioning. Open our product manager server.js file in our Cloud9 IDE. In Cloud9, navigate to `Lab2/Part2/product-manager/`, right-click `server.js` and click **Open**.

<p align="center"><img src="./images/lab2/part2/cloud9_open_script.png" alt="Lab 2 Part 2 Step 1 Cloud9 Open Script"/></p>

This file doesn't look all that different than our prior version. In fact, the only change here is that we've added a tenant identifier to the parameters that we supply to the DynamoDBHelper. Below is a snippet of the code from our file.

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

The line `product.tenant_id = req.body.tenant_id;` represents the only change you'll see between this version and the original. It extracts the tenant identifier from the incoming request and adds it to our product object. This, of course, means that REST calls to this method must supply a tenant identifier with each invocation of this method.

**Step 2** - Up to this point, we haven't really looked at the **DynamoDBHelper** to see how it accommodates our ability to get items from the DynamoDB table. This module is a wrapper of the AWS-provided DynamoDB client with some added elements to support isolation. In fact, even as we're introducing this tenant identifier model, it does not change how DynamoDBHelper processes this request. Below is a snipped of code from the DynamoDBHelper for the `getItem()` method:

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

You'll notice that we're passing through all the parameters that we constructed in our product manager service as the `keyParams` value in the `fetchParams` structure. The client will simply use the parameters to match the partition key for the table. The takeaway here is that nothing unique is done in the code to support the partitioning by a tenant identifier. It's simply just another key in our DynamoDB table.

**Step 3** - Now that you have a better sense of how this service changes to accommodate data partitioning, let's go ahead and deploy version 2 of the product manager, within our Cloud9 IDE. Navigate to `Lab2/Part2/product-manager` directory, right-click `deploy.sh`, and click **Run** to execute the shell script.

<p align="center"><img src="./images/lab2/part2/cloud9_run_script.png" alt="Lab 2 Part 2 Step 3 Cloud9 Run Script"/></p>

**Step 4** - Wait for the `deploy.sh` shell script to execute successfully.

<p align="center"><img src="./images/lab2/part2/cloud9_run_script_complete.png" alt="Lab 2 Part 2 Step 4 Cloud9 Script Finished"/></p>

**Step 5** - With this new partitioning scheme, we must also change the configuration of our DynamoDB table. If you recall, the current table used **product_id** as the partition key. We now need to have **tenant_id** be our partition key and have the **product_id** serve as a secondary index (since we may still want to sort on that value). The easiest way to introduce this change is to simply **_delete_** the existing **ProductBootcamp** table and create a new one with the correct configuration.

Navigate to the DynamoDB service in the AWS console and select the **Tables** option from the menu at the top left of the page. Select the radio button for the **ProductBootcamp** table. After selecting the product table, select the **Delete table** button. You will be prompted to confirm removal of CloudWatch alarms to complete the process.

<p align="center"><img src="./images/lab2/part2/dynamo_delete_table.png" alt="Lab 2 Part 2 Step 5 DynamoDB Delete Table"/></p>

**Step 6** - Now we can start the table creation process from scratch. Select the **Create table** button from the top of the page. As before, enter **ProductBootcamp** for the table name, but this time enter **tenant_id** for the partition key. Now click on the **Add sort key** checkbox and we'll enter **product_id** as the sort key. Click the **Create** button to finalize the table.

<p align="center"><img src="./images/lab2/part2/dynamo_create_table.png" alt="Lab 2 Part 2 Step 6 DynamoDB Create Table"/></p>

**Step 7** - The service has now been modified to support the introduction of a tenant identifier and we've modified DynamoDB to partition the data with this tenant identifier. It's time now to validate that the new version of the service is up-and-running. Issue the following cURL command to invoke the health check on the service. Refer to Part 1 if you need to find your API Gateway URL. Copy and paste the following command, replacing **INVOKE-URL** with the URL and trailing stage name you captured from the API Gateway settings.

```bash
curl -w "\n" --header "Content-Type: application/json" INVOKE-URL/product/health
```

**Be sure you've included the API stage name at the end of the URL before /product/health.** You should get a JSON formatted success message from the **cURL** command indicating that the request was successfully processed and the service is ready to process requests.

**Step 8** - Now that we know the service is up-and-running, we can add a new product to the catalog via the REST API. Unlike our prior REST call, this one must provide the tenant identifier as part of the request. Submit the following REST command to create a product for tenant "**123**". Copy and paste the following command (be sure to scroll to select the entire command), replacing **INVOKE-URL** with the URL and trailing stage name you captured from the API Gateway settings.

```bash
curl -w "\n" --header "Content-Type: application/json" --request POST --data '{"tenant_id": "123", "sku": "1234", "title": "My Product", "description": "A Great Product", "condition": "Brand New", "conditionDescription": "New", "numberInStock": "1"}' INVOKE-URL/product
```

This looks much like the prior example. However, notice that we pass a parameter of `tenant_id` ("123") in the body.

**Step 9** - Before we verify that this data was successfully written, let's introduce another product for a different tenant. This will highlight the fact that our partitioning scheme can store data separately for each tenant. To add another product for a different tenant, we just issue another POST command for a different tenant. Submit the following POST for tenant "**456**". Copy and paste the following command (be sure to scroll to select the entire command), replacing **INVOKE-URL** with the URL and trailing stage name you captured from the API Gateway settings.

```bash
curl -w "\n" --header "Content-Type: application/json" --request POST --data '{"tenant_id": "456", "sku": "1234", "title": "My Product", "description": "A Great Product", "condition": "Brand New", "conditionDescription": "New", "numberInStock": "1"}' INVOKE-URL/product
```

**Step 10** - Let's go verify that the data we submitted landed successfully in the DynamoDB table we created. Navigate to the DynamoDB service in the AWS console and select **Tables** from the list of options at the upper left-hand side of the page. The center of the page should now display a list of tables. Find your **ProductBootcamp** table and select the link with the table name. This will display basic information about the table. Now select the **Items** tab from the top of the screen and you'll see the list of items in your table which should include the two items you just added. Verify that these two items exist and are partitioned based on the two tenant identifiers that you suppled ("123" and "456").

<p align="center"><img src="./images/lab2/part2/dynamo_scan_table.png" alt="Lab 2 Part 2 Step 10 DynamoDB Scan Table"/></p>

**Recap**: You've now successfully introduced data partitioning into your service. The microservice achieved this by adding a tenant identifier parameter to the product manager service and changing the product table to use **tenant_id** as the partition key. Now, all of your CRUD operations are multi-tenant aware.

## Part 3 - Applying Tenant Context

At this stage we have data partitioning, but our way of introducing the tenant context was somewhat crude. It's simply not practical or secure to expect that tenant identifiers are to be passed as a parameter to every call to the product management service. Instead, this tenant context should come from the tokens that are part of the authentication process that we setup in the prior lab.

So, our next step is to enable our service to be aware of these security tokens and extract our tenant context from these tokens. Then, our partitioning that we just setup can rely on a tenant identifier that was provisioned during onboarding and simply flowed through to your product manager service in the header of each HTTP request.

For this section, we'll see how our product manager service gets retrofitted with new code to extract these tokens from the HTTP request and applies them to our security and data partitioning model.

**Step 1** - For this iteration, we'll need a new version of our service. While we won't modify the code directly, we'll take a quick look at how the code changes to support acquiring tenant context from identity tokens. View the new version of our Product Manager service in Cloud9 by opening `Lab2/Part3/product-manager/server.js`.

<p align="center"><img src="./images/lab2/part3/cloud9_open_script.png" alt="Lab 2 Part 3 Step 1 Cloud9 Open Script"/></p>

Version 3 of our product manager service introduces a new **TokenManager** helper object that abstracts away many aspects of the token processing. Let's take a look at a snippet of this updated version to see how tenant context is acquired from the user's identity:

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

The TokenManager's `getTenantId` function shown here, which appears in most of the services of our application, provides the mechanism for acquiring the tenant identifier from the **HTTP headers** for each REST request. It achieves this by using the middleware construct of the Express framework. This middleware allows you to introduce a function that intercepts and pre-processes each HTTP request before it is processed by the functions for each REST method.

You'll see after the response headers are set, the bearer token is extracted from the **HTTP Authorization** header. This token holds the data we want to use to acquire our tenant context. We then use the **TokenManager** helper to get the tenant identifier out of the request. The call to this function returns the tenant identifier and assigns it to the `tenantId` variable. This variable is then referenced throughout the service to acquire tenant context.

**Step 2** - Now that you have the tenant identifier, the application of this is relatively straightforward. You can see that we've changed the way we're acquiring the tenant identifier, referencing the **tenantId** that we extracted from the bearer token in the middleware in Step 1 (instead of pulling this from the incoming requests).

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

**Step 3** - As you can imagine, most of the token processing work here is intentionally embedded in the helper class. Let's take a quick look at what is in that class to see how it's extracting this context from the tokens.

Below is a snippet of the code from the TokenManager that is invoked to extract the token. This function extracts the security token from the Authorization header of the HTTP request, decodes it, then acquires the tenantId from the decoded token. _In a production environment, this unpacking would use a signed certificate as a security measure to ensure the token contents were not modified during transmission_.

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

**Step 4** - Now that you have a better sense of how we've introduced tenant context through HTTP headers, let's go ahead and deploy version 3 of the product manager, within our Cloud9 IDE. Navigate to `Lab2/Part3/product-manager/` directory, right-click `deploy.sh`, and click **Run** to execute the shell script.

<p align="center"><img src="./images/lab2/part3/cloud9_run_script.png" alt="Lab 2 Part 3 Step 4 Cloud9 Run Script"/></p>

**Step 5** - Wait for the `deploy.sh` shell script to execute successfully.

<p align="center"><img src="./images/lab2/part3/cloud9_run_script_complete.png" alt="Lab 2 Part 3 Step 5 Cloud9 Script Finished"/></p>

**Step 6** - Now that the application is deployed, it's time to see how this new tenant and security context gets processed. We'll need to have a valid token for our service to be able to succeed. That means returning our attention to the web application, which already has the ability to authenticate a user and acquire a valid token from our identity provider, Cognito. First, we'll need to make sure we have at least two tenants registered.

You already registered a tenant in Lab 1. Let's add a second tenant following the same steps as in Lab 1. Enter the URL to your application (created in Lab 1) and select the **Register** button when the login screen appears. Refer to Lab 1 if you need to capture the URL for your application from the **CloudFront** service.

**Step 7** - Fill in the form with data about your new tenant. Since we're creating two tenants as part of this flow, you'll need **two separate** email addresses. If you don't have two, you can use the same trick with the plus (**+**) symbol in the username before the at (**@**) symbol as described in Lab 1. After you've filled in the form, select the **Register** button.

**Step 8** - Just like in Lab 1, you'll now check your email for the validation message that was sent by Cognito. You should find a message in your inbox that includes your username (your email address) along with a temporary password (generated by Cognito). The message will be similar to the following:

<p align="center"><img src="./images/lab2/part3/cognito_email.png" alt="Lab 2 Part 3 Step 8 Cognito Validation Email"/></p>

**Step 9** - We can now login to the application using these credentials. Return to the application using the public URL (created in Lab 1). Enter the temporary credentials that were provided in your email and select the **Login** button.

<p align="center"><img src="./images/lab2/part3/login.png" alt="Lab 2 Part 3 Step 9 Login"/></p>

**Step 10** - The system will detect that this is a temporary password and indicate that you need to setup a new password for your account. To do this, application redirects you to a new form where you'll setup your new password (as shown below). Create your new password and select the **Confirm** button.

<p align="center"><img src="./images/lab2/part3/change_password.png" alt="Lab 2 Part 3 Step 10 Reset Password"/></p>

After you've successfully changed your password, you'll be logged into the application and landed at the home page. We won't get into the specifics of the application yet.

**Step 11** - You must have two tenants to finish the lab exercises. If you only have one tenant registered, create another by repeating steps 6-10 again, supplying a different email address for your tenant.

**Step 12** - Now that our tenants have been created through the onboarding flow let's actually create some products via the application. Log into the application as your first tenant and navigate to the **Catalog** menu item at the top of the page.

<p align="center"><img src="./images/lab2/part3/catalog.png" alt="Lab 2 Part 3 Step 12 Catalog Page"/></p>

**Step 13** - With the **Catalog** page open, select the **Add Product** button from the top right of the page. Fill in the details with the product data of your choice. However, for the **SKU**, precede all of your SKU's with **TENANTONE**. So, SKU one might be "**TENANTONE-ABC**". The key here is that we want to have _specific_ values that are prepended to your SKU that clearly identify the products as belonging to this specific tenant.

<p align="center"><img src="./images/lab3/part1/add_product1.png" alt="Lab 3 Part 1 Step 7 Add Product"/></p>

**Step 14** - Once you've added a couple of products for one of your tenants, select the dropdown menu with your tenant name at the top right of the screen and select **Logout**. This will return you to the login page.

<p align="center"><img src="./images/lab2/part3/logout.png" alt="Lab 2 Part 3 Step 14 Logout"/></p>

**Step 15** - Enter the credentials of the other tenant that you created and select the **Login** button. You're now logged in as a different tenant and you should see a different name in the profile menu selection in the upper right of the screen.

**Step 16** - Now navigate to the **Catalog** view again. You should note that the list of products is empty at this point. The products that you previously created were associated with another tenant so they are intentionally not show here. **The illustrates that our partitioning is working**.

**Step 17** - As before, click the **Add Product** button to fill in the details with the product data of your choice. However, for the SKU, precede all of your SKU's with **TENANTTWO**. So, SKU one might be **TENANTTWO-ABC**. The key here is that we want to have _specific_ values that are prepended to your SKU that clearly identify the products as belonging to this specific tenant.

<p align="center"><img src="./images/lab3/part1/add_product2.png" alt="Lab 3 Part 1 Step 13 Add Product"/></p>

**Step 18** - After completing this onboarding process and adding these products for two separate tenants, we can now go see how this data landed in DynamoDB tables that support this experience.

Navigate to the **DynamoDB** service in the AWS console and select **Tables** from the list of options at the top left of the page. Select the **ProductBootcamp** table and then the **Items** tab. Notice that the table is partitioned by `tenant_id`. You should be able to see the products you entered through the web application while logged in as the 2 different tenants (separate from the products you entered via the REST API earlier in the lab).

<p align="center"><img src="./images/lab2/part3/product_table.png" alt="Lab 2 Part 3 Step 18 Product Table"/></p>

**Step 19** - If you review the **TenantBootcamp** table, you should see entries for the tenants you onboarded through the web application and their automatically generated GUIDs in the **tenant_id** field will match the **tenant_id** field in the entries in the **ProductBootcamp** table.

**Recap**: You've now elevated the mechanism of acquiring tenant context in our microservices by extracting our custom "claims" from the security token passed in the Authorization HTTP header. We reduced developer complexity in applying the tenant context by creating a custom TokenManager helper class that takes advantage of the Express framework's "middleware" concept to intercept all incoming requests prior to executing a REST resource's method.

[Continue to Lab 3](Lab3.md)
