- Установлен кластер consul
```bash
git clone https://github.com/hashicorp/consul-helm.git
helm install consul consul-helm
```i
- Установлен кластер vault. Файл переменных [values.yaml](kubernetes-vault/vault-helm/values.yaml), добавлены параметры для включения tls.
```bash
git clone https://github.com/hashicorp/vault-helm.git
helm install vault vault-helm

helm status vault
NAME: vault
LAST DEPLOYED: Sat Jun 30 17:27:36 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing HashiCorp Vault!

Now that you have deployed Vault, you should look over the docs on using
Vault with Kubernetes available here:

https://www.vaultproject.io/docs/


Your release is named vault. To learn more about the release, try:

  $ helm status vault
  $ helm get vault
```
- Выполнена инициализация vault
```bash
kubectl exec -it vault-0 -- vault operator init --key-shares=1 --key-threshold=1
Unseal Key 1: J9rYhuKu3yiXF83ftvCukKwnThchbH+7Ice8FhBpm2I=

Initial Root Token: s.Odn2G0LWhZQDqnVgMu202bKo
```
Распечатываем поды vault:
```bash
kubectl exec -it vault-0 -- vault operator unseal 'J9rYhuKu3yiXF83ftvCukKwnThchbH+7Ice8FhBpm2I='
Key                    Value
---                    -----
Seal Type              shamir
Initialized            true
Sealed                 false
Total Shares           1
Threshold              1
Version                1.4.2
Cluster Name           vault-cluster-57c6a60f
Cluster ID             56560c42-f820-6dd9-4192-007d1f9b2a02
HA Enabled             true
HA Cluster             n/a
HA Mode                standby
Active Node Address    <none>

kubectl exec -it vault-1 -- vault operator unseal 'J9rYhuKu3yiXF83ftvCukKwnThchbH+7Ice8FhBpm2I='
Key                    Value
---                    -----
Seal Type              shamir
Initialized            true
Sealed                 false
Total Shares           1
Threshold              1
Version                1.4.2
Cluster Name           vault-cluster-57c6a60f
Cluster ID             56560c42-f820-6dd9-4192-007d1f9b2a02
HA Enabled             true
HA Cluster             https://vault-0.vault-internal:8201
HA Mode                standby
Active Node Address    http://10.244.1.6:8200

kubectl exec -it vault-2 -- vault operator unseal 'J9rYhuKu3yiXF83ftvCukKwnThchbH+7Ice8FhBpm2I='
Key                    Value
---                    -----
Seal Type              shamir
Initialized            true
Sealed                 false
Total Shares           1
Threshold              1
Version                1.4.2
Cluster Name           vault-cluster-57c6a60f
Cluster ID             56560c42-f820-6dd9-4192-007d1f9b2a02
HA Enabled             true
HA Cluster             https://vault-0.vault-internal:8201
HA Mode                standby
Active Node Address    http://10.244.1.6:8200
```
Логинимся в vault root token-ом
```bash
kubectl exec -it vault-0 -- vault login
Token (will be hidden): 
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                s.Odn2G0LWhZQDqnVgMu202bKo
token_accessor       v0R00gxXSmZZVekLY3b7TByC
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```
Получаем список авторизаций
```bash
kubectl exec -it vault-0 -- vault auth list
Path      Type     Accessor               Description
----      ----     --------               -----------
token/    token    auth_token_d4c88ac3    token based credentials
```
- Работа с ключами
Включаем kv engine и создаем секреты:
```bash
kubectl exec -it vault-0 -- vault secrets enable --path=otus kv
kubectl exec -it vault-0 -- vault secrets list --detailed
kubectl exec -it vault-0 -- vault kv put otus/otus-ro/config username='otus' password='asajkjkahs'
kubectl exec -it vault-0 -- vault kv put otus/otus-rw/config username='otus' password='asajkjkahs'
kubectl exec -it vault-0 -- vault read otus/otus-ro/config
kubectl exec -it vault-0 -- vault kv get otus/otus-rw/config
Success! Enabled the kv secrets engine at: otus/
Path          Plugin       Accessor              Default TTL    Max TTL    Force No Cache    Replication    Seal Wrap    External Entropy Access    Options    Description                                                UUID
----          ------       --------              -----------    -------    --------------    -----------    ---------    -----------------------    -------    -----------                                                ----
cubbyhole/    cubbyhole    cubbyhole_f0bd3412    n/a            n/a        false             local          false        false                      map[]      per-token private secret storage                           62aebf2a-6043-7d01-f9a6-fb4208ffbb8b
identity/     identity     identity_8f248057     system         system     false             replicated     false        false                      map[]      identity store                                             1e4cf999-4292-61ea-a837-4554e2c5b818
otus/         kv           kv_d3379188           system         system     false             replicated     false        false                      map[]      n/a                                                        616c42fb-37b8-77c9-f266-72ce31ca7bf4
sys/          system       system_41a9c89c       n/a            n/a        false             replicated     false        false                      map[]      system endpoints used for control, policy and debugging    a8460ecc-ea6d-ec68-352e-4c3896800491
Success! Data written to: otus/otus-ro/config
Success! Data written to: otus/otus-rw/config
Key                 Value
---                 -----
refresh_interval    768h
password            asajkjkahs
username            otus
====== Data ======
Key         Value
---         -----
password    asajkjkahs
username    otus
```

- Работа с k8s
Включем авторизацию через k8s:
```bash
kubectl exec -it vault-0 -- vault auth enable kubernetes
kubectl exec -it vault-0 -- vault auth list
Path           Type          Accessor                    Description
----           ----          --------                    -----------
kubernetes/    kubernetes    auth_kubernetes_276985be    n/a
token/         token         auth_token_d4c88ac3         token based credentials
```
Мы смогли записать otus-rw/config т.к. в политике есть разрешение на create, но нет на update. Файл политики [otus-policy.hcl](kubernetes-vault/otus-policy.hcl)
- Использование vault-agent для получения секретов
```bash
kubectl apply -f kubernetes-vault/configmap.yaml
kubectl apply -f kubernetes-vault/example-k8s-spec.yaml
```
Итоговый файл [index.html](kubernetes-vault/index.html)
Конфиг инит контейнера в kubernetes-vault/example-k8s-spec.yaml переделан для работы по https.
- Использование vault в качестве CA.
```bash
kubectl exec -it vault-0 -- vault write pki_int/issue/example-dot-ru common_name="gitlab.example.ru" ttl="24h"
Key                 Value
---                 -----
ca_chain            [-----BEGIN CERTIFICATE-----
MIIDnDCCAoSgAwIBAgIUJQeRmEvg38wV7HyhHXHaSKXGfwwwDQYJKoZIhvcNAQEL
BQAwFTETMBEGA1UEAxMKZXhtYXBsZS5ydTAeFw0yMDA2MTYwNzM0MDhaFw0yNTA2
MTUwNzM0MzhaMCwxKjAoBgNVBAMTIWV4YW1wbGUucnUgSW50ZXJtZWRpYXRlIEF1
dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMac7l3dm6uQ
0bgHySOU6EiE1l6YtqSBFPDVpmnJDyNnMn8n/tZH9AISit0B+4wM6gagfchEzbii
ZS99h9wqYk55piNkfkO8yUOgxUw9yTcWaC2bdnZZ4OZHgPPtd8tgGalcV7MDgyTi
n+pP0KL2roWKzbEuybnbZ3XYuXo+lsAd4a2JYtTI/3a04aaPuIBx/TIDHwlb56Wy
6BDUblvcyplDod/0B/mRbxqyh0be1WRuUnHggMccMfNUEF73hrxxof7IpHaGQVHc
oMR1IOdQxR9EpJPdOdNtCidSzBLH/CijQylDck5iImh5oKVL4Pu6gRtrNfJmEw4v
AdLn4nt1yxUCAwEAAaOBzDCByTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUw
AwEB/zAdBgNVHQ4EFgQUVt66P7FzKF96sTpZUAlT0cUhCXcwHwYDVR0jBBgwFoAU
jlgrHlD+qyErlnUGNj0NXDGZ/4gwNwYIKwYBBQUHAQEEKzApMCcGCCsGAQUFBzAC
hhtodHRwOi8vdmF1bHQ6ODIwMC92MS9wa2kvY2EwLQYDVR0fBCYwJDAioCCgHoYc
aHR0cDovL3ZhdWx0OjgyMDAvdjEvcGtpL2NybDANBgkqhkiG9w0BAQsFAAOCAQEA
EUakzY/EiLtj5jGqCPE5/0TCXbPssbhXWo97f2oizkQpaI7b8Fc/Rw7h3NdDMF5h
T2JXnEez+j8DttLNtBLgvA0yveMeDRyKk8sJLSG8vKOie4d0awGRDrJ4IZ65WjtK
lN21jIylizvN4K6HPq9ZwZlpzjotB9ex+bs2Ye5uTfVens8VW/GWF8xXxhpDJG+x
10IedMk8KRpbSIwNvinZpe/cirtA/emwIGrac4JH9JI+q3wB4iIsKKGKfJRhHFUa
zARc6uXCeAGbn+oontMASi+zsGWKIIJ07SUQxoK45q1eSlbrooCDiUiLUlENXdyP
1DaklS1sPKg/R6UrRK9TyA==
-----END CERTIFICATE-----]
certificate         -----BEGIN CERTIFICATE-----
MIIDZzCCAk+gAwIBAgIUex8RqbWqW2fIXbA38ptJB9nA26cwDQYJKoZIhvcNAQEL
BQAwLDEqMCgGA1UEAxMhZXhhbXBsZS5ydSBJbnRlcm1lZGlhdGUgQXV0aG9yaXR5
MB4XDTIwMDYxNjA3Mzk1OFoXDTIwMDYxNzA3NDAyN1owHDEaMBgGA1UEAxMRZ2l0
bGFiLmV4YW1wbGUucnUwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQC4
nJN4PHZBPuY24FgQfoVU9+r6iJofhde4nCtbH0cg8w77bgw18T7eQh6Cz0/fYGnK
S2cVcxiWTn3edEw32qVFMBezfPO5cLKqYqYGUs07r70r7ec7Jv9TsVKRXNmL7uDL
vN3cme+BmV7y6FAuafU5m3XdN8KXtyAiDe70jr6Mim7eXKaxx+qzPYGQL4yG3Ad5
ATtlwelxIqw++PJLEt6vbWayUFdFK0pHp7fE/t+7Xjr6kslf0j038EgDyYgSCD2l
3WG6JV7Nb/jJMJmSEOJy4Ln5W69n3uPyjvaz3ssDCp7qZQBSQH5YUbzJUozdrjwY
jiiDeUCeAsXujfQ3GZZ/AgMBAAGjgZAwgY0wDgYDVR0PAQH/BAQDAgOoMB0GA1Ud
JQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjAdBgNVHQ4EFgQUuPKWEsDgBgL+tj+f
9ZdiwlDT554wHwYDVR0jBBgwFoAUVt66P7FzKF96sTpZUAlT0cUhCXcwHAYDVR0R
BBUwE4IRZ2l0bGFiLmV4YW1wbGUucnUwDQYJKoZIhvcNAQELBQADggEBAJwMLvYo
fRhUEEZyNQtvuDp4L84N1qlj4YVRWYigBceWosTm8H2QWzlLsV+4Yl5UsZltPObK
2T8gM6t27ujI9FyyTeNGbqAmaJ31E6SyL1uYr+mlAVFA7eKEPzq6EpGfwqWy7MlI
xcASmV4OJC5TJhDaZ3XUiJ/IgT9E2kPmrw03NnTzK9qG6wAiU1HnSMdaqY1FTxoO
NdTHpIqID4HedCMWcsoVwcVEzPinOle3L75wq1G6to3N/KScAx+QUOTCSJwKjxD9
GSV4aXoaebJ3TJrz6zudL7pMD6vTcxy3BNT1Jg4/+xF7XCv/9xwN86+E9aYoiWK+
fsNHpQNk0FY7s9E=
-----END CERTIFICATE-----
expiration          1592379627
issuing_ca          -----BEGIN CERTIFICATE-----
MIIDnDCCAoSgAwIBAgIUJQeRmEvg38wV7HyhHXHaSKXGfwwwDQYJKoZIhvcNAQEL
BQAwFTETMBEGA1UEAxMKZXhtYXBsZS5ydTAeFw0yMDA2MTYwNzM0MDhaFw0yNTA2
MTUwNzM0MzhaMCwxKjAoBgNVBAMTIWV4YW1wbGUucnUgSW50ZXJtZWRpYXRlIEF1
dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMac7l3dm6uQ
0bgHySOU6EiE1l6YtqSBFPDVpmnJDyNnMn8n/tZH9AISit0B+4wM6gagfchEzbii
ZS99h9wqYk55piNkfkO8yUOgxUw9yTcWaC2bdnZZ4OZHgPPtd8tgGalcV7MDgyTi
n+pP0KL2roWKzbEuybnbZ3XYuXo+lsAd4a2JYtTI/3a04aaPuIBx/TIDHwlb56Wy
6BDUblvcyplDod/0B/mRbxqyh0be1WRuUnHggMccMfNUEF73hrxxof7IpHaGQVHc
oMR1IOdQxR9EpJPdOdNtCidSzBLH/CijQylDck5iImh5oKVL4Pu6gRtrNfJmEw4v
AdLn4nt1yxUCAwEAAaOBzDCByTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUw
AwEB/zAdBgNVHQ4EFgQUVt66P7FzKF96sTpZUAlT0cUhCXcwHwYDVR0jBBgwFoAU
jlgrHlD+qyErlnUGNj0NXDGZ/4gwNwYIKwYBBQUHAQEEKzApMCcGCCsGAQUFBzAC
hhtodHRwOi8vdmF1bHQ6ODIwMC92MS9wa2kvY2EwLQYDVR0fBCYwJDAioCCgHoYc
aHR0cDovL3ZhdWx0OjgyMDAvdjEvcGtpL2NybDANBgkqhkiG9w0BAQsFAAOCAQEA
EUakzY/EiLtj5jGqCPE5/0TCXbPssbhXWo97f2oizkQpaI7b8Fc/Rw7h3NdDMF5h
T2JXnEez+j8DttLNtBLgvA0yveMeDRyKk8sJLSG8vKOie4d0awGRDrJ4IZ65WjtK
lN21jIylizvN4K6HPq9ZwZlpzjotB9ex+bs2Ye5uTfVens8VW/GWF8xXxhpDJG+x
10IedMk8KRpbSIwNvinZpe/cirtA/emwIGrac4JH9JI+q3wB4iIsKKGKfJRhHFUa
zARc6uXCeAGbn+oontMASi+zsGWKIIJ07SUQxoK45q1eSlbrooCDiUiLUlENXdyP
1DaklS1sPKg/R6UrRK9TyA==
-----END CERTIFICATE-----
private_key         -----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAuJyTeDx2QT7mNuBYEH6FVPfq+oiaH4XXuJwrWx9HIPMO+24M
NfE+3kIegs9P32BpyktnFXMYlk593nRMN9qlRTAXs3zzuXCyqmKmBlLNO6+9K+3n
Oyb/U7FSkVzZi+7gy7zd3JnvgZle8uhQLmn1OZt13TfCl7cgIg3u9I6+jIpu3lym
scfqsz2BkC+MhtwHeQE7ZcHpcSKsPvjySxLer21mslBXRStKR6e3xP7fu146+pLJ
X9I9N/BIA8mIEgg9pd1huiVezW/4yTCZkhDicuC5+VuvZ97j8o72s97LAwqe6mUA
UkB+WFG8yVKM3a48GI4og3lAngLF7o30NxmWfwIDAQABAoIBADCOFgdYt62fcoNa
bC8iZ8UaU7ZDOW4zELLgeFLGHjofU4BzyEhjxCpG76luB070F776KAmvNPdLe7WH
lwhVvIQ/CuzNX3kVmBhSS+J74rjhFvs33kpjjmIf0FylNB6m3H8ZlKzR2/mVMjDn
QzeB7NqS9eQSJ18p7gym54NxC9MAn5dz3p0GSe8pWq4hXy7/xRU3T5GjLwUSU+T3
DZP8+7l/FEvgMsV6G0deQVam9iQr7cR6HQ8OOUNMjUaM7w+GUNV4iLUii3JDSQd7
R1QnrEwFi+VyEANli/JOdLmn03knN/MAg+DOJK70w6SgaKMcbW/nH/u0sG3aq6jB
j5F1hNkCgYEA1Hv+XxoWaVPb3XoNRCungJfrU3IFta7s52tFOV0UxRqsxfXFnwOx
NrKaRQlzFUco0RqgjF7pLxbXYJDJ1+gNoWc3K6vZWeSyqWWiBxtn9eqrChxgPSE/
ZN9LwheH4nxi5nsh0EvcqfdPSXdYzI5SbRDyhEWFauvBVsyCll2N6i0CgYEA3mtJ
j3oJLQ1pAoG/xCPpqGotKsksdPvcBSVImnQFgBiBZGEqmrSf3O997Mz6QuopDsw1
wueVesMiA7m4RZDlaVnSkhyvQAWlIIj2xmTLC+7pe9ZrGF8LvvCm49/zkBlOXuPX
izH6xphTZgl0IDrY7T+ICfbcpi1rfJjikGAOitsCgYAKUD5nhU+jKyPX2y27qlbG
Ahm1AirOx7/N98HzZ9YzPvk13pkJ/9bhLcgZI71HQh30EFPMnGq7E2O+1yhE54mJ
1QWzg/LXzybw2/MCX00rfYlxwzDUpsF59vCpahT5ZEo0n7NjddsvEMbzbOyNeTb8
/j6XNvyj1O+cc+6+t6nEvQKBgCrX38OTblEPVDr3Y0kU4d1fFnQ3bCjcmvUiyWl3
D9gs4D/Ft781K9YTC96hXVOmZ2JCU9jHYzPSgqrVC3na/1Xbx4P9ooRikfxCZcax
g6s4yiDgnKCFLm4JTRx39yK6vS3qFYrqhbPbg7UT/Rp4O3D32+yPcNFRznKhwIKu
/h4hAoGBAJJr7C+DA81dsmpnSmLPbzrQE1Gc/5woO+LVZoeIe7e70iQCMv2muno7
9LPomPfR+WakYEP6JSj0d+38Laj6UNH3yh6dVi0hCJlTVy6Vn3Rg246xvQW8Jm5+
eOqgnSmTzYSud1a0yuOg+wWYjps6XIxqXTQaApI5YjXxbg2Dm0C9
-----END RSA PRIVATE KEY-----
private_key_type    rsa
serial_number       7b:1f:11:a9:b5:aa:5b:67:c8:5d:b0:37:f2:9b:49:07:d9:c0:db:a7


