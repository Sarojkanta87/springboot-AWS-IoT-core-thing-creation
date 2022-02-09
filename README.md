# springboot-AWS-IoT-core-thing-creation
AWS IoT Thing creation and auto provisioning using java spring boot

1. Register the RootCA with Verification certificate in AWS IoT core
2. Create the thing and attach device certificate and policy to connect programatically
	a. Create the Thing
	b. Register and activate the Public Key/certificate of the device
	c. Attach the Certificate to the thing.
	d. Attach policies to the thing so that it can connect

So now in detail:


1. Register the RootCA with Verification certificate in AWS IoT core
	
	Follow the link
		https://docs.aws.amazon.com/iot/latest/developerguide/register-CA-cert.html
		

2. Create the thing and attach device certificate and policy to connect programatically

Below steps needed to do the process programatically,

Dependencies needed:
	
	<dependency>
		<groupId>com.amazonaws</groupId>
		<artifactId>aws-iot-device-sdk-java</artifactId>
		<version>1.3.9</version>
	</dependency>

	<dependency>
		<groupId>com.amazonaws</groupId>
		<artifactId>aws-java-sdk-core</artifactId>
		<version>1.12.150</version>
	</dependency>

	<dependency>
		<groupId>com.amazonaws</groupId>
		<artifactId>aws-java-sdk-iot</artifactId>
		<version>1.12.150</version>
	</dependency>


Aws config class:
	
	public class AwsConfig {
	
		@Bean
		public AWSIot getIotClient() {
			return AWSIotClientBuilder.standard()
					.withCredentials(new AWSStaticCredentialsProvider(new BasicAWSCredentials("users_aws_access_key", "users_aws_secret_key")))
					.withRegion("users_aws_region").build();
		}	
	}
	

Service class:

	import org.springframework.beans.factory.annotation.Autowired;
	import org.springframework.stereotype.Service;
	import com.amazonaws.services.iot.model.AttachPolicyRequest;
	import com.amazonaws.services.iot.model.AttachPolicyResult;
	import com.amazonaws.services.iot.model.AttachThingPrincipalRequest;
	import com.amazonaws.services.iot.model.AttachThingPrincipalResult;
	import com.amazonaws.services.iot.model.CertificateStatus;
	import com.amazonaws.services.iot.model.CreateThingRequest;
	import com.amazonaws.services.iot.model.CreateThingResult;
	import com.amazonaws.services.iot.model.DescribeThingRequest;
	import com.amazonaws.services.iot.model.DescribeThingResult;
	import com.amazonaws.services.iot.model.RegisterCertificateRequest;
	import com.amazonaws.services.iot.model.RegisterCertificateResult;
	import com.amazonaws.services.iot.model.ResourceNotFoundException;

	@Service
	public class RegisterService {

		@Autowired
		private AwsConfig iotClient;

		public String RegisterDevice() {

			// check if thing Already exists
			if (!describeThing("Unique Id of Device")) {

				// Thing Creation
				CreateThingResult response = iotClient.getIotClient()
						.createThing(new CreateThingRequest().withThingName("Unique Id of Device/Thing"));

				// Register and activate the Public Key of the device
				RegisterCertificateResult registerCert = iotClient.getIotClient()
						.registerCertificate(new RegisterCertificateRequest().withCaCertificatePem("CA Pem as String")
								.withCertificatePem("Device Public Key in Pem as String").withStatus(CertificateStatus.ACTIVE));

				// Attach the Cert to the thing
				AttachThingPrincipalResult attachThingPrincipalResult = iotClient.getIotClient().attachThingPrincipal(
						new AttachThingPrincipalRequest()
							.withThingName("Unique Id of Device/Thing").withPrincipal(registerCert.getCertificateArn()));

				// Attach policies to the thing so it can connect
				AttachPolicyResult policyResult = iotClient.getIotClient()
						.attachPolicy(new AttachPolicyRequest()
							.withPolicyName("policy_that_allow_device_connections").withTarget(registerCert.getCertificateArn()));

				return "Thing Created Successfully";
			}	

			// Thing exists
			return "Thing Already Exists on IoT Console";
		}

		private boolean describeThing(String thingName) {
			if (thingName == null) {
				return false;
			}
			try {
				describeThingResponse(thingName);
				return true;
			} catch (ResourceNotFoundException e) {
				// e.printStackTrace();
				return false;
			}
		}

		private DescribeThingResult describeThingResponse(String thingName) {
			DescribeThingRequest describeThingRequest = new DescribeThingRequest();
			describeThingRequest.setThingName(thingName);
			return iotClient.getIotClient().describeThing(describeThingRequest);
		}	
	}
