## Dependency ##

```xml
<dependency>
		<groupId>com.amazonaws</groupId>
		<artifactId>aws-java-sdk-s3</artifactId>
		<version>1.12.310</version>
</dependency>
```

## application.properties file ##
```properties
accessKey=******************
secretKey=*************************
region=eu-north-1
cloud.aws.region.static=eu-north-1
cloud.aws.region.auto=false
cloud.aws.stack.audo=false
spring.main.allow-bean-definition-overriding=true
logging.level.com.amazonaws.util.EC2MetadataUtils=error
logging.level.com.amazonaws.internal.InstanceMetadataServiceResourceFetcher=error
```
## Config File ##
```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import com.amazonaws.auth.AWSCredentials;
import com.amazonaws.auth.AWSStaticCredentialsProvider;
import com.amazonaws.auth.BasicAWSCredentials;
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.AmazonS3ClientBuilder;

@Configuration
public class AWSS3Config {

    @Value("${accessKey}")
    private String accessKey;

    @Value("${secretKey}")
    private String secretKey;

    @Value("${region}")
    private String region;

    public AWSCredentials credentials() {
        AWSCredentials credentials = new BasicAWSCredentials(accessKey, secretKey);
        return credentials;
    }

    @Bean
    public AmazonS3 amazonS3() {

        AmazonS3 s3client = AmazonS3ClientBuilder.standard()
                .withCredentials(new AWSStaticCredentialsProvider(credentials())).withRegion(region).build();
        return s3client;
    }
}

```

## Service ##
```java
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.model.AmazonS3Exception;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.File;

@Service
public class BucketService {

    @Autowired
    private AmazonS3 amazonS3;

    public String uploadFile(MultipartFile file, String bucketName) {
        if (file.isEmpty()) {
            throw new IllegalStateException("Cannot upload empty file");
        }
        try {
            File convFile = new File(System.getProperty("java.io.tmpdir") + "/" + file.getOriginalFilename());
            file.transferTo(convFile);
            try {
                amazonS3.putObject(bucketName, convFile.getName(), convFile);
                return amazonS3.getUrl(bucketName, file.getOriginalFilename()).toString();
            } catch (AmazonS3Exception s3Exception) {
                return "Unable to upload file :" + s3Exception.getMessage();
            }



        } catch (Exception e) {
            throw new IllegalStateException("Failed to upload file", e);
        }

    }

//    public String deleteBucket(String bucketName) {
//        amazonS3.deleteBucket(bucketName);
//        return "File is deleted";
//    }

}
```

## Controller ##
```java
import com.s3example1.service.BucketService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

@RestController
@RequestMapping("s3bucket")
@CrossOrigin("*")
public class BucketController {

    @Autowired
    BucketService service;

    @PostMapping(path = "/upload/file/{bucketName}", consumes = MediaType.MULTIPART_FORM_DATA_VALUE, produces = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<String> uploadFile(@RequestParam MultipartFile file,
                                             @PathVariable String bucketName) {
        return new ResponseEntity<>(service.uploadFile(file,bucketName), HttpStatus.OK);
    }
}

//@DeleteMapping(path="/delete/file/{bucketName}/{fileName}")
//public ResponseEntity<String> deleteFile(@PathVariable String bucketName,@PathVariable String fileName)
//{
//    return new ResponseEntity<>(service.deleteFile(bucketName,fileName),HttpStatus.OK);
//}

```
