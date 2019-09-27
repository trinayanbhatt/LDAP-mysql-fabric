# Fabric CA Cluster with MySQL

In this tutorial we will configure Fabric CA cluster with shared MySQL database

![Cluster Configuration](img/cluster0.png)

## Prerequisities

Download binaries for Hyperledger Fabric v1.4.1

```bash
curl -sSL http://bit.ly/2ysbOFE | bash -s -- 1.4.1 -d -s
rm -f config
```

## Prepare the CA Cert and Key

Use OpenSSL to generate the CA Cert and Key

```bash
mkdir -p ca/private ca/certs ca/newcerts
touch ca/index.txt ca/serial
echo 1000 > ca/serial
openssl ecparam -name prime256v1 -genkey -noout -out ca/private/ca.org1.example.com.key.pem
openssl req -config openssl_root.cnf -new -x509 -sha256 -extensions v3_ca -key ca/private/ca.org1.example.com.key.pem -out ca/certs/ca.org1.example.com.crt.pem -days 3650 -subj "/C=SG/ST=Singapore/L=Singapore/O=org1.example.com/OU=/CN=ca.org1.example.com"
```

Copy the Cert and Key to a new folder. The cert and key pair will be used by both `ca0` and `ca1`

```bash
mkdir ca-store
cp ca/certs/ca.org1.example.com.crt.pem ca-store/
cp ca/private/ca.org1.example.com.key.pem ca-store/ca.org1.example.com.key
```

## Start Components

Bring up the components:

* db - The MySQL DB
* adminer - To browse the MySQL DB contents
* haproxy - CA Proxy
* ca0 - First CA
* ca1 - Second CA

```console
$ docker-compose up -d
Creating network "mysqlcasample_basic" with the default driver
Creating db ... done
Creating ca1     ... done
Creating adminer ... done
Creating ca0     ... done
Creating haproxy ... done
```

## Check Database

Visit `localhost:9000` and login using the following credential:

* System: MySQL
* Server: db
* Username: root
* Password: password
* Database: *Leave blank for now*

You will be able to login and view the databases. Now let's try to register and enroll users to make sure that the CA functions properly

## Register and Enroll

Create the directory for the registrar and user

```bash
REGISTRAR_DIR=$PWD/users/admin
USER1_DIR=$PWD/users/user1
USER2_DIR=$PWD/users/user2
mkdir -p $REGISTRAR_DIR $USER1_DIR $USER2_DIR
```

Enroll the registrar. Note that `7054` is the port of `haproxy`

```bash
export FABRIC_CA_CLIENT_HOME=$REGISTRAR_DIR
fabric-ca-client enroll --csr.names C=SG,ST=Singapore,L=Singapore,O=org1.example.com -m admin -u http://admin:adminpw@localhost:7054
```

Register `User1@org1.example.com` and `User2@org1.example.com`

```bash
fabric-ca-client register --id.name User1@org1.example.com --id.secret mysecret --id.type client --id.affiliation org1 -u http://localhost:7054
fabric-ca-client register --id.name User2@org1.example.com --id.secret mysecret --id.type client --id.affiliation org1 -u http://localhost:7054
```

Enroll `User1@org1.example.com`

```bash
export FABRIC_CA_CLIENT_HOME=$USER1_DIR
fabric-ca-client enroll --csr.names C=SG,ST=Singapore,L=Singapore,O=org1.example.com -u http://User1@org1.example.com:mysecret@localhost:7054
mkdir -p $USER1_DIR/msp/admincerts && cp $USER1_DIR/msp/signcerts/*.pem $USER1_DIR/msp/admincerts/
```

Enroll `User2@org1.example.com`

```bash
export FABRIC_CA_CLIENT_HOME=$USER2_DIR
fabric-ca-client enroll --csr.names C=SG,ST=Singapore,L=Singapore,O=org1.example.com -u http://User2@org1.example.com:mysecret@localhost:7054
mkdir -p $USER2_DIR/msp/admincerts && cp $USER2_DIR/msp/signcerts/*.pem $USER2_DIR/msp/admincerts/
```

## Stop network

```bash
./destroy.sh
```
