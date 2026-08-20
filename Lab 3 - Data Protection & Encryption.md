Lab 3 – Data Protection & Encryption

Name:Sicid Sukhra Abdirahman

ID:52215124928

 Objective

The objective of this lab is to understand different ways of protecting data using encryption. In this lab, I used AES, RSA, TLS, and KMS to protect data at rest and in transit. I also learned about envelope encryption, cryptographic erasure, and how hashing can be used to check data integrity.

Environment

The following tools and environment were used to complete this lab:

- Windows 11 with Git Bash for running the commands.
- OpenSSL for AES, RSA, digital signatures, and TLS certificates.
- Docker and Nginx for the TLS demonstration.
- AWS CLI with LocalStack for KMS operations.
- LocalStack KMS endpoint: `http://localhost:4566`.
- Test file: `record.txt`, containing a sample confidential patient record.





Session A — Encryption Fundamentals


 
 
 Task 1 – Symmetric Encryption with AES



In this task, I created a sample patient record and used AES-256-CBC encryption to protect the file.

 Commands

```bash
echo 'Patient: Ahmad, Diagnosis: confidential' > record.txt

openssl enc -aes-256-cbc -pbkdf2 -salt -in record.txt -out record.enc

cat record.enc

openssl enc -d -aes-256-cbc -pbkdf2 -in record.enc -out record.dec.txt

diff record.txt record.dec.txt && echo 'MATCH: decryption successful'
```

Result

The patient record was encrypted successfully using AES-256-CBC. The encrypted file was not readable as normal text. After decrypting the file with the correct password, the original data was recovered. The final output showed `MATCH: decryption successful`, confirming that the decrypted file matched the original file.

Evidence

<img width="446" height="262" alt="lab 3 task 1" src="https://github.com/user-attachments/assets/ac3026db-5a48-4ddc-9b82-9a442aceb01b" />














Task 2 – Asymmetric Encryption and Digital Signatures






In this task, I used RSA public and private keys to encrypt and decrypt the patient record. I also created a digital signature and verified it using the public key.

Commands

```bash
openssl genrsa -out private.pem 2048

openssl rsa -in private.pem -pubout -out public.pem

openssl pkeyutl -encrypt -pubin -inkey public.pem -in record.txt -out record.rsa

openssl pkeyutl -decrypt -inkey private.pem -in record.rsa -out record.rsa.txt

cat record.rsa.txt

openssl dgst -sha256 -sign private.pem -out record.sig record.txt

openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```

 Result

The patient record was encrypted using the public key and successfully decrypted using the private key. A digital signature was also created using the private key. The final output showed `Verified OK`, confirming that the signature was valid and the file had not been changed.

Evidence

<img width="441" height="156" alt="lab 3 task 2" src="https://github.com/user-attachments/assets/2625a36a-8fbd-47da-8dbb-0acb0067f0af" />




 
 

 Task 3 – Encryption in Transit with TLS




In this task, I used TLS to protect data while it was being transferred. I created a self-signed certificate and used Nginx to provide an HTTPS connection.

Commands

```bash
MSYS_NO_PATHCONV=1 openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem \
  -days 7 -nodes -subj '/CN=localhost'

cat <<'EOF' > tls.conf
server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate /etc/nginx/cert.pem;
    ssl_certificate_key /etc/nginx/key.pem;

    location / {
        root /usr/share/nginx/html;
    }
}
EOF

MSYS_NO_PATHCONV=1 docker run -d --name tls -p 8443:443 \
  --mount type=bind,source="$(pwd)/cert.pem",target=/etc/nginx/cert.pem,readonly \
  --mount type=bind,source="$(pwd)/key.pem",target=/etc/nginx/key.pem,readonly \
  --mount type=bind,source="$(pwd)/record.txt",target=/usr/share/nginx/html/record.txt,readonly \
  --mount type=bind,source="$(pwd)/tls.conf",target=/etc/nginx/conf.d/default.conf,readonly \
  nginx

docker ps --filter name=tls

curl -k https://localhost:8443/record.txt
```

Result

The TLS container started successfully and HTTPS was available on port 8443. The `curl` command returned the patient record successfully through the HTTPS connection. This showed that TLS was working and the data could be transferred securely.

Evidence

<img width="453" height="137" alt="lab 3 task 3" src="https://github.com/user-attachments/assets/10e52baf-e505-4499-9bf0-031432b6702d" />





Session B — Key Management, Envelope Encryption and Erasure





Task 4 – Create and Use a KMS Master Key



In this task, I used LocalStack KMS to create a master key for Tenant A. The key was then used to encrypt a small test value.

 Commands

```bash
EP='--endpoint-url=http://localhost:4566'

aws $EP kms create-key --description 'CCSE tenant-A master key'

KEY_A=454d1904-0a11-48a0-995c-456c645e4a2d

echo $KEY_A

aws $EP kms encrypt --key-id $KEY_A --plaintext "$(echo -n 'hello' | base64)" \
  --query CiphertextBlob --output text
```

Result

The KMS master key for Tenant A was created successfully. The Key ID was saved in the `KEY_A` variable and used to encrypt the test value `hello`. The command returned a ciphertext value, showing that KMS encryption was successful.

Evidence

<img width="470" height="103" alt="lab 3 task 4" src="https://github.com/user-attachments/assets/9623bcc2-d18b-4b78-9524-0b53f5c7e7b1" />





Task 5 – Envelope Encryption



In this task, I used a data key to encrypt the patient record. The plaintext data key was used for local encryption, while the encrypted copy of the data key was kept for protection with KMS.

Commands

```bash
cat datakey.json

echo 'R6aQ+EO1vf4bFXsWVtLitniEjt7p43YSnc121fLDhaw=' > datakey.b64

echo 'NDU0ZDE5MDQtMGExMS00OGEwLTk5NWMtNDU2YzY0NWU0YTJk8NkRFMasGyY3NhqKa21SnYDtvQc1OTSOoTOWOD9VpdIBl7TFka8lBvtrMHdXDRgTuPnFRsbIAy/0qkGhJrfJiK4K4VxyUw5mDWvhnK870MY=' > datakey.enc

base64 -d datakey.b64 > datakey.bin

openssl enc -aes-256-cbc -pbkdf2 -in record.txt -out record.env.enc \
  -pass file:./datakey.bin

ls -l record.env.enc datakey.enc

rm datakey.bin datakey.b64

echo 'Only the KMS-wrapped data key (datakey.enc) remains.'
```

Result

The patient record was successfully encrypted using the data key. After encryption, the plaintext copy of the data key was removed. Only the KMS-wrapped data key remained. This showed how envelope encryption can protect the encryption key as well as the data.

Evidence

<img width="460" height="140" alt="lab 3 task 5" src="https://github.com/user-attachments/assets/3d00fb41-00af-4ca0-ae4c-d427ea8bcadf" />


 
 

 Task 6 – Per-Tenant Keys and Cryptographic Erasure



In this task, I created a separate KMS key for Tenant B and tested cryptographic erasure for Tenant A. The purpose was to show that when the key protecting encrypted data is disabled or removed, the encrypted data can no longer be recovered.

 Commands

```bash
aws $EP kms create-key --description 'CCSE tenant-B master key'

KEY_B=<TENANT_B_KEY_ID>

echo $KEY_B

aws $EP kms schedule-key-deletion --key-id $KEY_A --pending-window-in-days 7

aws $EP kms disable-key --key-id $KEY_A

aws $EP kms decrypt --ciphertext-blob fileb://datakey.enc 2>&1 | head -3
```

Result

A separate KMS key was created for Tenant B. Tenant A's master key was then scheduled for deletion and disabled to demonstrate cryptographic erasure.

After the Tenant A key became unavailable, the attempt to decrypt the wrapped data key failed. This showed that encrypted data cannot be recovered when the key required to decrypt it is no longer available.

Evidence

<img width="509" height="58" alt="lab3 task 6" src="https://github.com/user-attachments/assets/a931ee62-1192-45f6-a8e3-c3cd8b420c54" />








Task 7 – Integrity and Tamper-Evidence



In this task, I used SHA-256 hashing to check file integrity. I also changed a copy of the original file to see how a small change affects the hash value. Finally, I created a simple hash chain to show how changes in records can be detected.

 Commands

```bash
sha256sum record.txt

cp record.txt tampered.txt
echo 'x' >> tampered.txt

sha256sum record.txt tampered.txt

PREV=0

for line in 'login ok' 'file read' 'export data'; do \
  PREV=$(echo -n "$PREV$line" | sha256sum | cut -d' ' -f1); \
  echo "$line | $PREV"; done
```

 Result

The original file and the tampered file produced different SHA-256 hash values. This showed that even a small change in the data can be detected using hashing.

The hash chain also generated a new hash for each record based on the previous hash. This helps detect if a record in the chain has been changed.

 Evidence

<img width="507" height="248" alt="lab 3 task 7" src="https://github.com/user-attachments/assets/d1e387e2-f380-424a-8c08-f160b5f889d8" />



 Verification

The following commands were used to verify that the encrypted files were created successfully and to check the KMS keys used during the lab.

 Commands

```bash
ls -lh record.enc record.rsa record.env.enc

aws $EP kms list-keys
```

 Result

The verification showed that the AES, RSA, and envelope-encrypted files were created successfully. The KMS key list also showed the keys that were created during the key management tasks.

Evidence

<img width="464" height="141" alt="Verify encrypted " src="https://github.com/user-attachments/assets/5077b672-fd0a-49a5-afff-573367d994e3" />



Cleanup & Teardown

After completing and verifying the lab tasks, I cleaned up the environment by stopping the TLS container, removing the files created during the lab, and removing the LocalStack container.

Commands

```bash
docker stop tls 2>/dev/null

rm -f record.* private.pem public.pem key.pem cert.pem datakey.* tampered.txt

docker stop localstack && docker rm localstack
```

 Result

The TLS container was stopped and the temporary files created during the lab were removed. The LocalStack container was also stopped and removed. This cleaned the lab environment after completing all tasks.

Evidence




<img width="468" height="152" alt="Cleanup   Teardown" src="https://github.com/user-attachments/assets/f0923b35-f693-4fdf-aeb7-ae77207ce0b8" />
















Short-Answer Questions



Q1. Compare symmetric and asymmetric encryption: speed, key distribution, and typical use?

 
 The symmetric key algorithm makes use of the same key for both encryption and decryption processes. Symmetric key algorithms are quick, and they are generally applied to encrypt large volumes of data. Nevertheless, the key has to be transferred in a safe manner. The asymmetric key algorithm makes use of the public key and the private key. It takes time, but distributing the key is easier.

 
 Q2. Why is key management described as the weakest link, not the algorithm?

 
 The latest encryption algorithms are highly secure, but if the key is not managed properly, the entire system will fall apart. The attacker can decrypt the message without having to break the encryption algorithm as long as he has got the key. This highlights the importance of key management.

 
 Q3. Explain envelope encryption and why only the master key needs hardware-grade protection?

 
 Envelope encryption uses a data key to encrypt the actual data, and then a master key is used to protect the data key. This means the master key does not need to encrypt all the data directly. The master key needs stronger protection because it protects the other encryption keys.

 

 
 Q4. How does cryptographic erasure achieve provable deletion where overwriting cannot (in the 
cloud)? 


In cloud environments, data may have copies or backups in different storage locations, so overwriting every copy can be difficult. Cryptographic erasure deletes or makes the encryption key unavailable. Without the key, the encrypted data cannot be decrypted even if a copy of the data still exists.


Q5. How does a hash chain make a log tamper-evident (link to tamper-proof logs, Week 6)? 


Each log entry in the hash chain relies on the hash of the previous entry to construct its hash. In light of the fact that the entries are linked, any alteration of an older log entry would cause a ripple effect on subsequent hashes.



 
 
 Security Checklist

The following security practices were completed and verified during the lab:


 ☑ Data was encrypted at rest using AES and the decryption was verified.

 
 ☑RSA public and private keys were used for encryption and digital signatures.

 
 ☑Data was successfully retrieved over a TLS connection.

 
 ☑Envelope encryption was used and the plaintext data-key files were removed.

 
 ☑Per-tenant KMS keys were used and cryptographic erasure was demonstrated.

 
 ☑SHA-256 hashing and a tamper-evident hash chain were demonstrated.
