+++
author = "Daniel Ancuta"
title = "AWS Amplify Override Amplify-generated resources. On example with resolvers."
date = "2024-01-13"
description = "AWS Amplify Override Amplify-generated resources. On example with resolvers."
tags = ["aws", "amplify", "aws appsync", "appsync resolver", "override amplify", "TypeScript"]
+++

[AWS Amplify](https://aws.amazon.com/amplify/) is a great tool when you're building
web and mobile apps, and you want to integrate them easily with the backend.

In few simple steps you can have working GraphQL endpoint, storage on backend, authentication and authorization through Cognito.

But it also has some limitations, either small bugs or just design decisions:
* [AWS Amplify Is A Grift](https://samthor.au/2023/aws-amplify-is-a-grift/) that explains problem of using SCAN instead of QUERY by default by resolvers
* [Use CDK for cross-account or cross-region Amplify backend deployments](https://docs.amplify.aws/javascript/tools/cli/usage/export-to-cdk/#use-cdk-for-cross-account-or-cross-region-amplify-backend-deployments)
* [Add support for overrides on amplify export #10235](https://github.com/aws-amplify/amplify-cli/issues/10235), amplify-cli issue with `amplify export`

Luckily all listed above can be mitigated by developers using Amplify CLI.

## Override Amplify-generated resources
Today I want to focus on "Overriding Amplify-generated resources", when you use CDK for your cross-account or cross-region amplify backend deployments.

As an example project we will modify auto-generated AppSync resolvers. You can use this pattern to override other resources which you can't simply override because of [amplify export](https://github.com/aws-amplify/amplify-cli/issues/10235) limitations.

But first things first.

## Configure amplify project to use CDK
For this exercise I've just created new amplify project, called `amplifytest`. So all examples below uses that name,
you should replace it with name of your project.

### Initialization of CDK
* `cd my_project`
* `npm i aws-cdk aws-cdk-lib`
* `mkdir amplifytest && cd amplifytest` (we need empty directory to initialize cdk)
* `./node_modules/.bin/cdk init app --language=typescript`
* `cd ..`
* `cp -R ./amplifytest/bin ./bin`
* `cp -R ./amplifytest/lib ./lib`
* `cp ./amplifytest/cdk.json ./cdk.json`

You need one more thing, install [@aws-amplify/cdk-exported-backend](https://www.npmjs.com/package/@aws-amplify/cdk-exported-backend).
This library has few dependencies, so most likely you will also need to install `@types/node` and `lodash`

At this point you should have in your amplify project three extra files:

#### ./cdk.json
```Json
{
  "app": "npx ts-node --prefer-ts-exts bin/amplifytest.ts",
  ...
}
```

#### ./bin/amplifytest.ts
```Typescript
#!/usr/bin/env node


import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { AmplifytestStack } from '../lib/amplifytest-stack';

const app = new cdk.App();
new AmplifytestStack(app, 'AmplifytestStack', {
});
```

#### ./lib/amplifytest-stack.ts
That's the one we will modify in order to set up [@aws-amplify/cdk-exported-backend](https://www.npmjs.com/package/@aws-amplify/cdk-exported-backend)

```Typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as path from 'path';
import { AmplifyExportedBackend }  from '@aws-amplify/cdk-exported-backend';

export class AmplifytestStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
        super(scope, id, props);
        
        const basePath = path.resolve(process.cwd());
        const basePathExport = path.resolve(basePath, 'export-amplify-stack/amplify-export-amplifytest');

        new AmplifyExportedBackend(this, 'AmplifyExportedBackend', {
          path: basePathExport,
          amplifyEnvironment: 'dev',
        });
  }
}
```

{{< alert "circle-info" >}}
**Worth reading**
{{< /alert >}}
1. [Use an exported Amplify backend in AWS Cloud Development Kit (CDK)](https://docs.amplify.aws/javascript/tools/cli/usage/export-to-cdk/#use-an-exported-amplify-backend-in-aws-cloud-development-kit-cdk)
2. [@aws-amplify/cdk-exported-backend](https://www.npmjs.com/package/@aws-amplify/cdk-exported-backend)

## Problem we want to solve: return custom validation error
Let's imagine you have your mutation, that has custom data source (lambda) connected:
```GraphQL
type Mutation {
  customMutation(id: String): Boolean @function(name: "my-test-function")
}
```

And your function `my-test-function` does some request payload validation. The only way to return back to client validation errors is by raising an exception:
```Python
raise Exception("My error")
```

Which then is being transformed by AppSync to this response:
```graphql
{
  "data": {
    "myMutation": null
  },
  "errors": [
    {
      "path": [
        "myMutation"
      ],
      "data": null,
      "errorType": "Lambda:Unhandled",
      "errorInfo": null,
      "locations": [
        {
          "line": 24,
          "column": 2,
          "sourceName": null
        }
      ],
      "message": "My error"
    }
  ]
}
```

But what if you want to be more flexible? You would want to modify other attributes like `errorType`, `errorInfo` or you just want to return back more than one validation error at a time?

To be able to achieve that, you would have to modify `*LambdaDataSource.res.vtl`, in our case `InvokeMyTestFunctionLambdaDataSource.res.vtl`, resolver template.

Auto generated resolvers are located in `amplify/backend/api/amplifytest/build/resolvers`, and you have to override them in `amplify/backend/api/amplifytest/resolvers`.

`InvokeMyTestFunctionLambdaDataSource.res.vtl` resolver, from:
```Apache
## [Start] Handle error or return result. **
#if( $ctx.error )
  $util.error($ctx.error.message, $ctx.error.type)
#end
$util.toJson($ctx.result)
## [End] Handle error or return result. **
```

would have to be overridden by:
```Apache
## [Start] Handle error or return result. **
#if( $ctx.result && $ctx.result.errorMessage )
    $util.error($ctx.result.errorMessage, $ctx.result.errorType, $ctx.result.data, $ctx.result.errorInfo)
#elseif( $ctx.error )
    $util.error($ctx.error.message, $ctx.error.type)
#else
    $utils.toJson($ctx.result)
#end
## [End] Handle error or return result. **
```

After that is done your lambda can return something like that:
```Python
return {
    "data": [],
    "errorType": "MyCustomErrorType",
    "errorMessage": 'Error message',
    "errorInfo": {"key": "value"},
}
```

As you can see it's rather easy process. That can be done manually.

But doing it manually has few disadvantages:
- Someone would have to remember to do it every time, new custom data source is introduced
- Change of resolver template requires changes in multiple files
- When your list of custom data sources is short you can get away with manual updates, but what if it grows to 10, 20, 30 and more?

It's not ideal situation. More robust way would be to do it automatically.

### Create template file
We will create "template" file, which then will be used for all of our data source resolvers.

```Apache
## amplify/backend/api/amplifytest/base_resolvers/_BaseLambdaDataSource.res.vtl
## [Start] Handle error or return result. **
#if( $ctx.result && $ctx.result.errorMessage )
    $util.error($ctx.result.errorMessage, $ctx.result.errorType, $ctx.result.data, $ctx.result.errorInfo)
#elseif( $ctx.error )
    $util.error($ctx.error.message, $ctx.error.type)
#else
    $utils.toJson($ctx.result)
#end
## [End] Handle error or return result. **
```

### Modify ./lib/amplifytest-stack.ts
Here we will add code that will be executed every time you deploy your stack with cdk.

```Typescript
import * as cdk from 'aws-cdk-lib';
import {Construct} from 'constructs';
import * as path from 'path';

const fs = require('fs');
import {AmplifyExportedBackend} from '@aws-amplify/cdk-exported-backend';

export class AmplifytestStack extends cdk.Stack {
    constructor(scope: Construct, id: string, props?: cdk.StackProps) {
        super(scope, id, props);

        const basePath = path.resolve(process.cwd());
        const basePathExport = path.resolve(basePath, 'export-amplify-stack/amplify-export-amplifytest');
        const functionDirectiveStack = require(
            path.resolve(basePathExport,
                "api", "amplifytest", "amplify-appsync-files", "stacks", "FunctionDirectiveStack.json")
        );
        const resolverTemplate = fs.readFileSync(
            path.resolve(basePath, "amplify", "backend", "api", "amplifytest", "base_resolvers", "_BaseLambdaDataSource.res.vtl")
        );

        Object
            .entries(functionDirectiveStack.Resources)
            .filter(([_, value]: [string, any]) => value.Type === "AWS::AppSync::FunctionConfiguration")
            .forEach(([_, value]: [string, any], index) => {
                let responseMapping = value.Properties["ResponseMappingTemplateS3Location"]["Fn::Join"][1][4];

                fs.writeFileSync(
                    path.resolve(
                        basePathExport, "api", "amplifytest", "amplify-appsync-files", "resolvers",
                        path.parse(responseMapping).base
                    ),
                    resolverTemplate
                );
            });

        new AmplifyExportedBackend(this, 'AmplifyExportedBackend', {
            path: basePathExport,
            amplifyEnvironment: 'dev',
        });
    }
}
```

In those few lines of code, we loop through all custom data source functions, and create modified version of resolver.
Which is more flexible than the default one.

Whole process happens during `cdk deploy` execution.

### Test a solution
To test it we need to deploy our stack

1. `./node_modules/.bin/amplify export`
2. `npx cdk deploy AmplifytestStack/AmplifyExportedBackend-amplify-backend-stack`

## Final Thoughts
Even though I have started with listing some limitations of amplify, I hope you haven't got scared. 

AWS Amplify is a good tool, but as every tool it has its own problems. Still I would recommend using it for your production applications!

## The End
That's it! If your company needs some help with AWS, [get in touch]({{< ref "/contact" >}} "Get in touch").
