This serverless plugin is a wrapper for [amplify-appsync-simulator](amplify-appsync-simulator) made for testing AppSync APIs built with [serverless-appsync-plugin](https://github.com/sid88in/serverless-appsync-plugin).


# Requires
- [serverless framework](https://github.com/serverless/serverless)
- [serverless-appsync-plugin](https://github.com/sid88in/serverless-appsync-plugin)

# Install

````bash
npm install serverless-appsync-simulator
# or
yarn add serverless-appsync-simulator
````

# Usage

## CLI

````bash
sls appsync-offline
AppSync Simulator: AppSync endpoint: http://localhost:20002/graphql
AppSync Simulator: GraphiQl: http://localhost:20002
````

(*)

### Options

| Option | Description |
|--------| ------------|
| --port / -p | The port of the operations server |
| --wsPort / -w | The port of the subscriptions server |
| --apiKey / -k | The api key to use for authentication |
| --location / -l | The location of your Lambda handlers |

## With `serverless-offline`

This plugin also works with the [serverless-offline](https://github.com/dherault/serverless-offline)

````yml
plugins:
  - serverless-dynamodb-local # only if you need dynamodb resolvers and you don't have an external dynamodb
  - serverless-appsync-simulator
  - serverless-offline
````

**Note:** Order is important `serverless-appsync-simulator` must go **before** `serverless-offline`

To start the simulator, run the following command:
````bash
sls offline start
...
Serverless: AppSync endpoint: http://localhost:20002/graphql
Serverless: GraphiQl: http://localhost:20002
...
````

# Yml Configuration

Put options under `custom.appsync-simulator` in your `serverless.yml` file

| option                   | default               | description                                                                                                                    |
| ------------------------ | --------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| apiKey                   | `0123456789`          | When using `API_KEY` as authentication type, the key to authenticate to the endpoint.                                          |
| port                     | 20002                 | AppSync operations port                                                                                                        |
| wsPort                   | 20003                 | AppSync subscriptions port                                                                                                     |
| location                 | . (base directory)    | Location of the lambda functions handlers.                                                                                     |
| refMap | {}        | A mapping of [resource resolutions](#resource-cloudformation-functions-resolution) for the `Ref` function |
| getAttMap | {}        | A mapping of [resource resolutions](#resource-cloudformation-functions-resolution) for the `GetAtt` function |
| dynamoDb.endpoint        | http://localhost:8000 | Dynamodb endpoint. Specify it if you're not using serverless-dynamodb-local. Otherwise, port is taken from dynamodb-local conf |
| dynamoDb.region          | localhost             | Dynamodb region. Specify it if you're connecting to a remote Dynamodb intance.                                                 |
| dynamoDb.accessKeyId     | DEFAULT_ACCESS_KEY    | AWS Access Key ID to access DynamoDB                                                                                           |
| dynamoDb.secretAccessKey | DEFAULT_SECRET        | AWS Secret Key to access DynamoDB |

Example:

````yml
custom:
  appsync-simulator:
    location: '.webpack/service' # use webpack build directory
    dynamoDb:
      endpoint: 'http://my-custom-dynamo:8000'

````

# Resource CloudFormation functions resolution

This plugin supports *some* resources resolution from the `Ref` and `Fn::GetAtt` functions
in your yaml file. It also supports *some* other Cfn functions such as `Fn::Join`, `Fb::Sub`, etc.

**Note:** Under the hood, this features relies on the [cfn-resolver-lib](https://github.com/robessog/cfn-resolver-lib) package. For more info on supported cfn functions, refer to [the documentation](https://github.com/robessog/cfn-resolver-lib/blob/master/README.md)

## Basic usage

You can reference resources in your functions' environment variables (that will be accessible from your lambda functions) or datasource definitions.
The plugin will automatically resolve them for you.

````yaml
provider:
  environment:
    BUCKET_NAME:
      Ref: MyBucket # resolves to `my-bucket-name`

resources:
  Resources:
    MyDbTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: myTable
      ...
    MyBucket:
      Type: AWS::S3::Bucket
      Properties:
        BucketName: my-bucket-name
    ...

# in your appsync config
dataSources:
  - type: AMAZON_DYNAMODB
    name: dynamosource
    config:
      tableName:
        Ref: MyDbTable # resolves to `myTable`
````

## Override (or mock) values

Sometimes, some references **cannot** be resolved, as they come from an *Output* from Cloudformation; or you might want to use mocked values in your local environment.

In those cases, you can define (or override) those values using the `refMap` and `getAttMap` options.

- `refMap` takes a mapping of *resource name* to *value* pairs
- `getAttMap` takes a mapping of *resource name* to *attribute/values* pairs

Example:

````yaml
custom:
  serverless-appsync-simulator:
    refMap:
      # Override `MyDbTable` resolution from the previous example.
      MyDbTable: 'mock-myTable'
    getAttMap:
      # define ElasticSearchInstance DomainName
      ElasticSearchInstance:
        DomainEndpoint: "localhost:9200"

# in your appsync config
dataSources:
  - type: AMAZON_ELASTICSEARCH
    name: elasticsource
    config:
      # endpoint resolves as 'http://localhost:9200'
      endpoint:
        Fn::Join:
          - ""
          - - https://
            - Fn::GetAtt:
                - ElasticSearchInstance
                - DomainEndpoint
````

## Limitations

This plugin only tries to resolve the following parts of the yml tree:
- `provider.environment`
- `functions[*].environment`
- `custom.appSync`

If you have the need of resolving others, feel free to open an issue and explain your use case.

For now, the supported resources to be automatically resovled by `Ref:` are:
- DynamoDb tables
- S3 Buckets

Feel free to open a PR or an issue to extend them as well.

# Supported Resolver types

This plugin supports resolvers implemented by `amplify-appsync-simulator`, as well as custom resolvers.

**From Aws Amplify:**
- NONE
- AWS_LAMBDA (*)
- AMAZON_DYNAMODB
- PIPELINE

**Implemented by this plugin**
- AWS_LAMBDA (*)
- AMAZON_ELASTIC_SEARCH
- HTTP

(*) The `AWS_LAMBDA` dataloader has been partially copied from Aws Amplify but has been extended
to support the *BatchInvoke* operations

**Not Supported / TODO**
- RELATIONAL_DATABASE
