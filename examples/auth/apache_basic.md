# [RequestHeaderIdentityProvider](http://docs.openshift.org/latest/admin_guide/configuring_authentication.html#RequestHeaderIdentityProvider) configuration example

This example configures an authentication proxy on the same host as the Master.
The traffic between the Proxy and the Master is excrypted and authentication is
handled by X509 client certificates.  This is merely a convenience and may not
be suitable for your environment.  For example, if you were already running the
router on the Master then port 443 would not be available.

This configuration is especially useful for porting authentication integrations
from previous versions of OpenShift.  Apache is not strictly required as this
example is meant to serve as a reference configuration for other Proxies.

## Install Apache

    yum install -y httpd mod_ssl

## Generate a client certificate for the proxy:

    osadm create-api-client-config --certificate-authority='/etc/openshift/master/ca.crt' \
                                   --client-dir='/etc/openshift/master/authproxy' \
                                   --signer-cert='/etc/openshift/master/ca.crt' \
                                   --signer-key='/etc/openshift/master/ca.key' \
                                   --signer-serial='/etc/openshift/master/ca.serial.txt' \
                                   --user='system:authproxy'

Note: the `--user` can be anything however in general it should be something
that will by default have no API access.

## Copy certificates to a location SELinux will allow Apache to read

    pushd /etc/openshift/master
      cp master.server.crt /etc/pki/tls/certs/localhost.crt
      cp master.server.key /etc/pki/tls/private/localhost.key
      cp ca.crt /etc/pki/CA/certs/ca.crt
      cat authproxy/system\:authproxy.crt \
          authproxy/system\:authproxy.key > \
          /etc/pki/tls/certs/authproxy.pem
    popd

Note: If you are running the authentication proxy on a different hostname that
the Master it will be important to generate a certificate that matches and not
use the default Master certificate as shown above.  The value for
`masterPublicURL` in `/etc/openshift/master/master-config.yaml` must be
included in the `X509v3 Subject Alternative Name` in the certificate that
Apache is going to use.  For convenience you could use `osadm
create-server-cert` to create a new server certificate if the default aren't
suitable for your environment.

## Set up Apache
Unlike OpenShift Enterprise version 2 this proxy does not need to reside on the
same host as the Master.  It uses a client certificate to connect to the Master
which is configured to trust the `X-Remote-User` header.

Note: This example uses file-backed Basic authentication.

~~~
# Nothing needs to be served over HTTP.  This virtual host simply redirects to
# HTTPS.
<VirtualHost *:80>
  DocumentRoot /var/www/html
  RewriteEngine              On
  RewriteRule     ^(.*)$     https://%{HTTP_HOST}$1 [R,L]
</VirtualHost>

<VirtualHost *:443>
  ServerName ose3-master.example.com
  DocumentRoot /var/www/html
  SSLEngine on
  SSLCertificateFile /etc/pki/tls/certs/localhost.crt
  SSLCertificateKeyFile /etc/pki/tls/private/localhost.key
  SSLCACertificateFile /etc/pki/CA/certs/ca.crt

  SSLProxyEngine on
  SSLProxyCACertificateFile /etc/pki/CA/certs/ca.crt
  SSLProxyMachineCertificateFile /etc/pki/tls/certs/authproxy.pem

  # Insert your backend server name/ip here.
  ProxyPass / https://ose3-master.example.com:8443/
  ProxyPassReverse / https://ose3-master.example.com:8443/

  # Requests should be able to access /oauth/token/request and
  # /oauth/token/display without authentication.  In the case of
  # /outh/token/display OpenShift will check one of the
  # ORIGIN_AUTH_REQUEST_HANDLERS to see if the request is authenticated.
  # Technically it would require authentication for /oauth/token/display simply
  # by modifying these two ProxyMatch stanzas.
  <ProxyMatch /oauth/token/.*>
    Allow from all
  </ProxyMatch>

  # /oauth/authorize and /oauth/approve should be protected by Apache.
  <ProxyMatch /oauth/a.*>
    AuthUserFile /etc/openshift/htpasswd
    AuthType basic

    # For ldap:
    # AuthBasicProvider ldap
    # AuthLDAPURL "ldap://ldap.example.com:389/ou=People,dc=my-domain,dc=com?uid?sub?(objectClass=*)"

    # For Kerberos remove "AuthType basic" and insert the following:
    # AuthType Kerberos
    # KrbMethodNegotiate on
    # KrbMethodK5Passwd off
    # KrbServiceName Any
    # KrbAuthRealms EXAMPLE.COM
    # Krb5Keytab /path/to/keytab
    # KrbSaveCredentials off

    AuthName openshift
    Require valid-user
    RequestHeader set X-Remote-User %{REMOTE_USER}s
  </ProxyMatch>

  # All other requests should use Bearer tokens.  These can only be verified by
  # OpenShift so we need to let these requests pass through.
  <Proxy *>
    SetEnvIfNoCase Authorization Bearer passthrough
    Allow from env=passthrough

    Order Deny,Allow
    Deny from all
    Satisfy any
  </Proxy>
</VirtualHost>

RequestHeader unset X-Remote-User
~~~

If you are using the default htpasswd implementation you can run the following:

yum -y install httpd-tools
touch /etc/openshift/htpasswd
htpasswd -b /etc/openshift/htpasswd joe redhat

## OpenShift Master configuration

All instances of `masterPublicURL` and `assetPublicURL` need to match the
hostname and port for the Apache VirtualHost.

    masterPublicURL: https://ose3-master.example.com:443
    assetPublicURL: https://ose3-master.example.com:443/console/
    publicURL: https://ose3-master.example.com:443/console/

Add the correct host/port combination to the `corsAllowedOrigins`:

    - ose3-master.example.com:443

Configure the [RequestHeaderIdentityProvider](http://docs.openshift.org/latest/admin_guide/configuring_authentication.html#RequestHeaderIdentityProvider) 

~~~~
  identityProviders:
  - name: requestheader
    challenge: false
    login: false
    provider:
      apiVersion: v1
      kind: RequestHeaderIdentityProvider
      clientCA: /etc/openshift/master/ca.crt
      headers:
      - X-Remote-User
~~~~

Now restart everything:

  systemctl restart httpd
  systemctl restart openshift-master

## Testing

    osc login -u joe \
      --certificate-authority=/etc/openshift/master/ca.crt \
      --server=https://ose3-master.example.com:443

Be sure to sanity test by passing an incorrect password to `osc login`.  You
can also run sanity tests with curl:

Bypass the proxy.  The following should work:

    curl -L -k -H "X-Remote-User: joe" --cert /etc/pki/tls/certs/authproxy.pem https://ose3-master.example.com:8443/oauth/token/request

Bypass the proxy.  This should be denied:

    curl -L -k -H "X-Remote-User: joe" https://ose3-master.example.com:8443/oauth/token/request

## Known limitations

Putting authentication in front of the master means that OpenShift cannot
provide the branded OpenShift login form.  This example will result in a
standard browser 401 basic authentication challenge box.

In addition, this confuses the web console logout scenario as well.  Clicking
logout from the Console will clear the session cookie however to actually
logging out will be dependent on the authentication method chosen.  In the case
of basic authentication the Browser's authentication cache will need to be
cleared.  For kerberos it will be required to run kdestroy.
