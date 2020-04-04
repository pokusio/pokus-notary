# pokus-notary

A POC on using Notary to manage Content Trust 

# Scenario

### Install The Notary Service (include server and signer, client is not signer)

```bash
git clone https://github.com/theupdateframework/notary.git
cd notary
docker-compose build
docker-compose up -d
# ---
# retrieving config and auth secret for notary client, from notary server
mkdir -p ~/.notary && cp cmd/notary/config.json cmd/notary/root-ca.crt ~/.notary
```

> 
> You can also leave off the `-d ~/.docker/trust` argument if you do not care to use `notary` with `Docker` images.
> 

### Install `notary` client

* Tested on `Debian Stretch `, should work on `Ubuntu` : 

```bash
export NOTARY_VERSION=0.6.1
export NOTARY_OS=linux
export NOTARY_CPU_ARCH=amd64
export NOTARY_VERSION_TAG="v${NOTARY_VERSION}"
export NOTARY_PKG_DWLD_URI=https://github.com/theupdateframework/notary/releases/download/${NOTARY_VERSION_TAG}/notary-${NOTARY_OS}-${NOTARY_CPU_ARCH}

# curl -LO ${NOTARY_PKG_DWLD_URI} 

curl -o ./notary-${NOTARY_VERSION_TAG}-${NOTARY_OS}-${NOTARY_CPU_ARCH} -LO ${NOTARY_PKG_DWLD_URI} 

chmod +x ./notary-${NOTARY_VERSION_TAG}-${NOTARY_OS}-${NOTARY_CPU_ARCH}

sudo mv ./notary-${NOTARY_VERSION_TAG}-${NOTARY_OS}-${NOTARY_CPU_ARCH} /usr/bin
sudo ln -s /usr/bin/notary-${NOTARY_VERSION_TAG}-${NOTARY_OS}-${NOTARY_CPU_ARCH} /usr/bin/notary
# # you'll have to delete the symbolic link to unistall : 
# rm /usr/bin/notary
# # and then install new binary version and recreate the same symbolic link, but
# # fom the newly installed binary, the new version.
# 

# 
# ---
# 
notary --help
notary version

```
* after that, you can inspect signatures from docher hub : 

```bash
# # mkdir -p ~/.notary && cp cmd/notary/config.json cmd/notary/root-ca.crt ~/.notary  # # rememmber ? 
# 
jbl@pegasusio:~/notary$ cat ~/.notary/config.json 
{
	"remote_server": {
		"url": "https://notary-server:4443",
		"root_ca": "root-ca.crt"
	}
}

jbl@pegasusio:~/notary$ cp  ~/.notary/config.json ~/.notary/config.mine.json
jbl@pegasusio:~/notary$ rm  ~/.notary/config.json
jbl@pegasusio:~/notary$ touch  ~/.notary/config.json
jbl@pegasusio:~/notary$ notary -s https://notary.docker.io -d ~/.docker/trust list docker.io/library/alpine
ERROR: 2020/04/04 Error parsing config: unexpected end of JSON input
jbl@pegasusio:~/notary$ notary -s https://notary.docker.io list docker.io/library/alpine
ERROR: 2020/04/04 Error parsing config: unexpected end of JSON input
jbl@pegasusio:~/notary$ rm  ~/.notary/config.json
jbl@pegasusio:~/notary$ notary -s https://notary.docker.io list docker.io/library/alpine
NAME               DIGEST                                                              SIZE (BYTES)    ROLE
----               ------                                                              ------------    ----
2.6                9ace551613070689a12857d62c30ef0daa9a376107ec0fff0e34786cedb3399b    528             targets
2.7                9f08005dff552038f0ad2f46b8e65ff3d25641747d3912e3ea8da6785046561a    1374            targets
20190228           6199d795f07e4520fa0169efd5779dcf399cbfd33c73e15b482fcd21c42e1750    2364            targets
20190408           8b6b8c0f71e83cdbf888169bdd9b89f028cba03abff05c50246a191fec31b35a    2364            targets
20190508           5040b958c6bcf588c2ae798fe3ce3455100a388858db6381a6ccd9479f5c1088    1638            targets
# [...]

```

* and from private notary : 

```bash
# ---- 
# --- > First, we need to trust the notary server TLS Cert

export ONPREM_NOTARY_NETNAME=notary-server

cat /etc/hosts | grep ${ONPREM_NOTARY_NETNAME}
ping -c 4 ${ONPREM_NOTARY_NETNAME}

# ex +'/BEGIN CERTIFICATE/,/END CERTIFICATE/p' <(echo | openssl s_client -CApath /dev/null -showcerts -connect notary-server:4443) -scq > ./pegasusio.io.notary-server.crt

# ---
# Trust the downloaded cert : The certificate is not 
# self signed, it is signed by a fiwed root-ca, provided at ./cmd/notary/root-ca.crt, in
# the git repo for your gitops on your Notary Server
# --- 
# https://github.com/theupdateframework/notary/raw/v0.6.1/cmd/notary/root-ca.crt
# 

export NOTARY_VERSION=0.6.1
export NOTARY_OS=linux
export NOTARY_CPU_ARCH=amd64
export NOTARY_VERSION_TAG="v${NOTARY_VERSION}"
export NOTARY_ROOT_CA_DWLD_URI=https://github.com/theupdateframework/notary/raw/${NOTARY_VERSION_TAG}/cmd/notary/root-ca.crt
export NOTARY_SERVICE_NET_NAME=notary-server

curl -o ./notary-root-ca.crt -LO ${NOTARY_ROOT_CA_DWLD_URI}

# sudo mkdir -p /usr/share/ca-certificates/notary
sudo cp ./notary-root-ca.crt /usr/share/ca-certificates/notary-root-ca.crt
sudo update-ca-certificates
# ---
# --- To remove CA : 
# sudo rm /usr/local/share/ca-certificates/notary-root-ca.crt && sudo update-ca-certificates --fresh
# ---
# 


curl -vvv https://${NOTARY_SERVICE_NET_NAME}:4443/

notary -s https://${NOTARY_SERVICE_NET_NAME}:4443/ -d ~/.docker/trust list docker.io/library/alpine


# Now downloading the Pub Cert form notary server
# openssl s_client -showcerts -connect ${NOTARY_SERVICE_NET_NAME}:4443 </dev/null 2>/dev/null|openssl x509 -outform PEM >./notary-server.pem

# openssl crl2pkcs7 -nocrl -certfile ./notary-server.pem | openssl pkcs7 -print_certs -noout


# -- 
# converting root-ca cert to X509 format (*.pem)
# 
# openssl x509 -in /usr/local/share/ca-certificates/notary-root-ca.crt -out /usr/local/share/ca-certificates/notary-root-ca.pem

# ---- 
# --- Now we can verify if the downloaded certificate actually was signed issued by the root-ca
# --- I personnally still get the error : 'error 20 at 0 depth lookup: unable to get local issuer certificate'
# Never the less, adding the root-ca to trusted certificates allows using notary commands on
# the private notary. Whitthout that trust, we would get an error for any notary client command

# openssl verify -CAfile /usr/local/share/ca-certificates/notary-root-ca.pem ./notary-server.pem 

echo | openssl s_client -connect ${NOTARY_SERVICE_NET_NAME}:4443 -CAfile /usr/local/share/ca-certificates/notary-root-ca.crt -no_ssl3




```

* note : 

```bash
jbl@pegasusio:~/notary$ tree ~/.docker/
/home/jbl/.docker/
├── config.json
└── trust
    ├── private
    └── tuf
        └── docker.io
            └── library
                └── alpine
                    ├── changelist
                    └── metadata
                        ├── root.json
                        ├── snapshot.json
                        ├── targets.json
                        └── timestamp.json

8 directories, 5 files
jbl@pegasusio:~/notary$ tree ~/.notary
/home/jbl/.notary
├── config.json
├── config.mine.json
├── private
├── root-ca.crt
└── tuf
    └── docker.io
        └── library
            └── alpine
                ├── changelist
                └── metadata
                    ├── root.json
                    ├── snapshot.json
                    ├── targets.json
                    └── timestamp.json

7 directories, 7 files
jbl@pegasusio:~/notary$ 
```
