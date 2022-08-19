# CDK Workshop

Git

- `git status`
- `git add --all`
- `git commit -m "<insert message>"`
- `git push`
- Ben-Withnell
- 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



#### Terminal Commands

- cdk init
- cdk synth
- cdk bootstrap
- cdk deploy
  - cdk deploy --hotswap
  - cdk watch
  - cdk watch --no-hotswap
- cdk diff
- cdk destroy

### Notes:

- [AWS Construct Library](https://docs.aws.amazon.com/cdk/api/latest/docs/aws-construct-library.html)
  -  (https://docs.aws.amazon.com/cdk/api/latest/docs/aws-construct-library.html)
- 



#### Gitignore:

```
*.js
!jest.config.js
*.d.ts
node_modules

# CDK asset staging directory
.cdk.staging
cdk.out

!lambda/*.js
```

#### Lambda Handler code

###### lambda/hello.js

```js
exports.handler = async function(event) {
  console.log("request:", JSON.stringify(event, undefined, 2));
  return {
    statusCode: 200,
    headers: { "Content-Type": "text/plain" },
    body: `Hello, CDK! You've hit ${event.path}\n`
  };
};
```



#### Add an AWS Lambda Function to the stack

###### lib/cdk-workshop-stack.ts

```ts
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';

export class CdkWorkshopStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // defines an AWS Lambda resource
    const hello = new lambda.Function(this, 'HelloHandler', {
      runtime: lambda.Runtime.NODEJS_14_X,    // execution environment
      code: lambda.Code.fromAsset('lambda'),  // code loaded from "lambda" directory
      handler: 'hello.handler'                // file is "hello", function is "handler"
    });
  }
}
```

- Signature: (scope, id, props)
  - scope: 
    - The current scope in which the construct is created
    - Normally `this` is passed through
  - id:
    - The local identity of the construct
    - An ID that  is unique amongst construct within the same scope
    - CDK uses this identity to calculate the CloudFormation **Logical ID** for each resource defined within the scope
  -  props:
    - always a set of initialization properties
    - These are specific to each construct

### Tests

- apigateway-aws-proxy

  ```json
  {
    "statusCode": 200,
    "headers": {
      "Content-Type": "text/plain"
    },
    "body": "Hello, CDK! You've hit /path/to/resource\n"
  }
  ```

#### CDK Deploy/ Watch

- cdk  deploy --hotswap
- In `cdk.json` you can change what files/ file types are excluded from `cdk watch`

#### Add a [lambdaRestAPI](https://cdkworkshop.com/20-typescript/30-hello-cdk/400-apigw.html) construct to your stack

```js
import * as apigw from 'aws-cdk-lib/aws-apigateway';
```

```js
// defines an API Gateway REST API resource backed by our "hello" function.
    new apigw.LambdaRestApi(this, 'Endpoint', {
      handler: hello
    });
```

- `cdk deploy`
- https://u990oz076d.execute-api.eu-west-2.amazonaws.com/prod/
- `curl https://u990oz076d.execute-api.eu-west-2.amazonaws.com/prod/`
  - </>  Good Night Ben, CDK! You've hit /

## Writing Constructs

#### Defining the HitCounter API

##### Create a new file for the hit counter construct

###### lib/hitcounter.ts

```typescript
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import { Construct } from 'constructs';

export interface HitCounterProps {
  /** the function for which we want to count url hits **/
  downstream: lambda.IFunction;
}

export class HitCounter extends Construct {
  constructor(scope: Construct, id: string, props: HitCounterProps) {
    super(scope, id);

    // TODO
  }
}
```

- Declared new construct called `HitCounter`
- `Scope`, `id` and `props` are used in the constructor argument, and we propagate them to the `cdk.Construct` base class.
- `props` argument is of type `HitCounterProps` which includes a single property `downstream` of type `lambda.IFunction`.
  - This is where we are going to "plug in" the lambda function we created in the previous chapter so it can be hit-counted.

###### lambda/hitcounter.js

```js
const { DynamoDB, Lambda } = require('aws-sdk');

exports.handler = async function(event) {
  console.log("request:", JSON.stringify(event, undefined, 2));

  // create AWS SDK clients
  const dynamo = new DynamoDB();
  const lambda = new Lambda();

  // update dynamo entry for "path" with hits++
  await dynamo.updateItem({
    TableName: process.env.HITS_TABLE_NAME,
    Key: { path: { S: event.path } },
    UpdateExpression: 'ADD hits :incr',
    ExpressionAttributeValues: { ':incr': { N: '1' } }
  }).promise();

  // call downstream function and capture response
  const resp = await lambda.invoke({
    FunctionName: process.env.DOWNSTREAM_FUNCTION_NAME,
    Payload: JSON.stringify(event)
  }).promise();

  console.log('downstream response:', JSON.stringify(resp, undefined, 2));

  // return response back to upstream caller
  return JSON.parse(resp.Payload);
};
```

###### lib/hitcounter.ts

```typescript
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import { Construct } from 'constructs';

export interface HitCounterProps {
  /** the function for which we want to count url hits **/
  downstream: lambda.IFunction;
}

export class HitCounter extends Construct {

  /** allows accessing the counter function */
  public readonly handler: lambda.Function;

  constructor(scope: Construct, id: string, props: HitCounterProps) {
    super(scope, id);

    const table = new dynamodb.Table(this, 'Hits', {
        partitionKey: { name: 'path', type: dynamodb.AttributeType.STRING }
    });

    this.handler = new lambda.Function(this, 'HitCounterHandler', {
        runtime: lambda.Runtime.NODEJS_14_X,
        handler: 'hitcounter.handler',
        code: lambda.Code.fromAsset('lambda'),
        environment: {
            DOWNSTREAM_FUNCTION_NAME: props.downstream.functionName,
            HITS_TABLE_NAME: table.tableName
        }
    });
    table.grantReadWriteData(this.handler);
  }
}
```

- Defined a DynamoDB table with `path` as the partition key

- Defined a Lambda function which is bound to the `lambda/hitcounter.handler` code

- Wired the Lambda environment variables to the `functionName` and the `tableName` of out resources

  ###### cdk-workshop-stack.ts

```typescript
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigw from 'aws-cdk-lib/aws-apigateway';
import { HitCounter } from './hitcounter';

export class CdkWorkshopStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const hello = new lambda.Function(this, 'HelloHandler', {
      runtime: lambda.Runtime.NODEJS_14_X,
      code: lambda.Code.fromAsset('lambda'),
      handler: 'hello.handler'
    });

    const helloWithCounter = new HitCounter(this, 'HelloHitCounter', {
      downstream: hello
    });

    // defines an API Gateway REST API resource backed by our "hello" function.
    new apigw.LambdaRestApi(this, 'Endpoint', {
      handler: helloWithCounter.handler
    });
  }
}
```

- Changed the API Gateway (apigw) to `helloWithCounter.handler`
  - Means that when the endpoint is hit, the API Gateway will route the request to the hit counter handler.
  - It then logs the hit and relays it over to the `hello` function.
  - The responses will be relayed back in the reverse order all the way to the user
- Curl <address>

### Using Construct Libraries

- [cdk-dynamo-table-viewer](https://www.npmjs.com/package/cdk-dynamo-table-viewer)

>  `npm install cdk-dynamo-table-viewer@0.2.0`

###### cdk-workshop-stack.ts

```typescript
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigw from 'aws-cdk-lib/aws-apigateway';
import { HitCounter } from './hitcounter';
import { TableViewer } from 'cdk-dynamo-table-viewer';

export class CdkWorkshopStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const hello = new lambda.Function(this, 'HelloHandler', {
      runtime: lambda.Runtime.NODEJS_14_X,
      code: lambda.Code.fromAsset('lambda'),
      handler: 'hello.handler'
    });

    const helloWithCounter = new HitCounter(this, 'HelloHitCounter', {
      downstream: hello
    });

    // defines an API Gateway REST API resource backed by our "hello" function.
    new apigw.LambdaRestApi(this, 'Endpoint', {
      handler: helloWithCounter.handler
    });
    new TableViewer(this, 'ViewHitCounter', {
      title: 'Hello Hits',
      table: helloWithCounter.table
    });
  }
}
```

###### lambda/hitcounter.js

```javascript
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import { Construct } from 'constructs';

export interface HitCounterProps {
  /** the function for which we want to count url hits **/
  downstream: lambda.IFunction;
}

export class HitCounter extends Construct {

  /** allows accessing the counter function */
  public readonly handler: lambda.Function;
  /** the hit counter table */
  public readonly table: dynamodb.Table;

  constructor(scope: Construct, id: string, props: HitCounterProps) {
    super(scope, id);

    const table = new dynamodb.Table(this, 'Hits', {
        partitionKey: { name: 'path', type: dynamodb.AttributeType.STRING }
    });
    this.table = table;

    this.handler = new lambda.Function(this, 'HitCounterHandler', {
        runtime: lambda.Runtime.NODEJS_14_X,
        handler: 'hitcounter.handler',
        code: lambda.Code.fromAsset('lambda'),
        environment: {
            DOWNSTREAM_FUNCTION_NAME: props.downstream.functionName,
            HITS_TABLE_NAME: table.tableName
        }
    });
    table.grantReadWriteData(this.handler);
  }
}
```



### Testing Constructs

##### CDK assert Library

- `assertions` (`aws-cdk-lib/assertions`)

- `hasResourceProperties`

  ```javascript
  template.hasResourceProperties('AWS::CertificateManager::Certificate', {
      DomainName: 'test.example.com',
  
      ShouldNotExist: Match.absent(),
      // Note: some properties omitted here
  });
  ```

  `test-cdk-workshop.test.ts`

  ```typescript
  import { Template, Capture } from 'aws-cdk-lib/assertions';
  import * as cdk from 'aws-cdk-lib';
  import * as lambda from 'aws-cdk-lib/aws-lambda';
  import { HitCounter }  from '../lib/hitcounter';
  
  test('DynamoDB Table Created', () => {
    const stack = new cdk.Stack();
    // WHEN
    new HitCounter(stack, 'MyTestConstruct', {
      downstream:  new lambda.Function(stack, 'TestFunction', {
        runtime: lambda.Runtime.NODEJS_14_X,
        handler: 'hello.handler',
        code: lambda.Code.fromAsset('lambda')
      })
    });
    // THEN
  
    const template = Template.fromStack(stack);
    template.resourceCountIs("AWS::DynamoDB::Table", 1);
  });	
  ```

> npm run test















