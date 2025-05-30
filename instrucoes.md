# Spring Batch Resilient S3 Multipart Upload Process

## 1. Overview

* **Objective**: Read records from a relational database and write directly to an S3 object in a resilient, chunk-oriented batch process, **without using any local temporary files**.
* **Key Requirements**:

  * Java 17
  * Spring Batch 5.1.1 (using new JobBuilder/StepBuilder API)
  * AWS SDK v2 for S3
  * Configured S3 bucket and credentials
  * Multipart upload parts must be at least 5 MiB (except the last part)

## 2. Dependencies

* Spring Batch Core 5.1.1
* AWS SDK v2 (S3 module)
* JDBC driver for your database (e.g., PostgreSQL, MySQL)

## 3. Job & Step Configuration

1. **Define a Job** with a single Step.
2. **Configure the Step** for chunk-oriented processing (e.g., chunks of 500 items).
3. **Enable fault tolerance** on the Step:

   * Set retry for network I/O exceptions (e.g., IOException).
   * Define a retry limit (e.g., 3 attempts).

## 4. Reader Configuration

* Use a `JdbcPagingItemReader` or `JdbcCursorItemReader`.
* Set `pageSize` equal to the chunk size.
* Define the SQL clauses (SELECT, FROM) and sorting keys (e.g., ORDER BY primary key).

## 5. Writer: S3 Multipart Upload

1. **Implement an ItemWriter** that also implements `StepExecutionListener`.
2. **beforeStep**:

   * Retrieve an existing uploadId from the `ExecutionContext`, or
   * Initiate a new multipart upload on S3 and store the `uploadId` in the context.
3. **write(List<T> items)**:

   * Convert each item to a line of text (CSV or TXT), adding a newline.
   * Append bytes to an in-memory buffer.
   * When the buffer size ≥ 5 MiB, upload a part using `UploadPart`.
   * Store the part metadata (part number and ETag) for later completion.
4. **afterStep**:

   * If buffer contains data (< 5 MiB), upload it as the final part.
   * Complete the multipart upload (`CompleteMultipartUpload`) with all collected parts.
   * On error, abort the multipart upload (`AbortMultipartUpload`) to avoid orphaned uploads.

## 6. Parameterization via JobParameters

* Pass the S3 **bucket name** and **object key** (path) as job parameters.
* Scope the writer bean to the Step (`@StepScope`) to inject these parameters at runtime.

## 7. Executing the Job

* Launch via Maven or Spring Boot CLI, providing `--bucket` and `--key`:

  ```bash
  mvn spring-boot:run -Dspring-boot.run.arguments="--bucket=meu-bucket,--key=export/saida.csv"
  ```

## 8. Resilience & Restart

* The `ExecutionContext` holds the `uploadId` (and optionally uploaded parts), allowing the Step to resume from a failure.
* Spring Batch’s chunk-oriented model provides retry, skip, and restart capabilities at the chunk boundary.

## 9. Final Considerations

* **No local temp files**: data is buffered in memory and sent directly to S3 parts.
* **AWS S3 requirement**: all parts except the last must be at least 5 MiB.
* **AWS credentials**: ensure `DefaultCredentialsProvider` or environment variables are set.
* **Tune memory and throughput** by adjusting chunk size and part size if necessary.

---

*This document describes all steps and details needed for an AI or developer to implement a resilient, chunk-oriented Spring Batch process in Java 17 / Spring Batch 5.1.1, uploading data directly to S3 via multipart upload.*
