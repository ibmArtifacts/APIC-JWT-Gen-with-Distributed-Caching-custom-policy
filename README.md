# APIC-JWT-Gen-with-Distributed-Caching-custom-policy
Documentation of a JWT-gen, which JWT is set for a backend call. In addition, the APIC application name is injected into the JWT, with thue JWT distributively cached across multiple gateway nodes (if applicable).
This solution was developed for APIC v10.0.8.x. 
Due to the distributed variable feature, this solution is not supported on APIC v10.0.5.x LTS or below.
The distributed variable feature was presented on DataPower firmware 10.5.x (CD version), which is aligned with APIC v10.0.7.x (CD version).  

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
  
### JWT-Gen-with-Distributed-Caching: Policies
![image](https://github.com/user-attachments/assets/07f2da17-efc0-484b-b06b-db0c1919e393)  
#### 1. xslt-get-cert-serial  
The purpose of this stylesheet is to get the certificate serial number of the crypto objects that will be used to sign the JWT, and verify whether the cached (if found) cert serial number is the same.
- The XSLT policy is used because the datapower extension to retrieve the certificate serial number is not available on gatewayscript.  
  
The certificate serial number is set to a variable to be used in the next policy.  

#### 2. gwyscript: get-cache  
This gatewayscript will take the certificate serial number set from xslt-get-cert-serial policy, and compare the cert serial number from the cache, in addition check the time-to-live (ttl) value.
- If the cert serial does not match the cached cert serial number, or if the time have past the ttl value set, then a flag will be set to generate a new JWT.  
- If the time passed is within the ttl value, and cert serial numbers match, then the cached JWT will be set to the header to the backend, thus skipping the JWT-gen processing.

#### 3. switch  
The switch policy will be checking whether the variable jwtNotFound is set to "true". If so, then the policies within this flow will trigger a new JWT and cache the newly generated content.  

#### 4. gwyscript: set variables  
This policy starts off the JWT generation flow.  
This policy sets the issuer claim as apic, the audience claim (developer defined), takes the client application name to inject into the JWT, the crypto object name that is used to sign the JWT, and sets up all the properties to be used in the next policy.  

#### 5. jwt-generate  
The jwt-generate uses all the properties set from the previous gatewayscript.  
- The issuer claim is hard-coded as apic.  
- The audience claim is developer input into the custom policy palette.  
- The validity of the JWT expiration.  
- Hard-coded Cryptographic Algorithm of RS256.  
- The crypto object name variable that will be used to sign the JWT.

#### 6. xslt-set-cert-serial  
This stylesheet is used to call the datapower extension dp:get-cert-serial() to set the cert serial number property at the time the JWT is generated for the next policy.  

#### 7. gwyscript: cache
This caches the newly generated JWT with the current value of "date|cert_serial_number, JWT_value" into the distibuted caching store as key 'var://dist/x-ibm-clientid/CLIENT_ID_OF_CONSUMER'.  
When the consumer calls the API again within the TTL seconds, the gatewayscript get-cache will check for this cached value, and reuse the JWT instead of generating a new one if found.  

#### 8. gwyscript: set jwt header
This last policy sets the newly generated JWT to the header for the backend to consume.  





