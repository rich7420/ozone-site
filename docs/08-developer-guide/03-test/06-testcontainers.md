---
sidebar_label: Testcontainers
---

# Testcontainers

The `ozone-testcontainers` module lets JVM integration tests start single-node
Ozone with S3 Gateway using [Testcontainers](https://java.testcontainers.org/).
It wraps `apache/ozone:2.2.0-all-in-one`, which runs SCM, OM, a Datanode and
S3 Gateway in one container.

This initial module supports S3 access in non-secure mode and requires a running
Docker daemon.

## Dependency

The module is unreleased and is being added in
[HDDS-16077](https://issues.apache.org/jira/browse/HDDS-16077)
([code PR](https://github.com/apache/ozone/pull/10966)). Until it is released,
build and install it from the source branch:

```shell
mvn -pl :ozone-testcontainers -am install -DskipTests -DskipDocs
```

Add the module to the project's test dependencies:

```xml
<dependency>
  <groupId>org.apache.ozone</groupId>
  <artifactId>ozone-testcontainers</artifactId>
  <version>2.3.0-SNAPSHOT</version>
  <scope>test</scope>
</dependency>
```

The following example also requires the AWS SDK for Java v2 S3 client as a test
dependency; the Ozone module does not include it.

## Usage

```java
try (OzoneContainer ozone = new OzoneContainer()) {
  ozone.start();

  try (S3Client s3 = S3Client.builder()
      .endpointOverride(URI.create(ozone.getS3Endpoint()))
      .credentialsProvider(StaticCredentialsProvider.create(
          AwsBasicCredentials.create(ozone.getAccessKey(), ozone.getSecretKey())))
      .region(Region.of(ozone.getRegion()))
      .forcePathStyle(true)
      .build()) {

    s3.createBucket(b -> b.bucket("my-bucket"));
    // ... put / get / list objects
  }
}
```

In non-secure mode, S3 Gateway accepts any access key and secret. The accessors
provide a consistent set of credentials for signing requests; the fluent setters
change these returned values, rather than configure authentication in Ozone.

The container waits for the S3 Gateway port, then retries
`ozone admin safemode wait` until SCM leaves safe mode. The startup timeout
defaults to four minutes and can be changed with `withStartupTimeout(Duration)`.

## Migrating from MinIO

Tests using the MinIO Testcontainers module can use these Ozone equivalents:

| MinIO | Ozone |
|-------|-------|
| `new MinIOContainer(...)` | `new OzoneContainer()` |
| `getS3URL()` | `getS3Endpoint()` |
| `getUserName()` | `getAccessKey()` |
| `getPassword()` | `getSecretKey()` |
| n/a | `getRegion()` |
