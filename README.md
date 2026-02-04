# Insights Ingress

Ingress is designed to receive payloads from clients and distribute them via a
Kafka message queue to other platform services.

## On-Prem Fork

This fork is maintained for **on-premises Insights deployments** (External Data Pipeline - EDP).

**Repository**: https://github.com/lenasolarova/insights-ingress-go
**Branch**: `new_image`
**Container Image**: https://quay.io/repository/rh-ee-lsolarov/insights-ingress?tab=tags&tag=edp-onprem

### Key Modifications for On-Prem

This fork includes the following changes to support standalone on-prem deployments without 3Scale authentication:

1. **Authentication disabled by default** - `INGRESS_AUTH` defaults to `false` instead of `true`
2. **Standard test identity injection** - When auth is disabled and no `x-rh-identity` header is present, automatically adds the standard test identity:
   - `eyJpZGVudGl0eSI6IHsidHlwZSI6ICJVc2VyIiwgImFjY291bnRfbnVtYmVyIjogIjAwMDAwMDEiLCAib3JnX2lkIjogIjAwMDAwMSIsICJpbnRlcm5hbCI6IHsib3JnX2lkIjogIjAwMDAwMSJ9fX0=`
   - Account: `0000001`, OrgID: `000001`
3. **Kafka broker configuration fix** - Added explicit binding for `INGRESS_KAFKA_BROKERS` environment variable (fixes viper prefix issue)
4. **Comma-separated broker support** - Parses comma-separated Kafka broker list from environment variable

### Building the On-Prem Image

```bash
# Build for AMD64 (required for OpenShift on x86_64)
docker build --platform linux/amd64 -t quay.io/rh-ee-lsolarov/insights-ingress:edp-onprem .

# Push to registry
docker push quay.io/rh-ee-lsolarov/insights-ingress:edp-onprem
```

### Upstream Sync

To sync with upstream changes:

```bash
# Add upstream remote (first time only)
git remote add upstream https://github.com/RedHatInsights/insights-ingress-go.git

# Fetch and merge upstream changes
git fetch upstream
git merge upstream/master
```

## How It Works (On-Prem)

For on-prem deployments, the ingress workflow is simplified:

1. **insights-operator** sends a payload with content type `application/vnd.redhat.<service>.filename+tgz`
2. **Ingress** receives the upload, uploads to S3-compatible storage (MinIO), and publishes to Kafka
3. **Processing services** consume from Kafka and process the data

No 3Scale gateway or authentication is required in this configuration.

### Announcement Topic

Ingress produces a message on the `platform.upload.announce` topic to signal services
that an upload has arrived. The announce message contains a header that specifies which
content-type the message contains. For example:

    - {"service": "advisor"}

These kafka headers allow the consumer to filter out the messages that do not belong
to them without having to extract the full JSON of the incoming message. No performance
impact has been observed relating to this method of filtering.

### Content Type

Uploads coming into Ingress should have the following content type:

`application/vnd.redhat.<service-name>.filename+tgz`

The filename and file type may vary. The portion to note is the service name as
this is where Ingress discovers the proper validating service and what header value
to apply to the `service` header.

Example:

  `application/vnd.redhat.advisor.example+tgz` => `{"service": "advisor"}`

### Message Formats

All messages placed on the Kafka topic will contain JSON with the details for the 
upload. They will contain the following structure:

Validation Messages:

       {
           "account": <account number>,
           "org_id": <org id>,
           "category": <currently translates to filename>,
           "content_type": <full content type string from the client>,
           "request_id": <uuid for the payload>,
           "principal": <currently the org ID>,
           "service": <service the upload goes to>,
           "size": <filesize in bytes>,
           "url": <URL to download the file>,
           "id": <host based inventory id if available>,
           "b64_identity": <the base64 encoded identity of the sender>,
           "timestamp": <the time the upload was received>,
           "metadata": <will contain additional json related to the uploading host>
       }

Any apps that will perform the validation should send **all** of the data they
received in addition to a `validation` key that contains `success` or `failure`
depending on whether the payload passed validation. This data should be sent to 
the `platform.upload.validation` topic.

The `platform.upload.validation` topic is consumed and handled by [Storage Broker](https://www.github.com/redhatinsights/insights-storage-broker).

If the app needs to relay a failure back to the customer via the notification
service, they can do so by supplying additional data as noted below:

Expected Validation Message:
    
    {
        ...all data received by validating app
        "validation": <"success"/"failure">
        ## additional notification data below ##
        "reason": "some error message",
        "system_id": <if available>,
        "hostname": <if available>,
        "reporter": "name of reporting app",
    }

## Errors

Ingress will report HTTP errors back to the client if something goes wrong with the
initial upload. It will be the responsibility of the client to communicate that
connection problem back to the user via a log message or some other means.

The connection from the client to Ingress is closed as soon as the upload finishes.
Errors regarding anything beyond that point (cloud storage uploads, message queue errors)
will only be reported in Platform logs. If the expected data is not available in
cloud.redhat.com, the customer should engage with support.

## Development

#### Prerequisites

Golang >= 1.21

**macOS additional dependencies (for running with `-tags dynamic`):**

```bash
brew install pkg-config librdkafka
```

These are required for building with the `dynamic` tag, which links against the system librdkafka library for Kafka support.

#### Launching the Service

Compile the source code into a go binary:

    $> make build

Launch the application

    $> ./insights-ingress-go

The server should now be available on TCP port 3000.

    $> curl http://localhost:3000/api/ingress/v1/version

#### The Docker Option

You can also build ingress using Docker/Podman with the provided Dockerfile.

    $> docker build . -t ingress:latest

### Local Development

More information on local development can be found [here](./development/README.md)

#### Uploading a File (On-Prem)

For on-prem deployments with `INGRESS_AUTH=false`, the `x-rh-identity` header is **optional** - the service will automatically add the standard test identity if it's missing:

```bash
# Simple upload without identity header (on-prem only)
curl -F "file=@somefile.tar.gz;type=application/vnd.redhat.openshift.periodic+tgz" \
  http://localhost:3000/api/ingress/v1/upload

# Or with explicit identity header
curl -F "file=@somefile.tar.gz;type=application/vnd.redhat.openshift.periodic+tgz" \
  -H "x-rh-identity: eyJpZGVudGl0eSI6IHsidHlwZSI6ICJVc2VyIiwgImFjY291bnRfbnVtYmVyIjogIjAwMDAwMDEiLCAib3JnX2lkIjogIjAwMDAwMSIsICJpbnRlcm5hbCI6IHsib3JnX2lkIjogIjAwMDAwMSJ9fX0=" \
  http://localhost:3000/api/ingress/v1/upload
```

The standard test identity used by default:
- **Base64**: `eyJpZGVudGl0eSI6IHsidHlwZSI6ICJVc2VyIiwgImFjY291bnRfbnVtYmVyIjogIjAwMDAwMDEiLCAib3JnX2lkIjogIjAwMDAwMSIsICJpbnRlcm5hbCI6IHsib3JnX2lkIjogIjAwMDAwMSJ9fX0=`
- **Decoded**: `{"identity": {"type": "User", "account_number": "0000001", "org_id": "000001", "internal": {"org_id": "000001"}}}`

#### Testing

Use `go test` to test the application

    $> make test
