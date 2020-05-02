---
layout: post
title:  Spring upload and download files to S3 with localstack
author: frandorado
categories: [spring]
tags: [spring, s3, localstack, upload, download]
image: assets/images/posts/2020-04-18/image.png
toc: true
hidden: false
---


In this post we are going to create an example of REST Controllers for upload and download files in AWS S3 using LocalStack.

## The controllers

The upload controller will use a MultipartFile to receive the file. The max filename size could be configured by properties.

```
@PostMapping(value = “/upload”, produces = “application/json”)
public ResponseEntity<String> upload(@RequestParam(“file”) MultipartFile file) {
    String key = storageService.upload(file);
    return new ResponseEntity<>(key, HttpStatus.OK);
}
```

The download controller will return a Resource file.

```
@GetMapping(“/download”)
public ResponseEntity<Resource> download(String id) {
    DownloadedResource downloadedResource = storageService.download(id);
        
    return ResponseEntity.ok()
		.header(HttpHeaders.CONTENT_DISPOSITION, “attachment; filename=“ 
			+ downloadedResource.getFileName())
          	.contentLength(downloadedResource.getContentLength())
		.contentType(MediaType.APPLICATION_OCTET_STREAM)
          	.body(new InputStreamResource(downloadedResource.getInputStream()));
}
```

## The Storage Service

One storage service will have to provide the implementation for the next interface:

```
public interface StorageService {
    
    String upload(MultipartFile multipartFile);
    
    DownloadedResource download(String id);
}
```

In our example, we provide a S3StorageService with this implementation and uses AmazonS3 client to upload and upload the file.

* The upload put the object in a bucket in S3 using the InputStream of MultipartFile and add some extra object metadata (content length, content type and file extension)

	```
	amazonS3.putObject(bucketName, key, multipartFile.getInputStream(), extractObjectMetadata(multipartFile));
	```

* The download create a DownloadedResource object with all the information stored in S3.

## Configuration for S3 in Localstack

First we should create a docker-compose.yml file where indicates the localstack service with S3 enabled.

```
services:
  localstack:
    image: localstack/localstack
    ports:
      - “4572:4572”
    environment:
      - SERVICES=s3
      - DOCKER_HOST=unix:///var/run/docker.sock
    volumes:
      - “${TMPDIR:-/tmp/localstack}:/tmp/localstack”
      - “/var/run/docker.sock:/var/run/docker.sock”
```

Then we have to create the AmazonS3 client like that:

```
@Value(“${aws.s3.endpoint-url}”)
private String endpointUrl;
    
@Bean
AmazonS3 amazonS3() {
    EndpointConfiguration endpointConfiguration = new EndpointConfiguration(endpointUrl,
                Regions.US_EAST_1.getName());
                
    return AmazonS3ClientBuilder.standard()
                                .withEndpointConfiguration(endpointConfiguration)
                                .withPathStyleAccessEnabled(true).build();
}
```

The value for `aws.s3.endpoint-url` will be defined in a property file with the value `http://localhost:4572`


# References

[1] [Link to the project in Github](https://github.com/frandorado/spring-projects/tree/master/spring-upload-s3-localstack)

