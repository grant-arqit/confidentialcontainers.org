---
title: Sealed Secrets
date: 2023-01-24
description: Create, sign, and deploy protected Kubernetes secrets for confidential workloads
weight: 24
categories:
  - feature
tags:
  - secrets 
---

{{% alert title="Note" color="primary" %}}
Sealed Secrets depend on attestation.
[Configure attestation](../attestation) before using sealed secrets.
{{% /alert %}}

In Confidential Containers, secrets can be protected with sealing.
A sealed secret is a way to encapsulate confidential data such that it can be accessed only inside an enclave in
conjunction with attestation.
At the same time, the untrusted control plane can still store and orchestrate it like a normal Kubernetes secret.
Once unsealed inside the guest, the secret can be transparently provisioned to your workload as an environment variable
or a volume.

## Basic Usage

Here's how you create a vault secret.
There are also envelope secrets, which are described later.
Vault secrets are a pointer to resource stored in a KBS,
while envelope secrets are wrapped secrets that are unwrapped with a KMS.

### Creating a sealed secret

1. Pull a prebuilt `secret` binary with [ORAS](https://oras.land/) from the
   [`secret-cli` GitHub package](https://github.com/confidential-containers/guest-components/pkgs/container/guest-components%2Fsecret-cli):

   ```bash
   oras pull ghcr.io/confidential-containers/guest-components/secret-cli:latest
   chmod +x secret
   ./secret --help
   ```

2. Generate a P-256 signing keypair in JWK format.

   ```bash
   SIGNING_KEY_KID=my-signing-key
   ./secret keygen --kid ${SIGNING_KEY_KID} --output-dir ./keys
   ```

   This writes two files: `keys/my-signing-key-private.json` (used for signing)
   and `keys/my-signing-key-public.json` (uploaded to the KBS in a later step).
   These keys are used for signing and verifying the sealed secret.
   This is separate from the resource that the sealed secret will point to.
   Only the public signing key will be provisioned to Trustee.

3. Create a vault sealed secret.

   ```bash
   SIGNING_KID_KBS_URI=default/test_signing/jwk_public
   SIGNING_JWK_PATH=./keys/${SIGNING_KEY_KID}-private.json
   SIGNING_RESOURCE_URI=default/test_secrets/your_secret

   export POINTER_TO_SECRET=$(./secret seal \
       --signing-kid kbs:///${SIGNING_KID_KBS_URI} --signing-jwk-path ${SIGNING_JWK_PATH} \
       vault \
       --resource-uri kbs:///${SIGNING_RESOURCE_URI} --provider kbs | grep -v '^Warning:')
   echo ${POINTER_TO_SECRET}
   ```

   Key options:

    - `--signing-kid`: the identifier embedded in the JWS header.
      When this is a Trustee resource URI, the guest can fetch the public key during verification.
    - `--signing-jwk-path`: path to a JWK containing the private signing key.
    - `vault`: selects the vault secret format.
    - `--resource-uri`: provider-specific location of the actual secret.
    - `--provider`: the provider that will resolve the secret, such as `kbs`.

   This command should return a base64 string (
   see [Integrity protection and signing](#integrity-protection-and-signing)) which you will use in the next step.

{{% alert title="Note" color="primary" %}}
For vault secrets, the secret-cli tool does not upload your resource to the KBS automatically.
In addition to generating the secret string, you must also upload the public key (JWK) and resource to your KBS.
{{% /alert %}}

### Adding a sealed secret to Kubernetes

Create a secret from your secret string using `kubectl`.

```bash
kubectl create secret generic sealed-secret --from-literal=secret=${POINTER_TO_SECRET}
```

When using `--from-literal` you provide a mapping of secret keys and values.
The secret value should be the string generated in the previous step.
The secret key can be whatever you want, but make sure to use the same one in future steps.
This is separate from the name of the secret.

### Configuring the KBS for your workload

#### Upload the secret value to the KBS

Create file with your secret value (which will be returned when the sealed secret is unsealed).

```bash
cat >> my_secret << 'END'
my-secret-value
END
```

Upload the secret to the KBS at the resource URI specified in the `--resource-uri` parameter when you created the sealed
secret.

{{< tabpane text=true right=true persist=header >}}

{{% tab header="kbs-client tool" %}}

Run the following command to add the secret to KBS storage:

```bash
./kbs-client --url <SCHEME>://<HOST>:<PORT> config \
  --admin-token-file admin-token \
  set-resource \
  --resource-file ./my_secret \
  --path ${SIGNING_RESOURCE_URI}
```

{{% /tab %}}

{{% tab header="Trustee Operator" %}}

Export the `KbsConfig` custom resource name:

```bash
export CR_NAME=$(kubectl get kbsconfig -n trustee-operator-system -o=jsonpath='{.items[0].metadata.name}')
```

Create a Secret containing your secret value:

```bash
kubectl create secret generic test_secrets \
  -n trustee-operator-system \
  --from-file=your_secret=./my_secret
```

Patch the `KbsConfig` custom resource to add the public key Secret:

```bash
kubectl patch KbsConfig -n trustee-operator-system $CR_NAME \
  --type=json \
  -p='[{"op":"add", "path":"/spec/kbsSecretResources/-", "value":"test_secrets"}]'
```

{{% /tab %}}

{{< /tabpane >}}

#### Upload the JWK public key to the KBS

This is needed for the guest to verify the signature of the sealed secret.

Upload the public key generated by `secret keygen` to the KBS at the resource URI specified in the `--signing-kid`
parameter when you created the sealed secret. Only the public component should be provisioned to Trustee.

{{< tabpane text=true right=true persist=header >}}

{{% tab header="kbs-client tool" %}}

Run the following command to add the public key to KBS storage:

```bash
./kbs-client --url <SCHEME>://<HOST>:<PORT> config \
  --admin-token-file admin-token \
  set-resource \
  --resource-file ./keys/${SIGNING_KEY_KID}-public.json \
  --path ${SIGNING_KID_KBS_URI}
```

{{% /tab %}}

{{% tab header="Trustee Operator" %}}

Export the `KbsConfig` custom resource name:

```bash
export CR_NAME=$(kubectl get kbsconfig -n trustee-operator-system -o=jsonpath='{.items[0].metadata.name}')
```

Create a Secret containing the JWK public key:

```bash
kubectl create secret generic test_signing \
  -n trustee-operator-system \
  --from-file=jwk_public=./keys/${SIGNING_KEY_KID}-public.json
```

Patch the `KbsConfig` custom resource to add the JWK public key secret:

```bash
kubectl patch KbsConfig -n trustee-operator-system $CR_NAME \
  --type=json \
  -p='[{"op":"add", "path":"/spec/kbsSecretResources/-", "value":"test_signing"}]'
```

{{% /tab %}}

{{< /tabpane >}}

### Deploying a sealed secret to a confidential workload

In order for your workload to use the sealed secret, the CDH inside the guest needs to know how to retrieve the
plaintext secret value from the KBS.

Perform below steps to configure the CDH and reference the sealed secret from your workload.

1. Choose either of the following approaches to provide KBS configuration to the guest:

   {{< tabpane text=true right=true persist=header >}}

   {{% tab header="kernel params" %}}

   Set the following kernel parameters in the `io.katacontainers.config.hypervisor.kernel_params` pod annotation:

   ```text
   agent.aa_kbc_params=cc_kbc::<SCHEME>://<HOST>:<PORT>
   ```

   - `agent.aa_kbc_params` points to the KBS service. Replace `<SCHEME>`, `<HOST>`, and `<PORT>` with the values for
      your environment.

   {{% /tab %}}

   {{% tab header="init-data" %}}

   Run the following commands to prepare the init data file with the appropriate KBS configuration:

   - Export environment variables:

      ```bash
      export KBS_ADDRESS=scheme://host:port
      ```

   - Create file `$HOME/initdata.toml`
       ```bash
       cat <<EOF> initdata.toml
       algorithm = "sha256"
       version = "0.1.0"
       
       [data]
       "aa.toml" = '''
       [token_configs]
       [token_configs.coco_as]
       url = '${KBS_ADDRESS}'
       
       [token_configs.kbs]
       url = '${KBS_ADDRESS}'
       '''
       
       "cdh.toml" = '''
       socket = 'unix:///run/confidential-containers/cdh.sock'
       credentials = []
       
       [kbc]
       name = 'cc_kbc'
       url = '${KBS_ADDRESS}'
       '''
       EOF
       ```

   - Encode the init data file in base64:

      ```bash
      export INIT_DATA=$(cat $HOME/initdata.toml | gzip | base64 -w 0)
      ```

   Set the output from above command in the `io.katacontainers.config.hypervisor.cc_init_data` pod annotation:

   ```text
   io.katacontainers.config.hypervisor.cc_init_data = ${INIT_DATA}
   ```

   {{% /tab %}}

   {{< /tabpane >}}

2. Reference the sealed secret from your workload. You can reference the secret as either an environment variable or a
   volume mount.

   {{< tabpane text=true right=true persist=header >}}

   {{% tab header="environment variable" %}}

   Expose your sealed secret as an environment variable.

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: sealed-secret-pod
     annotations:
     #  io.katacontainers.config.hypervisor.cc_init_data: VALUE # Uncomment if init-data approach 
     #  io.katacontainers.config.hypervisor.kernel_params: VALUE # Uncomment if kernel params approach
   spec:
     runtimeClassName: kata-qemu-coco-dev # use your CoCo runtime class
     containers:
       - name: busybox
         image: quay.io/prometheus/busybox:latest
         imagePullPolicy: Always
         command: ["/bin/sh", "-c", "echo $PROTECTED_SECRET"]
         env:
           - name: PROTECTED_SECRET
             valueFrom:
               secretKeyRef:
                 name: sealed-secret
                 key: secret
   ```

   {{% /tab %}}

   {{% tab header="volume" %}}

   Expose your sealed secret as a volume mount.

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: sealed-secret-pod-volume
     annotations:
     #  io.katacontainers.config.hypervisor.cc_init_data: VALUE # Uncomment if init-data approach 
     #  io.katacontainers.config.hypervisor.kernel_params: VALUE # Uncomment if kernel params approach
   spec:
     runtimeClassName: kata-qemu-coco-dev # use your CoCo runtime class
     containers:
       - name: busybox
         image: quay.io/prometheus/busybox:latest
         imagePullPolicy: Always
         command: ["/bin/sh", "-c", "cat /sealed/secret-value/secret"]
         volumeMounts:
           - name: sealed-secret-volume
             mountPath: "/sealed/secret-value"
     volumes:
       - name: sealed-secret-volume
         secret:
           secretName: sealed-secret
   ```
   {{% alert title="Note" color="primary" %}}
   Currently sealed secret volumes must be mounted in the `/sealed` directory.
   {{% /alert %}}

   {{% /tab %}}

   {{< /tabpane >}}

## Sealed secret formats

Sealed secrets are defined in two main formats: vault secrets and envelope secrets.

### Vault Secrets

A vault secret is a pointer to a secret stored elsewhere, either in a KMS or a KBS.
To fulfill a vault secret, the CDH retrieves the secret value from the selected provider during unsealing.

Creating a vault secret does not require encrypting the secret value itself.
Instead, the sealed secret stores metadata that identifies where the plaintext secret can be retrieved.

The format of a vault sealed secret is:

```json
{
  "version": "0.1.0",
  "type": "vault",
  "provider": "xxx",
  "name": "xxx",
  "provider_settings": {
    "...": "..."
  },
  "annotations": {
    "...": "..."
  }
}
```

Field descriptions:

- `version`: **REQUIRED** format version. The current version is `0.1.0`.
- `type`: **REQUIRED** and must be `vault`.
- `provider`: **REQUIRED** provider of the secret value.
- `name`: **REQUIRED** identifier of the secret value used by the provider.
- `provider_settings`: **REQUIRED** provider-specific configuration used to create the vault client.
- `annotations`: optional provider-specific metadata used to retrieve the plaintext secret.

### Envelope Secrets

You can also create envelope secrets.
With envelope secrets, the secret value itself is included in the sealed secret, unlike a vault secret,
which only contains a pointer to a secret stored elsewhere.

An envelope secret uses envelope encryption: a data encryption key encrypts the plaintext secret, and a KMS or KBS
unwraps that key during unsealing.
This allows the key used for unwrapping to remain in the KMS, while the sealed secret can still travel with the
workload.
It also decouples the protected secret material from Trustee/KBS resource storage.

The envelope model can be summarized as:

`$$Sealed\ Secret := \{Enc_{Sealing\ key}(Encryption\ Key),\ Enc_{Encryption\ Key}(secret\ value)\}$$`

The format of an envelope sealed secret is:

```json
{
  "version": "0.1.0",
  "type": "envelope",
  "provider": "xxx",
  "key_id": "xxx",
  "encrypted_key": "ab27dc=",
  "encrypted_data": "xxx",
  "wrap_type": "A256GCM",
  "iv": "xxx",
  "provider_settings": {
    "...": "..."
  },
  "annotations": {
    "...": "..."
  }
}
```

Field descriptions:

- `version`: **REQUIRED** format version. The current version is `0.1.0`.
- `type`: **REQUIRED** and must be `envelope`.
- `provider`: **REQUIRED** provider of the sealing key.
- `key_id`: **REQUIRED** identifier of the sealing key used to unwrap the data encryption key.
- `encrypted_key`: **REQUIRED** wrapped data encryption key, base64 encoded.
- `encrypted_data`: **REQUIRED** encrypted secret value, base64 encoded.
- `wrap_type`: **REQUIRED** algorithm used by the data encryption key to encrypt the secret value. `A256GCM` is
  preferred.
- `iv`: **REQUIRED** initialization vector used during secret encryption, base64 encoded.
- `provider_settings`: **REQUIRED** provider-specific configuration used to create the KMS client.
- `annotations`: optional provider-specific metadata used during unsealing.

## Supported providers

| Provider | Usage            | Notes                                                                                                                                                 |
|----------|------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| `kbs`    | Vault secrets    | Built into the CoCo/Trustee flow                                                                                                                      |
| `aliyun` | Envelope secrets | See the [Aliyun KMS guide](https://github.com/confidential-containers/guest-components/blob/main/confidential-data-hub/docs/kms-providers/alibaba.md) |
| `ehsm`   | Envelope secrets | See the [eHSM guide](https://github.com/confidential-containers/guest-components/blob/main/confidential-data-hub/docs/kms-providers/ehsm-kms.md)      |

## Integrity protection and signing

Sealed secrets support integrity protection with [JWS](https://datatracker.ietf.org/doc/html/rfc7515).
In practice the CLI emits a string that starts with `sealed.` followed by the protected header, payload,
and signature as dot-separated base64url sections.

```text
sealed.<base64url(JWS protected header)>.<base64url(JWS payload)>.<base64url(JWS signature)>
```

The JWS `kid` identifies the public key used to verify the signature. That public key can be:

- provisioned directly as a CDH credential, or
- stored in Trustee and referenced through a resource URI.

Using a resource URI for `kid` is convenient because the guest can fetch the verification key from Trustee during
unsealing.
