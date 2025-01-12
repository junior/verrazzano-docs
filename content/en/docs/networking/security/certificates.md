---
title: "Customize Certificates"
description: "Customize SSL certificate generation for Verrazzano system endpoints"
weight: 2
draft: false
aliases:
  - /docs/customize/certificates
---

Verrazzano issues certificates to secure access from external clients to secure system endpoints.  
A certificate from a certificate authority (CA) must be configured to issue the endpoint certificates in one of the
following ways:

* Let Verrazzano generate a self-signed CA (the default).
* Configure a CA that you provide.
* Configure [LetsEncrypt](https://letsencrypt.org/) as the certificate issuer (requires [Oracle Cloud Infrastructure DNS](https://docs.cloud.oracle.com/en-us/iaas/Content/DNS/Concepts/dnszonemanagement.htm)).

In all cases, Verrazzano uses [cert-manager](https://cert-manager.io/) to manage the creation of certificates.

{{< alert title="NOTE" color="danger" >}}
Self-signed certificate authorities generate certificates that are NOT signed by a trusted authority; typically, they are not used in production environments.
{{< /alert >}}

## Use the Verrazzano self-signed CA

By default, Verrazzano creates its own self-signed CA.  No configuration is required.

## Use a custom CA

If you want to provide your own CA, you must:

* (Optional) Create your own signing key pair and CA certificate.

  For example, you can use the `openssl` CLI to create a key pair for the `nip.io` domain.
{{< clipboard >}}
<div class="highlight">

  ```
  # Generate a CA private key
  $ openssl genrsa -out tls.key 2048

  # Create a self-signed certificate, valid for 10yrs with the 'signing' option set
  $ openssl req -x509 -new -nodes -key tls.key -subj "/CN=*.nip.io" -days 3650 -reqexts v3_req -extensions v3_ca -out tls.crt
  ```

</div>
{{< /clipboard >}}

  The output of these commands will be two files, `tls.key` and `tls.crt`, the key and certificate for your signing key pair.
  These files must be named in that manner for the next step.

  If you already have generated your own key pair, you must name the private key and certificate, `tls.key` and `tls.crt`,
  respectively.  If your issuer represents an intermediate, ensure that `tls.crt` contains the issuer’s full chain in the
  correct order.

  You can find more details on providing your own CA, in the cert-manager [CA](https://cert-manager.io/docs/configuration/ca/) documentation.

* Save your signing key pair as a Kubernetes secret.  The secret must be created in the namespace corresponding to
  the `clusterResourceNamespace` used by cert-manager (`cert-manager` by default).  
  For more information, see the [cert-manager](https://cert-manager.io/docs/configuration/)
  documentation.
{{< clipboard >}}
<div class="highlight">

   ```
   $ kubectl create ns mynamespace
   $ kubectl create secret tls myca --namespace=mynamespace --cert=tls.crt --key=tls.key
   ```

</div>
{{< /clipboard >}}

* Specify the secret name and namespace location in the Verrazzano custom resource.

  The custom CA secret must be provided to cert-manager using the following fields in
  [`spec.components.clusterIssuer.ca`](/docs/reference/vpo-verrazzano-v1beta1#install.verrazzano.io/v1beta1.CAIssuer) in the Verrazzano custom resource:

  * `spec.components.clusterIssuer.ca.secretName`
  * (If needed) `spec.components.clusterIssuer.clusterResourceNamespace`

   For example, if you created a CA secret named `myca` in the namespace `mynamespace`, you would configure it as shown:
{{< clipboard >}}
<div class="highlight">

```
apiVersion: install.verrazzano.io/v1beta1
kind: Verrazzano
metadata:
  name: custom-ca-example
spec:
  profile: dev
  components:
    clusterIssuer:
      clusterResourceNamespace: mynamespace
      ca:
        secretName: myca
```

</div>
{{< /clipboard >}}

   In this example, `mynamespace` will be configured as the `clusterResourceNamespace` for the Verrazzano cert-manager
   instance when it is installed.

## Use LetsEncrypt certificates

You can configure Verrazzano to use certificates generated by [LetsEncrypt](https://letsencrypt.org/).  LetsEncrypt
implements the [ACME protocol](https://tools.ietf.org/html/rfc8555), which provides a standard protocol for the
automated issuance of certificates signed by a trusted authority.  This is managed through the
[`spec.components.clusterIssuer.letsEncrypt`](/docs/reference/vpo-verrazzano-v1beta1#install.verrazzano.io/v1beta1.LetsEncryptACMEIssuer)
field in the Verrazzano custom resource.

{{< alert title="NOTE" color="primary" >}}
Using LetsEncrypt for certificates also requires using Oracle Cloud Infrastructure DNS for DNS management.
For details, see the [Customize DNS]({{< relref "/docs/networking/traffic/dns.md" >}}) page.
{{< /alert >}}

To configure cert-manager to use LetsEncrypt as the certificates provider, you must configure the Verrazzano Let's Encrypt
issuer with the following values in the Verrazzano custom resource:

* Set the `spec.components.clusterIssuer.letsEncrypt.emailAddress` field to a valid email address for the `letsEncrypt` account.
* (Optional) Set the `spec.components.clusterIssuer.letsEncrypt.environment` field to either `staging` or `production` (the default).

The following example configures Verrazzano to use the LetsEncrypt `production` environment by default, with Oracle Cloud Infrastructure DNS
for DNS record management.
{{< clipboard >}}
<div class="highlight">

```
apiVersion: install.verrazzano.io/v1beta1
kind: Verrazzano
metadata:
  name: letsencrypt-certs-example
spec:
  profile: dev
  components:
    clusterIssuer:
      letsEncrypt:
        emailAddress: jane.doe@mycompany.com
    dns:
      oci:
        ociConfigSecret: oci
        dnsZoneCompartmentOCID: ocid1.compartment.oc1.....
        dnsZoneOCID: ocid1.dns-zone.oc1.....
        dnsZoneName: example.com
```

</div>
{{< /clipboard >}}

The following example configures Verrazzano to use the LetsEncrypt `staging` environment with Oracle Cloud Infrastructure DNS.
{{< clipboard >}}
<div class="highlight">

```
apiVersion: install.verrazzano.io/v1beta1
kind: Verrazzano
metadata:
  name: letsencrypt-certs-example
spec:
  profile: dev
  components:
    clusterIssuer:
      letsEncrypt:
        emailAddress: jane.doe@mycompany.com
        environment: staging
    dns:
      oci:
        ociConfigSecret: oci
        dnsZoneCompartmentOCID: ocid1.compartment.oc1.....
        dnsZoneOCID: ocid1.dns-zone.oc1.....
        dnsZoneName: example.com
```

</div>
{{< /clipboard >}}

{{< alert title="NOTE" color="danger" >}}
Certificates issued by the LetsEncrypt `staging` environment are signed by untrusted authorities, similar to
self-signed certificates.  They are typically not used in production environments.
{{< /alert >}}

### LetsEncrypt staging versus production

LetsEncrypt provides rate limits on generated certificates to ensure fair usage across all clients.  The
`production` environment limits can be exceeded more frequently in environments where Verrazzano may be
installed or reinstalled frequently (like a test environment).  This can result in failed installations due to
rate limit exceptions on certificate generation.

In such environments, it is better to use the LetsEncrypt `staging` environment, which has much higher limits
than the `production` environment.  For test environments, the self-signed CA also may be more appropriate to completely
avoid LetsEncrypt rate limits.

## Use your own cert-manager

You can either use the Verrazzano-provided cert-manager or use your own. To use your own cert-manager, disable the
`certManager` component in the Verrazzano CR as shown:

{{< clipboard >}}
<div class="highlight">

```
apiVersion: install.verrazzano.io/v1beta1
kind: Verrazzano
metadata:
  name: custom-cm-example
spec:
  profile: dev
  components:
    certManager:
      enabled: false
```

</div>
{{< /clipboard >}}

The Verrazzano certificates issuer is configured using the [`spec.components.clusterIssuer`](/docs/reference/vpo-verrazzano-v1beta1#install.verrazzano.io/v1beta1.ClusterIssuerComponent)
component.  The Verrazzano default `clusterIssuer` assumes that cert-manager is installed in the `cert-manager` namespace, and that the `clusterResourceNamespace`
for cert-manager is `cert-manager`.

If your cert-manager is installed into a different namespace or uses a separate namespace for the `clusterResourceNamespace`,
you can configure this on the Verrazzano `clusterIssuer` component.

In the following example, the `clusterIssuer` is configured to use the namespace `my-cm-resources` as the `clusterResourceNamespace`:

{{< clipboard >}}
<div class="highlight">

```
apiVersion: install.verrazzano.io/v1beta1
kind: Verrazzano
metadata:
  name: custom-cmres-example
spec:
  profile: dev
  components:
    certManager:
      enabled: false
    clusterIssuer:
      clusterResourceNamespace: my-cm-resources
```

</div>
{{< /clipboard >}}

### Use Let's Encrypt with OCI DNS with your own cert-manager

Verrazzano uses a webhook component to support Let's Encrypt certificates with OCI DNS.  This webhook implements the
[cert-manager solver webhook pattern](https://cert-manager.io/docs/configuration/acme/dns01/webhook/) to support
solving `DNS01` challenges for OCI DNS.

In most circumstances, you will not need to configure this because it will automatically deploy when OCI DNS and Let's Encrypt
certificates are in use.  However, if you are using your own cert-manager installed in a namespace other than `cert-manager`
and/or the `clusterResourceNamespace` is something other than `cert-manager`, you will need to configure this for the webhook.

The following example configures the webhook to use the namespace `my-cm` for the cert-manager installation namespace and
`my-cm-resources` for the `clusterResourceNamespace`:

{{< clipboard >}}
<div class="highlight">

```
apiVersion: install.verrazzano.io/v1beta1
kind: Verrazzano
metadata:
  name: custom-cm-letsencrypt-example
spec:
  profile: dev
  components:
    certManager:
      enabled: false
    clusterIssuer:
      clusterResourceNamespace: my-cm-resources
      letsEncrypt:
        emailAddress: jane.doe@mycompany.com
        environment: staging
    dns:
      oci:
        ociConfigSecret: oci
        dnsZoneCompartmentOCID: ocid1.compartment.oc1.....
        dnsZoneOCID: ocid1.dns-zone.oc1.....
        dnsZoneName: example.com
    certManagerWebhookOCI:
      overrides:
      - values:
          certManager:
            namespace: my-cm
            clusterResourceNamespace: my-cm-resources
```

</div>
{{< /clipboard >}}
