Secure Isolation & Multi-Tenancy

Name:Sicid Sukhra Abdirahman  
Course:IKB42603 Cloud Computing  
Lab 2 — Secure Isolation and Multi-Tenancy

Objective

The objective of this lab is to understand how security and isolation work in a multi-tenant cloud environment. In this lab, I used Kubernetes namespaces to separate two tenants and applied security controls such as ResourceQuota, NetworkPolicy, and RBAC. I also tested secure data deletion and learned about the risk of data remanence.

Environment

- Kubernetes cluster: `ccse-lab2` using kind
- Namespaces: `tenant-a` and `tenant-b`
- Web application: Nginx
- Network security: Calico NetworkPolicy
- Test tool: `curlimages/curl`
- Storage: Docker volume `ccse-vol`

   Task 1 — Two Tenants on One Cluster

In this task, I created two separate namespaces, `tenant-a` and `tenant-b`, to represent two tenants using the same Kubernetes cluster. I deployed an Nginx web application in each namespace and exposed each deployment as a service.

 Commands

```bash
kubectl create namespace tenant-a
kubectl create namespace tenant-b

kubectl -n tenant-a create deployment web --image=nginx
kubectl -n tenant-b create deployment web --image=nginx

kubectl -n tenant-a expose deployment web --port=80
kubectl -n tenant-b expose deployment web --port=80

kubectl get pods,svc -n tenant-a
```

Result

The Nginx pod in `tenant-a` was running successfully and the `web` service was available on port `80/TCP`. This confirmed that the tenant workload and service were created successfully.

Evidence


<img width="451" height="96" alt="task step 2" src="https://github.com/user-attachments/assets/8f87e434-8b88-4deb-aab4-e5aa11450569" />


Task 2 — Observe the Default-Open Risk

In this task, I tested whether `tenant-a` could access the web service in `tenant-b` before applying a NetworkPolicy. The purpose was to check the default network communication between the two tenants.

Command

```bash
kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never \
  -- curl -s -m 5 http://10.96.130.249 -o /dev/null -w 'HTTP %{http_code}\n'
```

 Result

The request returned `HTTP 200`, which means `tenant-a` was able to reach the web service in `tenant-b`. This shows that namespaces alone do not block network communication between tenants.

 Evidence

<img width="447" height="124" alt="task 2" src="https://github.com/user-attachments/assets/6b7f65f2-434d-470b-aa9f-9312460b38d6" />


 Task 3 — Contain the Noisy Neighbour

In this task, I applied a `ResourceQuota` to `tenant-a` to limit how many shared resources the tenant could use. This helps prevent one tenant from using too much CPU, memory, or too many pods in the shared cluster.

 Command

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 512Mi
    pods: "5"
EOF

kubectl describe resourcequota tenant-a-quota -n tenant-a
```

Result

The ResourceQuota was created successfully. The hard limits for `tenant-a` were:

- Maximum pods: `5`
- CPU requests: `1`
- Memory requests: `512Mi`

This helps stop one tenant from exhausting the shared cluster resources.

Evidence


<img width="453" height="95" alt="task 3" src="https://github.com/user-attachments/assets/f13c7ed7-40f7-4e4c-82f8-38524ae85e87" />


Task 4 — Default-Deny Network Isolation

In this task, I applied a default-deny ingress NetworkPolicy to `tenant-b`. The purpose was to block incoming traffic to the pods in `tenant-b` unless the traffic is specifically allowed.

 Command

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: tenant-b
spec:
  podSelector: {}
  policyTypes: [Ingress]
EOF
```

 Verification

After applying the NetworkPolicy, I tested the connection from `tenant-a` to the web service in `tenant-b` again.

```bash
kubectl logs probe -n tenant-a
```

Result

The test returned:

```text
HTTP 000
```

Before applying the NetworkPolicy, the connection returned `HTTP 200`. After applying the default-deny policy, it returned `HTTP 000`. This confirms that incoming cross-tenant traffic to `tenant-b` was blocked.

 Evidence
 
<img width="448" height="128" alt="task 4" src="https://github.com/user-attachments/assets/332f72c1-4ab2-42d2-9556-0c2331d40d81" />

 Task 5 — Storage & Secret Isolation

In this task, I tested secret isolation between the two tenants using Kubernetes RBAC. Separate secrets were created in `tenant-a` and `tenant-b`. A service account called `app-a` was then given permission to read secrets only in `tenant-a`.

Commands

```bash
kubectl -n tenant-a create secret generic data --from-literal=value=SECRET_A
kubectl -n tenant-b create secret generic data --from-literal=value=SECRET_B

kubectl -n tenant-a create serviceaccount app-a

kubectl -n tenant-a create role reader --verb=get --resource=secrets

kubectl -n tenant-a create rolebinding rb \
  --role=reader \
  --serviceaccount=tenant-a:app-a

SA=system:serviceaccount:tenant-a:app-a

kubectl auth can-i get secrets -n tenant-a --as=$SA
kubectl auth can-i get secrets -n tenant-b --as=$SA
```

Result

The authorization test returned:

```text
tenant-a: yes
tenant-b: no
```

This means the `app-a` service account can read secrets inside its own namespace, `tenant-a`, but it cannot read secrets from `tenant-b`. This confirms that RBAC is enforcing tenant isolation.

Evidence
<img width="443" height="182" alt="task 5" src="https://github.com/user-attachments/assets/d941e2bb-9f40-42f4-b2b2-cb72e19904d0" />

 Task 6 — Data Remanence & Secure Deletion

In this task, I tested what happens when sensitive data is deleted from storage and learned about data remanence. I also tested overwriting sensitive data with zeros before deleting the file.

 Data Remanence Test

```bash
docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE-PATIENT-RECORD > /data/phi.txt; sync; rm /data/phi.txt; \
  grep -a SENSITIVE /data/* 2>/dev/null; echo scan-done'
```

Result

The command created sensitive data, wrote it to the Docker volume, deleted the file, and performed a scan. The command finished with:

```text
scan-done
```

Secure Deletion Test

```bash
docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE > /data/phi2.txt; sync; \
  dd if=/dev/zero of=/data/phi2.txt bs=1k count=1 conv=notrunc; rm /data/phi2.txt; echo wiped'
```

 Result

The `dd` command overwrote the file with zeros before the file was deleted. The output showed:

```text
1+0 records in
1+0 records out
1024 bytes (1.0KB) copied
wiped
```

This task helped me understand that deleting a file does not always mean that all traces of its data are immediately removed. Overwriting sensitive data before deletion is one method used to reduce the risk of data remanence.

Evidence
<img width="448" height="176" alt="task 6" src="https://github.com/user-attachments/assets/9a9e1ee5-a9aa-4d0a-b87c-55ebb521f31f" />

Final Verification

The final verification was performed to confirm that the NetworkPolicy and ResourceQuota created during the lab were still configured correctly.

 Commands

```bash
kubectl get networkpolicy -A
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

Result

The NetworkPolicy verification showed that `default-deny-ingress` was active in `tenant-b`.

The ResourceQuota verification confirmed that `tenant-a-quota` was still configured with the required resource limits:

- Maximum pods: `5`
- CPU requests: `1`
- Memory requests: `512Mi`

These results confirm that the network isolation and resource limits configured during the lab were successfully applied.

Evidence
<img width="441" height="221" alt="final ver" src="https://github.com/user-attachments/assets/157646d1-ecd3-4eb7-8306-e7cb99892c40" />

 Cleanup & Teardown

After completing all tasks and verification, I removed the resources that were created for Lab 2. This included deleting the Kubernetes cluster and the Docker volume used for the data-remanence test.

Commands

```bash
kind delete cluster --name ccse-lab2
docker volume rm ccse-vol
```

 Result

The `ccse-lab2` cluster and `ccse-vol` Docker volume were successfully removed. This completed the cleanup of the Lab 2 environment.

 Evidence
<img width="337" height="107" alt="image" src="https://github.com/user-attachments/assets/b562c33e-b827-453a-9d1f-93a10a6e879f" />

Short-answer question
 Q1. Why can pods in various namespaces communicate with each other by default, and why is it risky in multi-tenancy cloud? 
Answer: 
By default, Kubernetes namespaces do not enforce any kind of network separation for the traffic between pods. It means a pod in tenant-a can talk to the service in tenant-b. It is risky in a multi-tenant cloud as tenants will be able to access other tenants' services if network segregation is not done.
Q2. What does the default-deny policy mean, and how does your NetworkPolicy support it?
Answer: 
Default-deny implies that all kinds of incoming traffic are denied unless explicitly allowed. In this case, the default-deny-ingress NetworkPolicy denies all kinds of incoming traffic to pods in tenant-b. After deploying this policy, the connection from tenant-a was denied.
Q3. Which one of them provides better isolation and when will you use a VM isolation boundary?
Answer:
Containers have more similarities to the underlying system, while virtual machines have better isolation boundaries. A VM isolation boundary can be used when better isolation is required among tenants.
Note: the lab poses this question, but doesn't describe VM vs container comparison, hence a short explanation of my interpretation based on the topic of the lab.
Q4. What is meant by data remanence, and why should cryptographic erasure be used in the cloud?
Answer:
Data remanence implies that some amount of data can still remain after file deletion. In cloud storage users generally do not control the underlying storage blocks. Therefore, according to the lab, cryptographic erasure is the only option: destroy encryption keys so that the data that remains cannot be decrypted anymore.
Q5. Which isolation dimension did each of the tasks perform: compute, network, or storage?
Answer:
Task 1 and Task 3 dealt with compute isolation as they involved isolating tenants in namespaces with the limited sharing of resources with the help of quota. Task 2 and Task 4 dealt with network isolation, as they tested cross-tenant traffic both before and after deploying NetworkPolicy. Task 5 dealt with storage isolation as it stopped tenant-a from accessing tenant-b's secret by using RBAC. Task 6 dealt with secure data deletion and data remanence.

Conclusion

In this lab, I learned how different security controls can be used to protect tenants that share the same Kubernetes cluster. I used namespaces to separate tenants, ResourceQuota to limit resource usage, NetworkPolicy to block cross-tenant traffic, and RBAC to control access to secrets. I also learned about data remanence and why sensitive data needs to be handled carefully when it is deleted. Overall, this lab helped me understand how compute, network, and storage isolation work together in a multi-tenant cloud environment.

