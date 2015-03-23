## Add the following settings to /etc/sysconfig/openshift-master:

    OPENSHIFT_OAUTH_REQUEST_HANDLERS=requestheader,session
    OPENSHIFT_OAUTH_REQUEST_HEADER_CA_FILE=/var/lib/openshift/openshift.local.certificates/ca/cert.crt

## Copy certificates to a location SELinux will allow Apache to read:

    pushd /var/lib/openshift
    cp openshift.local.certificates/master/cert.crt /etc/pki/tls/certs/localhost.crt
    cp openshift.local.certificates/master/key.key /etc/pki/tls/private/localhost.key
    cp openshift.local.certificates/ca/cert.crt /etc/pki/CA/certs/ca.crt
    cat openshift.local.certificates/openshift-client/cert.crt \
        openshift.local.certificates/openshift-client/key.key > \
        /etc/pki/tls/certs/openshift-client.pem
    popd

## Create the projects
    openshift ex new-project demo --display-name="OpenShift 3 Demo" \ 
    --description="This is the first demo project with OpenShift v3" \
    --admin=requestheader:joe

## Copy the CA certificate to a the user home directories so they can use `osc login`:

    pushd /var/lib/openshift
    cp openshift.local.certificates/ca/cert.crt ~joe/ca.crt
    popd

## Set up Apache
Unlike OpenShift Enterprise version 2 this proxy does not need to reside on the
same host as the Master.  It uses a client certificate to connect to the Master
which is configured to trust the `X-Remote-User` header.

Note: Currently we are not proxying the web console.  It will still need to be
accessed over port `8443`.  In the near future it will be possible to serve the
web console from a subcontext which will make the proxypass rules simple.

Here is an example of using a file-backed Basic authentication:

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
  SSLProxyMachineCertificateFile /etc/pki/tls/certs/openshift-client.pem

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
    AuthUserFile /etc/openshift-passwd
    AuthType basic

    # For ldap:
    # AuthBasicProvider ldap
    # AuthLDAPURL "ldap://ldap.example.com:389/ou=People,dc=my-domain,dc=com?uid?sub?(objectClass=*)"

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

## Log in

    osc login --cluster=master --server=https://ose3-master.example.com --namespace=sinatra --certificate-authority=/home/joe/ca.crt

## Testing
Bypass the proxy.  The following should work:

    curl -L -k -H "X-Remote-User: joe" --cert /etc/pki/tls/certs/openshift-client.pem https://ose3-master.example.com:8443/oauth/token/request

Bypass the proxy.  This should be denied:

    curl -L -k -H "X-Remote-User: joe" https://ose3-master.example.com:8443/oauth/token/request

Obviously passing an incorrect password to `osc login` should also return a
401.
