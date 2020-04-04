# pokus-notary

A POC on using Notary to manage Content Trust 

# Scenario

```bash
git clone https://github.com/theupdateframework/notary.git
cd notary
docker-compose build
docker-compose up -d
mkdir -p ~/.notary && cp cmd/notary/config.json cmd/notary/root-ca.crt ~/.notary
```
