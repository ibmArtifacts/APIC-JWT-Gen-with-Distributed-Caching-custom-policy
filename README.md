# APIC-JWT-Gen-with-Distributed-Caching-custom-policy
This custom policy (user-defined policy) generates a JWT, with the consumer app-name injected into the private claim, and sets it to authorization header downstream (header name is adjustable).  
This solution also uses the distributed variables, which ensures that the cache is synchronized via the gateway-peering instance across gateways nodes within a cluster (https://www.ibm.com/docs/en/datapower-gateway/10.6.0?topic=administration-distributed-variables). 

APIC/DataPower version notes:  
- This solution was developed for APIC v10.0.8.x.  
- Due to the distributed variable feature, this solution is not supported on APIC v10.0.5.x LTS or below.  
- The distributed variable feature was introduced on DataPower firmware 10.5.x (CD version), which is aligned with APIC v10.0.7.x (CD version).


The completed [JWT-Gen-with-Distributed-Caching-custom-policy](https://github.com/ibmArtifacts/APIC-JWT-Gen-with-Distributed-Caching-custom-policy/blob/main/JWT-Gen-with-Distributed-Caching.yaml).
- Zip up the yaml and upload into the catalog you would like to use this on. Remember, a TLS client profile needs to be created with the name 'jwt', and set in the catalog for the custom policy to work properly.  
  
## Overview  
The purpose of this solution is to provide a JWT to establish trust between the APIC gateways to backend systems.  
From a high-level,  
1. This solution will inject the APIC application name of the consumer as the private claim.  
2. It will also use the [DataPower Distributed Variables](https://www.ibm.com/docs/en/datapower-gateway/10.6.0?topic=administration-distributed-variables) to cache the JWT after it is generated based on the Time-to-Live (in seconds) specified so the JWT-Gen does not run on every transaction.  
3. The JWT will be sent downstream with the consumer's request to the backend within the header specified in the custom policy.

The JWT-gen policy uses a crypto object to sign the JWT, therefore, there is a dependency for this custom policy to properly work.
It requires a TLS client profile to be created with the name jwt and set in the catalog which the custom policy will be used on. More instructions will be provided later in the docuemnt.  

## JWT-Gen-with-Distributed-Caching Walk-Through  
All the following policies are used to create the custom policy later.  
A separate API is built to test the functionality of the complete solution before the assembly policies are extracted and put together into a user-defined (custom) policy yaml. The next section describes each policy in-depth for the API, which later, policy code will be used to build out the custom policy.  

The API to test this solution can be downloaded from [jwt-gen-api.yaml](https://github.com/ibmArtifacts/APIC-JWT-Gen-with-Distributed-Caching-custom-policy/blob/main/jwt-gen-api.yaml).  
  
### JWT-Gen-with-Distributed-Caching: Policies
![image](https://github.com/user-attachments/assets/07f2da17-efc0-484b-b06b-db0c1919e393)  
#### 1. xslt-get-cert-serial  
![image](https://github.com/user-attachments/assets/383dda2c-56d6-4fdd-bc8f-2212b4fa9129)  
The purpose of this stylesheet is to get the certificate serial number of the crypto objects that will be used to sign the JWT, and verify whether the cached (if found) cert serial number is the same.
- The XSLT policy is used because the datapower extension to retrieve the certificate serial number is not available on gatewayscript.  
  
The certificate serial number is set to a variable to be used in the next policy.  

#### 2. gwyscript: get-cache  
![image](https://github.com/user-attachments/assets/8a4ce8e8-ba8e-40a1-a251-a44bf853c3db)  

This gatewayscript will take the certificate serial number set from xslt-get-cert-serial policy, and compare the cert serial number from the cache, in addition check the time-to-live (ttl) value.
- If the cert serial does not match the cached cert serial number, or if the time have passed the ttl value set, then a flag will be set to generate a new JWT.  
- If the time passed is within the ttl value, and cert serial numbers match, then the cached JWT will be set to the header to the backend, thus skipping the JWT-gen processing.

#### 3. switch  
![image](https://github.com/user-attachments/assets/b7959695-b923-4f60-b333-d653bd651776)  

The switch policy will be checking whether the variable jwtNotFound is set to "true". If so, then the policies within this flow will trigger a new JWT and cache the newly generated content.  

#### 4. gwyscript: set variables  
![image](https://github.com/user-attachments/assets/f40c93e9-49c8-4f97-8049-b06ca91bef01)  

This policy starts off the JWT generation flow.  
This policy sets the issuer claim as apic, the audience claim (developer defined), takes the client application name to inject into the JWT, the crypto object name that is used to sign the JWT, and sets up all the properties to be used in the next policy.  

#### 5. jwt-generate  
![image](https://github.com/user-attachments/assets/0ecb98a8-3064-4b55-9508-b5dc72299cdc)  

The jwt-generate uses all the properties set from the previous gatewayscript.  
- The issuer claim is hard-coded as apic.  
- The audience claim is developer input into the custom policy palette.  
- The validity of the JWT expiration.  
- Hard-coded Cryptographic Algorithm of RS256.  
- The crypto object name variable that will be used to sign the JWT.

#### 6. xslt-set-cert-serial  
![image](https://github.com/user-attachments/assets/f1eb22c2-8166-4303-896e-35e7727037c1)  

This stylesheet is used to call the datapower extension dp:get-cert-serial() to set the cert serial number property at the time the JWT is generated for the next policy.  

#### 7. gwyscript: cache  
![image](https://github.com/user-attachments/assets/42a72f31-c052-495a-bc3f-7f2b76383c59)  

This caches the newly generated JWT with the current value of "date|cert_serial_number, JWT_value" into the distibuted caching store as key 'var://dist/x-ibm-clientid/CLIENT_ID_OF_CONSUMER'.  
When the consumer calls the API again within the TTL seconds, the gatewayscript get-cache will check for this cached value, and reuse the JWT instead of generating a new one if found.  

#### 8. gwyscript: set jwt header  
![image](https://github.com/user-attachments/assets/100f012e-86b7-4374-96e0-1a40f37e2917)  

This last policy sets the newly generated JWT to the header for the backend to consume.  

## JWT-Gen-with-Distributed-Caching User-Defined (custom) Policy  










