### Creating S3 Bucket not for public

1. Enter Bucket name.
2. Select Region (eu-london).
3. Add tags. Owner (emailaddress), Environment (Dev, Test etc), CostCenter (01-429) JobNumber(268030-00).
4. Keep all permission boxes selected.
5. Upload files to the bucket by selecting add file or click-drag-drop files into container.
6. Select properties then enable static website hosting (use this bucket to host a website). 
7. Link index & error html file.
8. Select permissions and add bucket policy object. Change Resource endpoint <kopaki-jc> to the bucket name

```javascript 
{
    "Version": "2012-10-17",
    "Id": "S3PolicyId1",
    "Statement": [
        {
            "Sid": "IPAllow",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::kopaki-jc/*",
            "Condition": {
                "IpAddress": {
                    "aws:SourceIp": [
                        "192.168.0.0/16",
                        "193.17.187.245/24",
                        "10.0.0.0/8",
                        "108.171.128.167/28",
                        "172.16.0.0/12"
                    ]
                }
            }
        }
    ]
}
```

### Linking S3 to CloudFront

1. Step 1. Delivery method select Web Get Started.
2. Enter Origin Domain Name, (S3 bucket name you created shown in the dropdown).
3. Yes, Restrict Bucket Access. Create a new Identity. Yes, Update Bucket Policy.
4. Change Viewer Protocol Policy (Redirect HTTP to HTTPS).
5. Include Lambda Auth function (Viewer Request) if you already have one, if not see the next section that will guide you how to setup a lambda function.
5. Input Default Root Object file `example index.html`
6. Create Distribution
7. Progress status shows when its finished


## Linking Auth Lambda function (need IAM:CreateRole permissions)

1. Create function from scratch and name it.
2. Select runtime, default is fine.
3. Copy and paste the function below, change authUser & authPass.
4. When done link this to an already Existing role. `example lnerailway-lambda`. Copy the ARN at the top `example arn:aws:lambda:us-east-1:986634511047:function:kopaki-simple-auth`
5. Return to CloudFront and scroll down and select (Viewer Request)
6. Enter ARN and add :2 at the end `example arn:aws:lambda:us-east-1:986634511047:function:kopaki-simple-auth:2`
7. Save

```javascript
'use strict';
exports.handler = (event, context, callback) => {

    // Get request and request headers
    const request = event.Records[0].cf.request;
    const headers = request.headers;

    // Configure authentication
    const authUser = 'enterUsername';
    const authPass = 'enterPassword';

    // Construct the Basic Auth string
    const authString = 'Basic ' + new Buffer(authUser + ':' + authPass).toString('base64');

    // Require Basic authentication
    if (typeof headers.authorization == 'undefined' || headers.authorization[0].value != authString) {
        const body = 'Unauthorized';
        const response = {
            status: '401',
            statusDescription: 'Unauthorized',
            body: body,
            headers: {
                'www-authenticate': [{key: 'WWW-Authenticate', value:'Basic'}]
            },
        };
        callback(null, response);
    }

    // Continue request processing if authentication passed
    callback(null, request);
};
```