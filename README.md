# serverless-single-page-app-plugin

A plugin for [Serverless Framework](https://serverless.com), to simplify deploying Single Page Application using S3 and CloudFront.

From the [official example](https://github.com/serverless/examples/tree/v3/aws-node-single-page-app-via-cloudfront)

## Prerequisite

[Nodejs](https://nodejs.org/en/) (at least version 8)

The `serverless-single-page-app-plugin` in this example requires the Serverless Framework version 1.2.0 or higher and the AWS Command Line Interface. Learn more [here](http://docs.aws.amazon.com/cli/latest/userguide/installing.html) on how to install the AWS Command Line Interface.

## Installation

Install the package via NPM:

```bash
npm install -D https://github.com/kanglecai/serverless-single-page-app-plugin.git
```

Then register it in your `serverless.yml` file, as a plugin:

```
plugins:
  - serverless-single-page-app-plugin
```

And set the `s3Bucket` and `s3LocalPath` custom variables:

```
custom:
  s3Bucket: your-bucket-name123
  s3LocalPath: dist/
```

Finally, add appropriately-named resources (Bucket, BucketPolicy and Distribution) and Outputs:

```
resources:
  Resources:
    ## Specifying the S3 Bucket
    WebAppS3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        BucketName: ${self:custom.s3Bucket}
        OwnershipControls:
          Rules:
            - ObjectOwnership: ObjectWriter
        PublicAccessBlockConfiguration:
          BlockPublicAcls: false
        WebsiteConfiguration:
          IndexDocument: index.html
          ErrorDocument: index.html
    ## Specifying the policies to make sure all files inside the Bucket are avaialble to CloudFront
    WebAppS3BucketPolicy:
      Type: AWS::S3::BucketPolicy
      Properties:
        Bucket:
          Ref: WebAppS3Bucket
        PolicyDocument:
          Statement:
            - Sid: PublicReadGetObject
              Effect: Allow
              Principal: "*"
              Action:
              - s3:GetObject
              Resource: arn:aws:s3:::${self:custom.s3Bucket}/*
    ## Specifying the CloudFront Distribution to server your Web Application
    WebAppCloudFrontDistribution:
      Type: AWS::CloudFront::Distribution
      Properties:
        DistributionConfig:
          Origins:
            - DomainName: ${self:custom.s3Bucket}.s3.amazonaws.com
              ## An identifier for the origin which must be unique within the distribution
              Id: WebApp
              CustomOriginConfig:
                HTTPPort: 80
                HTTPSPort: 443
                OriginProtocolPolicy: https-only
              ## In case you want to restrict the bucket access use S3OriginConfig and remove CustomOriginConfig
              # S3OriginConfig:
              #   OriginAccessIdentity: origin-access-identity/cloudfront/E127EXAMPLE51Z
          Enabled: 'true'
          ## Uncomment the following section in case you are using a custom domain
          # Aliases:
          # - mysite.example.com
          DefaultRootObject: index.html
          ## Since the Single Page App is taking care of the routing we need to make sure ever path is served with index.html
          ## The only exception are files that actually exist e.h. app.js, reset.css
          CustomErrorResponses:
            - ErrorCode: 404
              ResponseCode: 200
              ResponsePagePath: /index.html
          DefaultCacheBehavior:
            AllowedMethods:
              - DELETE
              - GET
              - HEAD
              - OPTIONS
              - PATCH
              - POST
              - PUT
            ## The origin id defined above
            TargetOriginId: WebApp
            ## Defining if and how the QueryString and Cookies are forwarded to the origin which in this case is S3
            ForwardedValues:
              QueryString: 'false'
              Cookies:
                Forward: none
            ## The protocol that users can use to access the files in the origin. To allow HTTP use `allow-all`
            ViewerProtocolPolicy: redirect-to-https
          ## The certificate to use when viewers use HTTPS to request objects.
          ViewerCertificate:
            CloudFrontDefaultCertificate: 'true'
          ## Uncomment the following section in case you want to enable logging for CloudFront requests
          # Logging:
          #   IncludeCookies: 'false'
          #   Bucket: mylogs.s3.amazonaws.com
          #   Prefix: myprefix

  ## In order to print out the hosted domain via `serverless info` we need to define the DomainName output for CloudFormation
  Outputs:
    WebAppS3BucketOutput:
      Value:
        'Ref': WebAppS3Bucket
    WebAppCloudFrontDistributionOutput:
      Value:
        'Fn::GetAtt': [ WebAppCloudFrontDistribution, DomainName ]
```

## Deploy

Warning: Whenever you making changes to CloudFront resource in `serverless.yml` the deployment might take a while e.g 20 minutes.

In order to deploy the Single Page Application you need to setup the infrastructure first by running

```bash
serverless deploy
```

The expected result should be similar to:

```bash
Serverless: Packaging service…
Serverless: Uploading CloudFormation file to S3…
Serverless: Uploading service .zip file to S3…
Serverless: Updating Stack…
Serverless: Checking Stack update progress…
...........................
Serverless: Stack update finished…

Service Information
service: serverless-simple-http-endpoint
stage: dev
region: us-east-1
api keys:
  None
endpoints:
  None
functions:
  None
```

After this step your S3 bucket and CloudFront distribution is setup. Now you need to upload your static file e.g. `index.html` and `app.js` to S3. You can do this by running

```bash
serverless syncToS3
```

The expected result should be similar to

```bash
Serverless: upload: app/index.html to s3://yourBucketName123/index.html
Serverless: upload: app/app.js to s3://yourBucketName123/app.js
Serverless: Successfully synced to the S3 bucket
```

Hint: The plugin is simply running the AWS CLI command: `aws S3 sync app/ s3://yourBucketName123/`

Now you just need to figure out the deployed URL. You can use the AWS Console UI or run

```bash
sls domainInfo
```

The expected result should be similar to

```bash
Serverless: Web App Domain: dyj5gf0t6nqke.cloudfront.net
```

Visit the printed domain domain and navigate on the web site. It should automatically redirect you to HTTPS and visiting <yourURL>/about will not result in an error with the status code 404, but rather serves the `index.html` and renders the about page.

This is how it should look like: ![Screenshot](https://cloud.githubusercontent.com/assets/223045/20391786/287cb3acd5-11e6-9eaf-89f641ed9e14.png)

## Re-deploying

If you make changes to your Single Page Application you might need to invalidate CloudFront's cache to make sure new files are served.
Meaning, run:

```bash
serverless syncToS3
```

To sync your files and then:

```bash
serverless invalidateCloudFrontCache
```

## Further Improvements

Here a list of potential improvements you can do with your CloudFront setup depending on your use-case:

- Setup a custom domain alias
- Logging for CloudFront requests
- Setup a restriction so the Bucket is not publicly accessible except via CloudFront
