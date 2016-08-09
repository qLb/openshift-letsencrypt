# Automatic Certificates for Openshift Routes

It will manage all `route`s with (by default) `butter.sh/letsencrypt-managed=yes` labels in the project/namespace, it's deployed in.
Certificates will be stored in secrets that belong to the letsencrypt service account.


## Limitations
For now, there are the following limitations.

 * It only implements `http-01`-type verification, better known as "Well-Known".
 * Multiple domains per certificate are not supported. See issue #1.
 * It will not create the letsencrypt account.
   It needs to be created before deploying.
   See Section **Installation** below.
 * It can't work cross-namespace.


## Customizing

The following env variables can be used.

 * `LETSENCRYPT_ROUTE_SELECTOR` (*optional*, defaults to `butter.sh/letsencrypt-managed=yes`), to filter the routes to use;
 * `LETSENCRYPT_RENEW_BEFORE_DAYS` (*optional*, defaults to `7`), renew this number of days before the certificate is about to expire;
 * `LETSENCRYPT_CONTACT_EMAIL` (*required for account generation*), the email that will be used by the ACME CA;
 * `LETSENCRYPT_CA` (*optional*, defaults to `https://acme-v01.api.letsencrypt.org/directory`);
 * `LETSENCRYPT_KEYTYPE` (*optional*, defaults to `rsa`), the key algorithm to use;
 * `LETSENCRYPT_KEYSIZE` (*optional*, defaults to `4096`), the size in bit for the private keys (if applicable);



## Implementation Details

### Secrets

Certificates are stored in secrets named `letsencrypt-<hostname>`, the ACME key is stored in `letsencrypt-creds`.

They use the following labels.

 * `butter.sh/letsencrypt-domainname`, the domain name
 * `butter.sh/letsencrypt-crt-enddate-secs`, the certificates `notAfter` date in seconds since the epoch.


### Containers

The pod consists of three containers, each doing exactly one thing.
They share the filesystem `/var/www/acme-challenge` to store the challenges.

 * **Watcher Container**
   watches routes and either generates a new certificate or set the already generated certificate.

 * **Cron container**
   periodically checks whether the certificates need to be regenerated.
   When Kubernetes cron jobs are implemented, this will move outside the pod.

 * **Webserver Container**
   serves `.well-known/acme-challenge` when asking to sign the certificate.
   Uses `ibotty/s2i-nginx` on dockerhub.


## Installing Openshift-Letsencrypt

### Service Account

The "letsencrypt" service account needs to be able to manage its secrets and manage routes.

```
> oc create -f policybinding.yaml
> oc create -f letsencrypt-role.yaml
> oc policy add-role-to-user letsencrypt --role-namespace <mynamespace> -z letsencrypt
```

To manage routes in all namespaces, additionally do the following.

```
> oc create -f letsencrypt-clusterrole.yaml
> oadm policy add-cluster-role-to-user letsencrypt -z letsencrypt
```


## Notes

### HPKP

It is necessary to pin _at least_ one key to use for disaster recovery, outside the cluster!

Maybe pre-generate `n` keys and pin all of them.
On key rollover, delete the previous key, use the oldest of the remaining keys to sign the certificate, generate a new key and pin the new keys.
That way, the pin can stay valid for `(n-1)* lifetime of a key`.
That is, if no key gets compromised!

### Locking

Should be done with `flock` around `/var/lib/letsencrypt-container/$DOMAINNAME`, whenever a certificate is to be touched.
