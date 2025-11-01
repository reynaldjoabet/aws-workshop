# aws-workshop

Using the Default Credential Provider Chain
When you initialize a new service client without supplying any arguments, the AWS SDK for Java attempts to find temporary credentials by using the default credential provider chain implemented by the `DefaultAWSCredentialsProviderChain` class. The default credential provider chain looks for credentials in this order:


- Environment variables-`AWS_ACCESS_KEY_ID`, `AWS_SECRET_KEY` or `AWS_SECRET_ACCESS_KEY`, and `AWS_SESSION_TOKEN`. The AWS SDK for Java uses the EnvironmentVariableCredentialsProvider class to load these credentials.

- Java system properties-`aws.accessKeyId`, `aws.secretKey `(but not aws.secretAccessKey), and aws.sessionToken. The AWS SDK for Java uses the SystemPropertiesCredentialsProvider to load these credentials.

- Web Identity Token credentials from the environment or container.

- The default credential profiles file- typically located at ~/.aws/credentials (location can vary per platform), and shared by many of the AWS SDKs and by the AWS CLI. The AWS SDK for Java uses the ProfileCredentialsProvider to load these credentials.

- You can create a credentials file by using the aws configure command provided by the AWS CLI, or you can create it by editing the file with a text editor. For information about the credentials file format, see AWS Credentials File Format.

- Amazon ECS container credentials- loaded from the Amazon ECS if the environment variable AWS_CONTAINER_CREDENTIALS_RELATIVE_URI is set. The AWS SDK for Java uses the ContainerCredentialsProvider to load these credentials. You can specify the IP address for this value.

- Instance profile credentials- used on EC2 instances, and delivered through the Amazon EC2 metadata service. The AWS SDK for Java uses the InstanceProfileCredentialsProvider to load these credentials. You can specify the IP address for this value.

`Instance profile credentials are used only if AWS_CONTAINER_CREDENTIALS_RELATIVE_URI is not set`

Unlike IAM users, which rely on permanent access keys that must be manually rotated and managed, IAM roles provide temporary security credentials that are automatically managed by AWS Security Token Service (STS).

When an entity assumes an IAM role, AWS generates temporary credentials that include:

- A temporary access key ID
- A temporary secret access key
- A security token
- An expiration timestamp

Dynamic Permission Assignment

IAM roles enable dynamic permission assignment based on context and trust relationships. Instead of pre-assigning permissions to specific users or services, roles allow permissions to be granted temporarily based on the assuming entity’s identity and the trust policies defined for the role.

This approach provides several advantages:

- Principle of Least Privilege: Permissions are granted only when needed and for the minimum duration required.

- Contextual Access: Role assumption can be conditional based on factors like IP address, time of day, or multi-factor authentication status.

- Scalable Permission Management: As organizations grow, managing permissions through roles becomes more efficient than maintaining individual user permissions.

## Service-to-Service Authentication
### Eliminating Hardcoded Credentials

In distributed cloud architectures, services often need to authenticate with other AWS services or external systems. Traditional approaches involve embedding access keys in application code or configuration files, creating significant security risks.

IAM roles provide a more secure alternative through service-linked roles and instance profiles. EC2 instances, Lambda functions, and other AWS services can assume roles to obtain temporary credentials for accessing other AWS resources

### Container and Serverless Integration

Modern cloud-native applications built on containers and serverless architectures benefit significantly from IAM roles. Kubernetes pods can assume IAM roles through service accounts, while Lambda functions automatically receive temporary credentials through their execution roles.


## Federation and Identity Integration
### Enterprise Identity Integration

IAM roles excel in enterprise environments where organizations need to integrate AWS access with existing identity management systems. Through Security Assertion Markup Language (SAML) or OpenID Connect (OIDC) federation, IAM roles can be assumed by users authenticated through corporate identity providers.

This federation approach offers several advantages:

- Single sign-on (SSO) experience for users
- Centralized identity management through existing corporate directories
- Conditional access based on corporate security policies
- No need to maintain separate user accounts in AWS

[what-is-a-unique-advantage-of-aws-iam-roles-understanding-temporary-security-credentials](https://medium.com/@use.abhiram/what-is-a-unique-advantage-of-aws-iam-roles-understanding-temporary-security-credentials-529317cc98f9)


[IAM roles for Amazon EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html)


Identity provider

An identity provider creates, maintains, and manages identity information while offering authentication, authorization, and auditing services.

[intro-vpc](https://cloudsecuritymasterclass.com/aws/intro-vpc/)

[vpc-endpoints](https://cloudsecuritymasterclass.com/vpc-endpoints/)

[aws-session-policy](https://cloudsecuritymasterclass.com/aws-session-policy/)


[how-to-use-trust-policies-with-iam-roles](https://aws.amazon.com/blogs/security/how-to-use-trust-policies-with-iam-roles/#:~:text=An%20IAM%20role%20has%20a,other%20principals%20to%20assume%20it.)

[saml-how-it-works-vulnerabilities-and-common-attacks](https://www.vaadata.com/blog/saml-how-it-works-vulnerabilities-and-common-attacks/)

[microsoft-entra-blog/how-to-break-the-token-theft-cyber-attack-chain](https://techcommunity.microsoft.com/blog/microsoft-entra-blog/how-to-break-the-token-theft-cyber-attack-chain/4062700)


`MDSv2 (Instance Metadata Service Version 2) is ONLY relevant for code running on an EC2 instance`

The Instance Metadata Service (IMDS) provides information about a running EC2 instance to code that is executing on that very instance. This includes vital details like the instance ID, region, security groups, and, most importantly, temporary security credentials for an associated IAM role (instance profile)

```java

/**

 *
 * Utility class for retrieving Amazon EC2 instance metadata.
 *
 * <p>
 * <b>Note</b>: this is an internal API subject to change. Users of the SDK
 * should not depend on this.
 *
 * <p>
 * You can use the data to build more generic AMIs that can be modified by
 * configuration files supplied at launch time. For example, if you run web
 * servers for various small businesses, they can all use the same AMI and
 * retrieve their content from the Amazon S3 bucket you specify at launch. To
 * add a new customer at any time, simply create a bucket for the customer, add
 * their content, and launch your AMI.<br>
 * <p>
 * If {@link SdkSystemSetting#AWS_EC2_METADATA_DISABLED} is set to true, EC2 metadata usage
 * will be disabled and {@link SdkClientException} will be thrown for any metadata retrieval attempt.
 * <p>
 * If {@link SdkSystemSetting#AWS_EC2_METADATA_V1_DISABLED} or {@link ProfileProperty#EC2_METADATA_V1_DISABLED}
 * is set to true, data will only be loaded from EC2 metadata service if a token is successfully retrieved -
 * fallback to load data without a token will be disabled.
 * <p>
 * More information about Amazon EC2 Metadata
 *
 * @see <a
 * href="http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AESDG-chapter-instancedata.html">Amazon
 * EC2 User Guide: Instance Metadata</a>
 */
@SdkInternalApi
public final class EC2MetadataUtils {
    private static final JsonNodeParser JSON_PARSER = JsonNode.parser();

    /** Default resource path for credentials in the Amazon EC2 Instance Metadata Service. */
    private static final String REGION = "region";
    private static final String INSTANCE_IDENTITY_DOCUMENT = "instance-identity/document";
    private static final String INSTANCE_IDENTITY_SIGNATURE = "instance-identity/signature";
    private static final String EC2_METADATA_ROOT = "/latest/meta-data";
    private static final String EC2_USERDATA_ROOT = "/latest/user-data/";
    private static final String EC2_DYNAMICDATA_ROOT = "/latest/dynamic/";

    private static final String EC2_METADATA_TOKEN_HEADER = "x-aws-ec2-metadata-token";

    private static final int DEFAULT_QUERY_ATTEMPTS = 3;
    private static final int MINIMUM_RETRY_WAIT_TIME_MILLISECONDS = 250;
    private static final Logger log = LoggerFactory.getLogger(EC2MetadataUtils.class);
    private static final Map<String, String> CACHE = new ConcurrentHashMap<>();

    private static final Ec2MetadataDisableV1Resolver EC2_METADATA_DISABLE_V1_RESOLVER = Ec2MetadataDisableV1Resolver.create();
    private static final Object FALLBACK_LOCK = new Object();
    private static volatile Boolean IS_INSECURE_FALLBACK_DISABLED;

    private static final InstanceProviderTokenEndpointProvider TOKEN_ENDPOINT_PROVIDER =
            new InstanceProviderTokenEndpointProvider();

    private static final Ec2MetadataConfigProvider EC2_METADATA_CONFIG_PROVIDER = Ec2MetadataConfigProvider.builder()
            .build();

    private EC2MetadataUtils() {
    }

    /**
     * Get the AMI ID used to launch the instance.
     */
    public static String getAmiId() {
        return fetchData(EC2_METADATA_ROOT + "/ami-id");
    }

    /**
     * Get the index of this instance in the reservation.
     */
    public static String getAmiLaunchIndex() {
        return fetchData(EC2_METADATA_ROOT + "/ami-launch-index");
    }

    /**
     * Get the manifest path of the AMI with which the instance was launched.
     */
    public static String getAmiManifestPath() {
        return fetchData(EC2_METADATA_ROOT + "/ami-manifest-path");
    }

    /**
     * Get the list of AMI IDs of any instances that were rebundled to created
     * this AMI. Will only exist if the AMI manifest file contained an
     * ancestor-amis key.
     */
    public static List<String> getAncestorAmiIds() {
        return getItems(EC2_METADATA_ROOT + "/ancestor-ami-ids");
    }

    /**
     * Notifies the instance that it should reboot in preparation for bundling.
     * Valid values: none | shutdown | bundle-pending.
     */
    public static String getInstanceAction() {
        return fetchData(EC2_METADATA_ROOT + "/instance-action");
    }

    /**
     * Get the ID of this instance.
     */
    public static String getInstanceId() {
        return fetchData(EC2_METADATA_ROOT + "/instance-id");
    }

    /**
     * Get the type of the instance.
     */
    public static String getInstanceType() {
        return fetchData(EC2_METADATA_ROOT + "/instance-type");
    }

    /**
     * Get the local hostname of the instance. In cases where multiple network
     * interfaces are present, this refers to the eth0 device (the device for
     * which device-number is 0).
     */
    public static String getLocalHostName() {
        return fetchData(EC2_METADATA_ROOT + "/local-hostname");
    }

    /**
     * Get the MAC address of the instance. In cases where multiple network
     * interfaces are present, this refers to the eth0 device (the device for
     * which device-number is 0).
     */
    public static String getMacAddress() {
        return fetchData(EC2_METADATA_ROOT + "/mac");
    }

}
```