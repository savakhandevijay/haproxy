## how to setup SSL certificate locally
- create certificate using mkcert / openssl / letsencrypt (best would be mkcert)
    - Let's Encrypt only issues certificates for publicly accessible domain names that it can validate over the internet.
    - OpenSSL downside is that every client (browser, PHP cURL, etc.) must explicitly trust your self-signed certificate or you'll get errors like: SSL peer certificate or SSH remote key was not OK
    - mkcert creates a local Certificate Authority (CA) and installs it into your operating system's trust store. Any certificates it generates are automatically trusted by your browser and can also be trusted inside your Docker containers.
- Install mkcert using brew or apt command 
- Then Install the local CA (This will create a CA certificate which you will have to add in /usr/local/share/ca-certificates/mkcert.crt of your client container and run update-ca-certificates command, so that browser will trust the certificate and wont ask you permission to allow domain)
    - ``mkcert -install`
- Generate a certificate for your domain
    - `mkcert techlab.dev "*.techlab.dev"`
    - This will create a certificate for above domain includeing wild card domain name like admin.techlab.dev , www.techlab.dev anythig api.techlab.dev
- Create a PEM file for HAProxy
    - HAproxy use only one pem file so you need to merge both public and private key into one file
    - `cat techlab.dev.pem techlab.dev-key.pem > techlab.dev-haproxy.pem`
    - cat takes the input from both file and merge into one file
    - include this certificate in haproxy.cfg frontend section which binds the host machine port with backend server
        - `bind :443 ssl crt /etc/haproxy/certs/techlab.dev-haproxy.pem`
- Update your hosts file
    - 127.0.0.1 techlab.dev api.techlab.dev www.techlab.dev
- Trusting the certificate inside client Docker
    - Find the root CA location using `mkcert -CAROOT`
    - copy rootCA.pem file path , in your Dockerfile which build your client image like nginx or webserver add below line
        > RUN apk add --no-cache ca-certificates
        >
        > COPY rootCA.pem /usr/local/share/ca-certificates/mkcert.crt
        >
        > RUN update-ca-certificates
- With this haproxy configuration for ssl ie. bind :443 section. ssl certificate maped at frontend proxy and your backend server doesnt have to map this ssl certificate

## For Nginx configuration
- follow above step to install and copy root CA certificate steps
- update nginx server conf file as mentioned below
    -
    > listen 443 ssl;
    > http2 on;
    > ssl_certificate /etc/nginx/http.d/cert/techlab.dev.pem;
    > ssl_certificate_key /etc/nginx/http.d/cert/techlab.dev-key.pem;

    - here ssl_certificate would be techlab.dev.pem file and ssl_certificate_key file would be techlab.dev-key.pem file
    - alway check configuration using below command
    - `nginx -t`  tells all the configuration are correct or not
    - ps aux to list down all the active services nginx must be listed here
    - if any directives in conf files are missed or wrongly mapped, nginx fails sliently and no error produce hence check nginx service are running or not always

- Check the domains are correctly mapped or not
- Client domains those are communicating each other must be on same docker network check using docker inspect network <network_name> all the required container must be on same networks
- to check request are communicated correctly across other container try using curl, telnet or wget command
    - `curl -vvv https://techlab.dev`
    - `telnet techlab.dev 443` then do `GET /`
    - `wget -O- http://techlab.dev`
    - `ping techlab.dev` tells where the ip address for request goes out to server
    - check ports are available or listing using `netstat -tlnp`
    - `getent hosts techlab.dev`  for inside the container to check hostname mapped with ip address