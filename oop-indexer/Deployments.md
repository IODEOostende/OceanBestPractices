# Deploying the OBP Insfrastructure

### Overview:

- Storage Resources (Creating S3 Buckets)
- Text Extractor
    - Text Extractor Permissions
- Elasticsearch 
    - Elasticsearch Cluster
    - Indices
- Virtuoso Instance (Please see `Virtuoso.md`)
- Ingest and Indexer (Recommended to deploy in the following order)
    - Scheduler
    - Downloader
    - Indexer
- Deploying the Static Site
    - Website

## Storage Resources

This stack creates S3 buckets we'll need in order to store Lambda functions later on.

`aws cloudformation deploy --template-file function-storage.yml --stack-name obp-function-storage-{ENVIRONMENT} --parameter-overrides Environment={ENVIRONMENT} --profile {AWS_PROFILE}`

## Text Extractor Permissions

`aws cloudformation deploy --template-file text-extractor-permissions.yml --stack-name obp-text-extractor-permissions-{ENVIRONMENT} --parameter-overrides Environment={ENVIRONMENT}  --capabilities CAPABILITY_NAMED_IAM --profile {AWS_PROFILE}`

## Text Extractor
Please refer to the [README](./README.md), [TextExtractor repo](https://github.com/Element84/lambda-text-extractor) and [apex](https://apex.run/) for installation of textextractor.

## Elasticsearch

### Elasticsearch Cluster

The Elasticsearch stack deploys an instance of Elasticsearch we'll use later on. This stack can take awhile to spin up so I recommend starting this one first. You can keep most of the default template parameters if you're just creating an instance suitable for the OBP infrastructure.

`aws cloudformation deploy --template-file elasticsearch.yml --stack-name obp-elasticsearch-{ENVIRONMENT} --parameter-overrides Environment={ENVIRONMENT} --capabilities CAPABILITY_NAMED_IAM --profile {AWS_PROFILE}`

## Virtuoso Instance

Refer to the instructions in [Virtuoso.md](./Virtuoso.md)

### Indexes

Please refer to the instructions [here](./README.md) for creating the document and terms indices. Specifically, the *Creating and Populating the Percolator Index* and *Creating the Document Index* sections.

## Ingest and Indexer

The ingest and indexer pipeline is made up of a few different stacks, each with a specific responsibility. It's best to create these stacks in order as some of them depend on inputs from the others.

### Scheduler

The scheduler is responsible for polling the DSpace RSS feed to discover new documents that need ingesting. It creates a Lambda function and SNS Topic where new document handles are published.

Since updating to Node.js 12.x dependencies must be included with the Lambda package. This function requires the `xml2js` library and must be installed prior to deploying. We use npm to manage dependencies. In order to install dependencies make sure you first have Node and npm installed. Most likely you'll need multiple versions of node on your machine so we recommend installing [nvm](https://github.com/nvm-sh/nvm). Once you have Node and npm installed navigate to the oop-indexer/ingest/scheduler directory and run:

`npm install`

This command will use the `package.json` file to install necessary dependencies into a `node_modules` directory. 

Then use the `aws cloudformation` CLI to package and deploy the stack:

```
> aws cloudformation package --template-file scheduler.yml --output-template-file scheduler-out.yml --s3-bucket obp-indexer-functions-{ENVIORNMENT} --profile {AWS_PROFILE}

> aws cloudformation deploy --template-file scheduler-out.yml --stack-name obp-scheduler-{ENVIRONMENT} --parameter-overrides Environment={ENVIORNMENT} --capabilities CAPABILITY_NAMED_IAM --profile {AWS_PROFILE}
```

### Downloader

The downloader is responsible for downloading document metadata and source binaries. It handles launching the text extraction process once the document source has been saved to S3.

```
> aws cloudformation package --template-file downloader.yml --output-template-file downloader-out.yml --s3-bucket obp-indexer-functions-{ENVIRONMENT} --profile {AWS_PROFILE}`

> aws cloudformation deploy --template-file downloader-out.yml --stack-name obp-downloader-{ENVIRONMENT} --parameter-overrides Environment={ENVIORNMENT} AvailableDocumentTopicArn={AVAILABLE_DOCUMENT_ARN} IndexerFunctionName={INDEXER_FUNCTION_NAME} TextExtractorBucketName={TEXT_EXTRACTOR_BUCKET_NAME} TextExtractorFunctionName={TEXT_EXTRACTOR_FUNCTION} TextExtractorTempBucketName=obp-doc-extracted-temp --capabilities CAPABILITIY_NAMED_IAM --profile {AWS_PROFILE}
```

### Indexer

The indexer is launched once a document has finished the text extraction process. It fetches document metadata and raw text from S3; runs the document through the tagging routine, and inserts a new document into Elasticsearch.

```
> aws cloudformation package --template-file indexer.yml --output-template-file indexer-out.yml --s3-bucket obp-indexer-functions-{ENVIRONMENT} --profile {AWS_PROFILE}

> aws cloudformation deploy --template-file indexer-out.yml --stack-name obp-indexer-{ENVIRONMENT} --parameter-overrides Environment=iode ElasticSearchHost={ElasticSearchHost} DocumentMetadataBucketArn={DocumentMetadataBucketArn} DocumentMetadataBucketName={DocumentMetadataBucketName} DocumentExtractedBucketArn={DocumentExtractedBucketArn} DocumentExtractedBucketName={DocumentExtractedBucketName} TextExtractorStatusTopicArn={TextExtractorStatusTopicArn} --capabilities CAPABILITY_NAMED_IAM --profile {AWS_PROFILE}
```

## Search API

```
> aws cloudformation package --template-file search.yml --output-template-file search-out.yml --s3-bucket obp-search-api-functions-minsem --profile {PROFILE}
 
> aws cloudformation deploy --template-file search-out.yml --stack-name obp-search-api-{ENVIRONMENT} --parameter-overrides Environment={ENVIRONMENT} ElasticSearchHost={ELASTIC_SEARCH_HOST} OntologyStoreHost={VIRTUOSO_HOST} OntologyStorePort={VIRTUOSO_PORT} --capabilities CAPABILITY_NAMED_IAM --profile {PROFILE}
```

## Static Site

### Website

main-site.yml can be found in **oop-web/cloudformation/**

```
> aws cloudformation deploy --template-file main-site.yml --stack-name obp-website-{ENVIORNMENT} --parameter-overrides Environment={ENVIRONMENT} WebsiteDomain={ENVIRONMENT}.oceanbestpractices.org --profile {AWS_PROFILE}
```

### CloudFront Distribution

The CloudFront Distribution serves the web app at edge locations and provides SSL support (if required). Use the following commands to deploy the CloudFront Distrubtion:

```
> aws cloudformation deploy --template-file ../oop-web/cloudformation/cloudfront-distribution.yml --stack-name obp-website-cloudfront-{ENVIRONMENT} --parameter-overrides Environment={ENVIRONMENT}  MainWebsiteS3WebsiteURL={WEBSITE_BUCKET_URL} --profile {AWS_PROFILE}
```