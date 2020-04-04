# pokus-notary

A POC on using Notary to manage Content Trust 

# Scenario

### Install The Notary Service (include server and signer, client is not signer)

```bash
git clone https://github.com/theupdateframework/notary.git
cd notary
docker-compose build
docker-compose up -d
mkdir -p ~/.notary && cp cmd/notary/config.json cmd/notary/root-ca.crt ~/.notary
```

> 
> You can also leave off the `-d ~/.docker/trust` argument if you do not care to use `notary` with `Docker` images.
> 

### Install `notary` client

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
