---
title: PCF Ops Manager API Reference

language_tabs:
  - shell

toc_footers:

includes:

search: true
---

#THE BASICS

Welcome to the Ops Manager API! You can use our API to access endpoints, which can create, read, update, and delete resources in Ops Manager.

We have language bindings in cURL! You can view code examples in the dark area to the right.

## Authentication

You must pass a token to each API endpoint. To get a token, and curl an API endpoint using that token, follow these instructions:

> From a command line with Ruby installed, install the cf-uaac gem:

```shell
gem install cf-uaac
```
> Target your Ops Manager IP:

```shell
uaac target https://YOUR_OPSMAN_IP/uaa
```
> Log in to your Ops Manager with the Client name "opsman":

```shell
uaac token owner get

Client name: opsman
Client secret:
User name: YOUR_USERNAME_HERE
Password: YOUR_PASSWORD_HERE
```
> Retrieve your Ops Manager access token:

```shell
uaac context
```

Ops Manager uses authorization tokens to allow access to the API. You can get an authorization token from the [settings page](https://10.85.43.43/settings/edit) or by using the uaac command line tool (instructions to the right).

Ops Manager expects for the API key to be included in all API requests to the server in a header that looks like the following:

`Authorization: Bearer YOUR_ACCESS_TOKEN`

<aside class="warning">
You must replace <code>YOUR_ACCESS_TOKEN</code> with your personal API authorization token.
</aside>


## Workflow

**Available** ---> **Staged** --Apply changes--> **Deployed**

Products (.pivotal files) can be [uploaded](#uploading-a-product) or [downloaded](#download-a-given-product-with-version-from-pivotal-network) to the [Available Products](#available-products) namespace. 

They are then moved into the [Staged Products](#staged-products) namespace, which describes the desired state of the installation, and where [configuration changes](#retrieving-compute-and-disk-configuration-for-a-job) are made. 

When queued changes are [applied](#triggering-an-install-process) successfully, the [Deployed products](#deployed-products) namespace mirrors the Staged Products namespace until further changes are made.


## Status Codes

Ops Manager uses conventional HTTP response codes to indicate the success or failure of an API request. Generally, codes in the 2xx range indicate success, codes in the 4xx range indicate an error that failed given the information provided (e.g., a required parameter was omitted), and codes in the 5xx range indicate an error with the Ops Manager server.

Code | Description
--------- | -------
200 - OK | Everything worked as expected
404 - not found | The route requested does not exist
400 - bad request | The request is syntactically incorrect
401 - unauthorized | The access token has expired or is invalid
409 - conflict | Another user is logged in
422 - unprocessable entity | The request is syntactically correct but the supplied values do not work
500 - internal server error | Something went wrong with our server
503 - service unavailable | The authentication service is not available yet

##Pivotal Network API token

You must set your Pivotal Network API Token to use Ops Manager's integration with Pivotal Network (e.g. finding and download product updates).


```shell
curl "https://example.com/api/v0/settings/pivotal_network_settings" -X PUT \
  -H "Content-Type: application/json" \
  -d '{"pivotal_network_settings": {"api_token": "pivnet-api-token"}}' \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK
{
  "success": true
}
```

#### HTTP Request

`PUT /api/v0/settings/pivotal_network_settings`



#COMMON TASKS
##Setting up Ops Manager
Ops Manager can be set up with an [internal user store](#setting-up-with-an-internal-userstore) or with [an external identity provider](#setting-up-with-saml).

##Installing products
1. Products can be [uploaded to Ops Manager](#uploading-a-product). Once the product has been uploaded, it is in the "Available Products" namespace. 
2. The product then needs to be [added to the "Staged Products" namespace](#adding-an-available-product)
3. The product needs to be [configured](#configuring-resources-for-a-job) completely with all required fields
4. Once all products in the installation have been completely configured, all changes can be applied by [triggering an install process](#triggering-an-install-process)

##Upgrading products
1. New versions of a product can be [downloaded](#downloading-a-product-from-pivotal-network) directly from Pivotal Network, or [imported into the Ops Manager application]((#uploading-a-product))
2. The existing version of the product can then be [upgraded](#upgrading-a-product)
3. Any necessary [configuration changes](#configuring-resources-for-a-job) are then made
4. Changes can be applied by [triggering an install process](#triggering-an-install-process)

##Viewing logs and credentials
Multiple types of logs and credentials are available in Ops Manager. These are:

* Installation logs - refers to the [changelog](#getting-a-list-of-recent-install-events) and the [more detailed BOSH logs associated with each Install action](#fetching-installation-logs)
* Product job logs - refers to BOSH logs associated with individual jobs belonging to product. First, [view a list of jobs for the product](), then use the relevant job id to [enqueue a log download for the job](#enqueueing-log-downloads-for-a-given-job), then [check the status of the job](#listing-log-download-tasks-for-a-given-job). When the status of the job is 'downloaded', [download the zip file with the logs](#download-zip-file-with-logs).
* Product credentials - Credentials that Ops Manager auto-creates for product jobs. These are obtained by first [getting the list of available credentials](#viewing-available-credentials), and then [requesting the appropriate credential](#fetching-credentials).
* BOSH director credentials - Credentials that Ops Manager auto-creates for the BOSH director and associated jobs. As with products, one needs to first [discover the list of available credentials](#getting-a-list-of-available-credentials), and then [request the appropriate credential](#listing-an-rsa_key-credential).

##Upgrading a stemcell
Stemcells may need to be upgraded when security vulnerabilities are discovered. 

To upgrade a stemcell, [upload it into Ops Manager](#uploading-stemcells). It will automatically be associated with the appropriate products. [Trigger an install process](#triggering-an-install-process) for the change to take effect.

##Upgrading Ops Manager
Upgrading Ops Manager is a two step process:

1. Export your existing Ops Manager installation using the [export installation asset collection endpoint](#exporting-an-installation-asset-collection)

2. After you have provisioned a fresh Ops Manager VM with a *.ova file corresponding to the latest version of Ops Manager, [import the installation asset collection](#importing-an-installation-asset-collection) you exported previously, and [trigger an install process](#triggering-an-install-process).

<aside class="warning">
You cannot import an Ops manager installation onto an Ops Manager that has previously been set up. You must import onto a fresh Ops Manager which has not been set up yet.
</aside>

#CORE CONCEPTS
##Setup

### Setting up with SAML

```shell
curl "https://example.com/api/v0/setup" -d 'setup[decryption_passphrase]=passphrase&setup[decryption_passphrase_confirmation]=passphrase&setup[eula_accepted]=true&setup[identity_provider]=saml&setup[idp_metadata]=%3Cvalid_xml%3E%3C%2Fvalid_xml%3E' -X POST \
	-H "Content-Type: application/x-www-form-urlencoded"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

setup[decryption_passphrase]=passphrase&setup[decryption_passphrase_confirmation]=passphrase&setup[eula_accepted]=true&setup[identity_provider]=saml&setup[idp_metadata]=%3Cvalid_xml%3E%3C%2Fvalid_xml%3E
```

#### HTTP Request

`POST /api/v0/setup`




#### Query Parameters

Parameter  | Description
---------- | -----------
setup[decryption_passphrase]	| Decryption passphrase
setup[decryption_passphrase_confirmation]	| Confirm decryption passphrase
setup[eula_accepted]	| Accept EULA
setup[identity_provider]	| Using SAML as our identity provider
setup[idp_metadata]	| URL or XML for idp_metadata



### Setting up with an internal userstore

```shell
curl "https://example.com/api/v0/setup" -d 'setup[decryption_passphrase]=passphrase&setup[decryption_passphrase_confirmation]=passphrase&setup[eula_accepted]=true&setup[identity_provider]=internal&setup[admin_user_name]=user-ed942e358eb61868dc87&setup[admin_password]=password&setup[admin_password_confirmation]=password' -X POST \
	-H "Content-Type: application/x-www-form-urlencoded"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

setup[decryption_passphrase]=passphrase&setup[decryption_passphrase_confirmation]=passphrase&setup[eula_accepted]=true&setup[identity_provider]=internal&setup[admin_user_name]=user-ed942e358eb61868dc87&setup[admin_password]=password&setup[admin_password_confirmation]=password
```

#### HTTP Request

`POST /api/v0/setup`


#### Query Parameters

Parameter  | Description
---------- | -----------
setup[decryption_passphrase]	| Decryption passphrase
setup[decryption_passphrase_confirmation]	| Confirm decryption passphrase
setup[eula_accepted]	| Accept EULA
setup[identity_provider]	| Using internal as our identity provider
setup[admin_user_name]	| User name
setup[admin_password]	| Password
setup[admin_password_confirmation]	| Confirm password

## Installations

### Triggering an install process


```shell
curl "https://example.com/api/v0/installations" -d 'enabled_errands[product_1_guid][pre_delete_errands][]=a&enabled_errands[product_1_guid][pre_delete_errands][]=b&enabled_errands[product_2_guid][post_deploy_errands][]=c&enabled_errands[product_2_guid][post_deploy_errands][]=d' -X POST \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN" \
	-H "Content-Type: application/x-www-form-urlencoded"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "install": {
    "id": 1
  }
}
```

#### HTTP Request

`POST /api/v0/installations`

Transmits pending changes to BOSH. Submitting a POST request to this endpoint is equivalent to hitting the "Apply changes" button in the GUI.


#### Query Parameters

Parameter  | Description
---------- | -----------
ignore_warnings	| When true, bypass warnings from ignorable verifiers
enabled_errands	| Hash of products with their enabled errands

### Getting the status of an install process

```shell
curl "https://example.com/api/v0/installations/4" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "status": "running"
}
```

#### HTTP Request

`GET /api/v0/installations/:id`

This endpoint returns the status of an install process. Possible values for "status" are ‘running’, ‘succeeded’, or ‘failed’.

###Getting a list of recent install events


```shell
curl "https://example.com/api/v0/installations" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "installations": [
    {
      "user_name": "admin",
      "finished_at": null,
      "status": "running",
      "id": 3
    },
    {
      "user_name": "admin",
      "finished_at": "2016-04-10T15:14:45.488Z",
      "status": "succeeded",
      "id": 2
    }
  ]
}
```

#### HTTP Request

`GET /api/v0/installations`

This endpoint returns a table containing a history of changes by user (i.e. each time the "Apply Changes" button was clicked in the GUI or the installations controller was triggered via the API). Possible values for "status" are ‘running’, ‘succeeded’, or ‘failed’.


### Fetching installation logs


```shell
curl "https://example.com/api/v0/installations/1/logs" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"

```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "logs": "some-text"
}
```

#### HTTP Request

`GET /api/v0/installations/:installation_id/logs`

This endpoint returns BOSH logs for a given installation id.

#### Query Parameters

Parameter  | Description
---------- | -----------
installation_id | ID of the installation


##Staged BOSH Director
### Fetching a manifest

```shell
curl "https://example.com/api/v0/staged/director/manifest" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "manifest": {
    "name": "p-bosh-installation-name",
    "releases": [
      {
        "name": "bosh",
        "url": "file:///tmp/FactoryGirl-Tempest::PackageLibrary-FACTORY20160412-125-nf7uex/internal_releases/bosh"
      },
      {
        "name": "bosh-vsphere-cpi",
        "url": "file:///tmp/FactoryGirl-Tempest::PackageLibrary-FACTORY20160412-125-nf7uex/internal_releases/cpi"
      },
      {
        "name": "uaa",
        "url": "file:///tmp/FactoryGirl-Tempest::PackageLibrary-FACTORY20160412-125-nf7uex/internal_releases/uaa"
      }
    ],
    "networks": [
      {
        "name": "default",
        "type": "manual",
        "subnets": [
          {
            "netmask": "255.255.255.0",
            "dns": [
              "192.168.163.1"
            ],
            "gateway": "192.168.163.2",
            "range": "192.168.163.0/24",
            "cloud_properties": {
              "name": "vsphere-network"
            }
          }
        ]
      }
    ],
    "disk_pools": [
      {
        "name": "director_disk_pool",
        "disk_size": 51200,
        "cloud_properties": {
          "type": "thin"
        }
      }
    ],
    "resource_pools": [
      {
        "name": "director_resource_pool",
        "network": "default",
        "stemcell": {
          "url": "file:///tmp/FactoryGirl-Tempest::PackageLibrary-FACTORY20160412-125-nf7uex/stemcells/bosh-stemcell-3215-vsphere-esxi-ubuntu-trusty-go_agent.tgz"
        },
        "cloud_properties": {
          "cpu": 2,
          "disk": 51200,
          "ram": 4096,
          "datacenters": [
            {
              "name": "vsphere-datacenter",
              "clusters": [
                {
                  "vsphere-cluster": {
                  }
                }
              ]
            }
          ]
        },
        "env": {
          "bosh": {
            "password": "$6$vm-salt-12345687$nWtgnl1OMN2ZYBP4KhIrjuSAKJ968h43goOBQBBkaGX9vJlK2DL5QanzPSppfEogEIF7MzxFHR.6xLKVe1olr."
          }
        }
      }
    ],
    "jobs": [
      {
        "name": "bosh",
        "instances": 1,
        "templates": [
          {
            "name": "postgres",
            "release": "bosh"
          },
          {
            "name": "nats",
            "release": "bosh"
          },
          {
            "name": "redis",
            "release": "bosh"
          },
          {
            "name": "director",
            "release": "bosh"
          },
          {
            "name": "health_monitor",
            "release": "bosh"
          },
          {
            "name": "uaa",
            "release": "uaa"
          },
          {
            "name": "vsphere_cpi",
            "release": "bosh-vsphere-cpi"
          },
          {
            "name": "blobstore",
            "release": "bosh"
          }
        ],
        "resource_pool": "director_resource_pool",
        "persistent_disk_pool": "director_disk_pool",
        "networks": [
          {
            "name": "default",
            "static_ips": [
              "192.168.163.3"
            ],
            "default": [
              "dns",
              "gateway"
            ]
          }
        ],
        "properties": {
          "env": {
          },
          "nats": {
            "address": "127.0.0.1",
            "user": "nats",
            "password": "nats-password"
          },
          "redis": {
            "listen_address": "127.0.0.1",
            "address": "127.0.0.1",
            "password": "redis-password"
          },
          "postgres": {
            "host": "127.0.0.1",
            "user": "postgres",
            "password": "postgres-password",
            "database": "bosh",
            "additional_databases": [
              "uaa"
            ],
            "adapter": "postgres"
          },
          "blobstore": {
            "address": "192.168.163.3",
            "port": 25250,
            "provider": "dav",
            "director": {
              "user": "blobstore",
              "password": "blobstore-password"
            },
            "agent": {
              "user": "blobstore",
              "password": "blobstore-password"
            }
          },
          "director": {
            "address": "192.168.163.3",
            "name": "p-bosh-installation-name",
            "cpi_job": "vsphere_cpi",
            "user_management": {
              "provider": "uaa",
              "uaa": {
                "url": "https://192.168.163.3:8443",
                "public_key": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAqLM8nKKL6NcKHCk4/jgc\naSFbz6APw7pbLTHb8nqezmTCs/R0spsXoUrwRuPrtBwwkzjc3SfGX6Lq2MouBa0F\nJMlw+o/Iq/+JHDnnH00rOjlBg62y5bL6ABBlKn0yh9HqnL5cwOArtd3J2xP87PEM\nykyR5ag1CfiVjwexOH1NgDUmw8pZ1kwILwtWmpDxFIB32fhaCMCcSXOvyFZJDOhj\n/IM8R2mAUNOz8vSmcCOWb/BLxjcjg5qsNLTBCnDOtmMC+EBX6eODZJ6g3aa5UnHA\nxskUCNDM3taBLQ+fIF3u+LZDeGdiy0Jv/xsEMHsgN9IiMwKWBSsLwHSnMDeaFT9P\ngwIDAQAB\n-----END PUBLIC KEY-----\n"
              }
            },
            "max_threads": 3,
            "db": {
              "host": "127.0.0.1",
              "user": "postgres",
              "password": "postgres-password",
              "database": "bosh",
              "additional_databases": [
                "uaa"
              ],
              "adapter": "postgres"
            },
            "trusted_certs": null,
            "ssl": {
              "key": "-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEAqLM8nKKL6NcKHCk4/jgcaSFbz6APw7pbLTHb8nqezmTCs/R0\nspsXoUrwRuPrtBwwkzjc3SfGX6Lq2MouBa0FJMlw+o/Iq/+JHDnnH00rOjlBg62y\n5bL6ABBlKn0yh9HqnL5cwOArtd3J2xP87PEMykyR5ag1CfiVjwexOH1NgDUmw8pZ\n1kwILwtWmpDxFIB32fhaCMCcSXOvyFZJDOhj/IM8R2mAUNOz8vSmcCOWb/BLxjcj\ng5qsNLTBCnDOtmMC+EBX6eODZJ6g3aa5UnHAxskUCNDM3taBLQ+fIF3u+LZDeGdi\ny0Jv/xsEMHsgN9IiMwKWBSsLwHSnMDeaFT9PgwIDAQABAoIBAQCKoRea49wzD5sI\nPzvNdJCsN7R5rt+liNtqDUHgRbGAi76QIL9REi/d5HYE20ES9eNY5+5fclMKvhdc\n5O/izCag70R/Mm7GIKwsXMy3pTNzmh9jNPcA2Q2lxdNMkitW/0JbYfdYrB5fSg2Z\nkRhUIVXQXBG8dnh3ZCaKrdiNQjLQugcWdkwREhe4gObftBFppbERSUVPVplHwt2t\nsCcsrc6O5KkIvdU1wFzGr1bWZ0mOrw8dL9kopbA1q+lSF4dHjeBLxLrV9bY4dpBl\nsKf4EG048KHJgM80Cco4AwVBjTQDkY/0OHteTLjLh0Z9HelpEX1yBH7sBuDRCHpO\nHiIGuupZAoGBANZDCy7Ehz2AFSew94bHNaKGeUSthzVS76PxAgOod2c3sh4vEmYC\n//OtDZLu3W/xfJnqJC2qEb+vRexDkSwytNNxVHC5w7MW10pTUI51w5FB62s7okqG\n+9i6MbPMk9mT42/AlsXASs5PIK/EkQkm1hbJfzVx0QMIX804gr8crslPAoGBAMmQ\nFtgYYU1HRT8a1OP6jpwxorZ+FQVLtRmvegKBr1ybC/nODs/KQI5ol9oPYlucGLuP\ndkYQEpJjaCItm2a4OsuHfK61VEIchOTrv/oxNcmY7SSRXshKrqTib2w5sfnR4WYW\nx0F47MnUXs8sl1CX5iSKoZJhXWLSeexxD7VaNOGNAoGBAJeEV78d2WljTxJ/cbuM\n2l/xaoZnlErgOHkdsMf3dWC3oSz5KrCbBHdEdGnoow1Ln0qUqjrknqKIBxF6AopX\n3Un9RbJlm3/k8iAsZLYpj0AEdr+hLzY22Jg9q3IzhIaDr31SmwyC3COjD0Fc5xeq\nsBDzMxMPRrg3TtAoW0Vcujm/AoGARRVLnxkMEG6C/1P074Zq5oHkoOOp1LzT/0+z\nY7SLJBRIEIBddz58zdJvaV+oeHmRyIctJGpR0zaa9EvpXVV7YVK4mzCvBlG8ArIC\nhH/lTYlKjiP89m0SWpT5V4CWzWbv+AuKk5gcoDhXnm5MFmVZjeCt6/vPBBXbj/xY\nQ/H8+ekCgYB36iuoWdihQDPb0wP3iUCs3/nfZjX7huVCon1MMXXvyKZS7lvgkUp2\nrhyV2A/QcaxW9f/hFyAgzj/e16de8ypy/CoSsRkBdIsZlRs9SUw3mX9a7SBMC9Le\nLU1aPXAqPdcBNIlBFEgLt6A18ZYD3wwdH6F+Mqocge8WljnTBVrt+g==\n-----END RSA PRIVATE KEY-----\n",
              "cert": "-----BEGIN CERTIFICATE-----\nMIIC3zCCAcegAwIBAgIBADANBgkqhkiG9w0BAQUFADAzMQswCQYDVQQGEwJVUzEQ\nMA4GA1UECgwHUGl2b3RhbDESMBAGA1UEAwwJc29tZS1uYW1lMB4XDTEzMDcwOTIy\nMTI1MVoXDTE1MDcwOTIyMTI1MVowMzELMAkGA1UEBhMCVVMxEDAOBgNVBAoMB1Bp\ndm90YWwxEjAQBgNVBAMMCXNvbWUtbmFtZTCCASIwDQYJKoZIhvcNAQEBBQADggEP\nADCCAQoCggEBALny6vikcrf/3Do/Nq2Sh/ji8d8dhACUIOaf+ml2cWuRKuP6TcQC\nohbLBHlbm3l5QqD/lvz1EaSy168SVPIsy34sAcioYv+7oOIAHqS4gCVEb7AQrsKb\nZIFUmp7J9qCmkJtyImvViQUUJrWOSaQi3eCAh8uSHyelBNPdaHnj0k8WufKp6hzK\npAr6Xv8OgFSwjD0+XROiTyRpvsQoTm8/XtdhAFpwjTnqR9gxlXqzDqmBRzWduO/M\nyctwF9gtggUp31USmqo5fVC+nr1wh6a/JlbUETtcRhk8jR/FnAVHLSJC4+FZqhmK\nDcemvIfEJaKCNqhvytRLI01l+0p1pm8cqxUCAwEAATANBgkqhkiG9w0BAQUFAAOC\nAQEAru0hKd5gd1WDS6AUrIa8AYCWUrGHMd5P63FWB0KyUnfIDTX4tegHTF+olOxA\nkrR4IRVgFbu3u0pnURFn2N1Et4pZwvW9PEamwkIGHEpYmASOiUZqvrthx/WpUaeu\n+xQIWa1S140v4wa/27UTakAuR+GnA6StJSIRBEBa7hafqpeLGPugZVWRtY3m/OIF\nLICs2U2X8P86RMUWgdtM9//x3t6O7IJzhrSKRkZDmSWAv6EbS/aTpXOPpJFpJtT8\n0aETgAhauKhyp6CeajL3Nc3FfoIONK427VbfIGKJ1Qw7OwTA4N0VPpETiGN7KrfD\nU4mSCEKQ0cIypQAm9rkPboHfwg==\n-----END CERTIFICATE-----\n"
            }
          },
          "hm": {
            "director_account": {
              "ca_cert": "-----BEGIN CERTIFICATE-----\nMIIC+zCCAeOgAwIBAgIBADANBgkqhkiG9w0BAQUFADAfMQswCQYDVQQGEwJVUzEQ\nMA4GA1UECgwHUGl2b3RhbDAeFw0xNjA0MTExNTE0NTJaFw0yMDA0MTIxNTE0NTJa\nMB8xCzAJBgNVBAYTAlVTMRAwDgYDVQQKDAdQaXZvdGFsMIIBIjANBgkqhkiG9w0B\nAQEFAAOCAQ8AMIIBCgKCAQEAqhDlqG9QlQB4jYA/gt4ed8kjLV4yW1RbUJzBazql\nmia5Y5en5X4Agb52pZ7gHQIvzbdXKlE+eWs1WtlcfoooUb7CNmFAQjwHRjQTNyzK\nwbPQLQpGQO4nmsLFq/lu2yn6HA7rTYuAGu94JkL1wuUWZMxqmi5huwRJLrV7c4Nh\nqBQL+0nuRdLtzEZrVefXiGKNaDy9+eZNzJJH9fT8sLniO4byM1ndSH+7tqAMpCac\n5RIjQkeYk00e2RtCmW76o9d/YLB2G2EeutOzDEIZgVMBHpL5WwMt/zo5WHT4Lnj0\nGvK9FZ9cNNOZy7/sOWDgv4NtyqDpT7h5hf/JR/fYBhvBvwIDAQABo0IwQDAdBgNV\nHQ4EFgQUJUWOCmGz0acVHqye2ceBjqlX/64wDwYDVR0TAQH/BAUwAwEB/zAOBgNV\nHQ8BAf8EBAMCAQYwDQYJKoZIhvcNAQEFBQADggEBAD48E5CPJf5ihmNiFvHVypz5\nPvZn6QFMRaMjOTUvtOrd/V9jp6t8L8NxQOvTVoCab2247iQVxjjn9iStZgW1Umon\n4tYLkBlH6AV3mLrwJ1yTyuTC8CDUm4tGCcBYp1D2MOV/HhQjF4kQft9PA6fdOKeu\nWCpccLdedQlX/FK3lknw7lJ99DNV3MjFHlP7e0m2On/ArdpqdJNxii3PRYOR6d7x\nhYvX1EPUxBj+rGG4tBl5kdr0gs1bogGsnDaoIqXspCWX4xOPA/qGcNmDaA28hcr/\nzqYTHB1LdZyFRdjlc3SJHxmV3rGoa2mL9taMryvBpS0r+yZXjKIe/Sp/eCEhfLo=\n-----END CERTIFICATE-----\n",
              "user": "health_monitor",
              "password": "health_monitor"
            },
            "resurrector_enabled": false,
            "pagerduty_enabled": false,
            "pagerduty": {
              "service_key": null,
              "http_proxy": null
            },
            "email_notifications": false,
            "email_recipients": [

            ],
            "smtp": {
              "from": null,
              "host": null,
              "port": 25,
              "domain": null,
              "tls": false,
              "user": null,
              "password": null
            }
          },
          "agent": {
            "mbus": "nats://nats:nats-password@192.168.163.3:4222"
          },
          "ntp": [
            "us.pool.ntp.org"
          ],
          "login": {
            "protocol": "https",
            "branding": {
              "company_name": "Pivotal",
              "product_logo": "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",
              "square_logo": "iVBORw0KGgoAAAANSUhEUgAAAGwAAABsCAYAAACPZlfNAAAAAXNSR0IArs4c6QAABYtJREFUeAHtnVtsFFUYx7/d3ruWotUKVIkNaCw02YgJGBRTMd4CokUejD4QH4gxQcIDeHnBmPjkhSghUYLGe3ywPtAHNCo0QgkWwi2tXG2V1kIpLXTbLt1tS9dzlmzSJssZhv32zDk7/2km2znn7Pd9+/vt2Z2dmW0D9Obat4gCiwiLBQQSLflSViAQeN6Can1fYiJBFPQ9BcsAQBiEWUbAsnIxwyDMMgKWlYsZBmGWEbCsXMwwCLOMgGXlYoZBmGUELCsXMwzCLCNgWbmYYRBmGQHLysUMgzDLCFhWLmYYhFlGwLJyMcMgzDIClpWLGQZhlhGwrFzMMAizjIBl5WKGQZhlBCwrV1xbb96y59V1VFJQmLawQNrWa43x8XEaHo1fW+Oj1H8lSqf6eulEbw+dvNhLvcNDinvb0WWksAdm3UWhwiJ2gt2RAWo80UY7jrdSU8cZGrt6lT1HtgMaKSxbD7qqfDq99tAjyTUSG6FP9v1BH+3dTUPxeLZSssf17U5HeXEJbXr8aerY+A6tf7iOxFeu2OFmI6BvhaVgVoRCtHl5PTW8/AoV5xekmo299b2wlJn6+WFqWrOWKkpDqSYjbyFskpZFs++hL1e9NKnFvF+t3OmQOwzdkcgUmnnBABXm5Ys1j8qKisVadFPvS8tramn1goX09eEDU+KbsmGlsMbjbbT6x++UDOVORGXoFppXOYMerLqbVsyrpcWzqykYdH5R+fjZlcnd/8sjV5Q5vOh0rt6LqhhyJsQ3uC+ID8ry89aHYtf90W1bKLzlffr19EnH6HIP8oXasOM4LwbkrLB0MP+6cJ6e+eoz+vTP5nTdU9peDC+Ysm3Khq+ESehy5r3e2ECHu7uUDuqq59Id4iXVtMV3wqSACSHt3V2/KF3I97qayjuVY7zo9KUwCfq3M6coNjamZD6zrFzZ70Wnb4XFxseoK3JZyXzWtGnKfi86fStMwu6LRpXMZ5RBmBKQ7k75XqZa8gLmPZ/Nq0hFkLnvttJSZUT5Oc60xbfC5CGs6lsrlD56hgaV/V50+lbYkuo5VFygPp3SMwxhXjwp0+bcsGRp2vZU48TEBB09153aNObWlzNMHo1/6r4apYTmsx10MTqsHONFp5VH6zMBtWbhYtq6YpVjiJ/ajjmO8WKAL4QFxamWZffPT1678dicex05D4jTKj8cO+Q4zosBOSXs7bonktci5ovjgPIUye3ieo3wzKrk+TC5faPLGz83On6ovtFY3ONySth7Ty67qbPMk6Hu+edv+vzg/slNRv3uy52O6xk40HWW6r/94nrdRrTn1AzLhOju9tP03DfbKTo6mkmYrN/X98L6xQHgTb/vpG0t+5LnybJOPMMEvhXWOXCJvj9yiD7Yu4sGRkYyxKjv7r4RJi+Na+05Rwf/66SG1qO0v/NffZQZM+WUsI07d1BC/MTE144GYzHxJYcYDYq1vb/f8WQlI9OshsopYZubm7IKy4Tg2K03wYKLGiDMBSwThkKYCRZc1ABhLmCZMBTCTLDgogYIcwHLhKEQZoIFFzVAmAtYJgyFMBMsuKgBwlzAMmEohJlgwUUNEOYClglDIcwECy5qgDAXsEwYCmEmWHBRA4S5gGXCUAgzwYKLGow84yyvuyhR/GW19kt9Lh5ibg01UtjS7VtzizLjo8FLIiNMHaEgTAdlxhwQxghTRygI00GZMQeEMcLUEQrCdFBmzAFhjDB1hIIwHZQZc0AYI0wdoSBMB2XGHBDGCFNHKAjTQZkxB4QxwtQRCsJ0UGbMAWGMMHWEgjAdlBlzQBgjTB2hIEwHZcYcEMYIU0coCNNBmTEHhDHC1BEKwnRQZswBYYwwdYSCMB2UGXNAGCNMHaEgTAdlxhziUu1Ei8M/+WFMh1CZEUi0/A+j7hNSB5Wo2wAAAABJRU5ErkJggg==",
              "footer_legal_text": "©2016 Pivotal Software, Inc. All Rights Reserved",
              "footer_links": null
            },
            "saml": {
            }
          },
          "uaa": {
            "admin": {
              "client_secret": "blank-password"
            },
            "disableInternalAuth": false,
            "sslCertificate": "-----BEGIN CERTIFICATE-----\nMIIC3zCCAcegAwIBAgIBADANBgkqhkiG9w0BAQUFADAzMQswCQYDVQQGEwJVUzEQ\nMA4GA1UECgwHUGl2b3RhbDESMBAGA1UEAwwJc29tZS1uYW1lMB4XDTEzMDcwOTIy\nMTI1MVoXDTE1MDcwOTIyMTI1MVowMzELMAkGA1UEBhMCVVMxEDAOBgNVBAoMB1Bp\ndm90YWwxEjAQBgNVBAMMCXNvbWUtbmFtZTCCASIwDQYJKoZIhvcNAQEBBQADggEP\nADCCAQoCggEBALny6vikcrf/3Do/Nq2Sh/ji8d8dhACUIOaf+ml2cWuRKuP6TcQC\nohbLBHlbm3l5QqD/lvz1EaSy168SVPIsy34sAcioYv+7oOIAHqS4gCVEb7AQrsKb\nZIFUmp7J9qCmkJtyImvViQUUJrWOSaQi3eCAh8uSHyelBNPdaHnj0k8WufKp6hzK\npAr6Xv8OgFSwjD0+XROiTyRpvsQoTm8/XtdhAFpwjTnqR9gxlXqzDqmBRzWduO/M\nyctwF9gtggUp31USmqo5fVC+nr1wh6a/JlbUETtcRhk8jR/FnAVHLSJC4+FZqhmK\nDcemvIfEJaKCNqhvytRLI01l+0p1pm8cqxUCAwEAATANBgkqhkiG9w0BAQUFAAOC\nAQEAru0hKd5gd1WDS6AUrIa8AYCWUrGHMd5P63FWB0KyUnfIDTX4tegHTF+olOxA\nkrR4IRVgFbu3u0pnURFn2N1Et4pZwvW9PEamwkIGHEpYmASOiUZqvrthx/WpUaeu\n+xQIWa1S140v4wa/27UTakAuR+GnA6StJSIRBEBa7hafqpeLGPugZVWRtY3m/OIF\nLICs2U2X8P86RMUWgdtM9//x3t6O7IJzhrSKRkZDmSWAv6EbS/aTpXOPpJFpJtT8\n0aETgAhauKhyp6CeajL3Nc3FfoIONK427VbfIGKJ1Qw7OwTA4N0VPpETiGN7KrfD\nU4mSCEKQ0cIypQAm9rkPboHfwg==\n-----END CERTIFICATE-----\n",
            "sslPrivateKey": "-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEAqLM8nKKL6NcKHCk4/jgcaSFbz6APw7pbLTHb8nqezmTCs/R0\nspsXoUrwRuPrtBwwkzjc3SfGX6Lq2MouBa0FJMlw+o/Iq/+JHDnnH00rOjlBg62y\n5bL6ABBlKn0yh9HqnL5cwOArtd3J2xP87PEMykyR5ag1CfiVjwexOH1NgDUmw8pZ\n1kwILwtWmpDxFIB32fhaCMCcSXOvyFZJDOhj/IM8R2mAUNOz8vSmcCOWb/BLxjcj\ng5qsNLTBCnDOtmMC+EBX6eODZJ6g3aa5UnHAxskUCNDM3taBLQ+fIF3u+LZDeGdi\ny0Jv/xsEMHsgN9IiMwKWBSsLwHSnMDeaFT9PgwIDAQABAoIBAQCKoRea49wzD5sI\nPzvNdJCsN7R5rt+liNtqDUHgRbGAi76QIL9REi/d5HYE20ES9eNY5+5fclMKvhdc\n5O/izCag70R/Mm7GIKwsXMy3pTNzmh9jNPcA2Q2lxdNMkitW/0JbYfdYrB5fSg2Z\nkRhUIVXQXBG8dnh3ZCaKrdiNQjLQugcWdkwREhe4gObftBFppbERSUVPVplHwt2t\nsCcsrc6O5KkIvdU1wFzGr1bWZ0mOrw8dL9kopbA1q+lSF4dHjeBLxLrV9bY4dpBl\nsKf4EG048KHJgM80Cco4AwVBjTQDkY/0OHteTLjLh0Z9HelpEX1yBH7sBuDRCHpO\nHiIGuupZAoGBANZDCy7Ehz2AFSew94bHNaKGeUSthzVS76PxAgOod2c3sh4vEmYC\n//OtDZLu3W/xfJnqJC2qEb+vRexDkSwytNNxVHC5w7MW10pTUI51w5FB62s7okqG\n+9i6MbPMk9mT42/AlsXASs5PIK/EkQkm1hbJfzVx0QMIX804gr8crslPAoGBAMmQ\nFtgYYU1HRT8a1OP6jpwxorZ+FQVLtRmvegKBr1ybC/nODs/KQI5ol9oPYlucGLuP\ndkYQEpJjaCItm2a4OsuHfK61VEIchOTrv/oxNcmY7SSRXshKrqTib2w5sfnR4WYW\nx0F47MnUXs8sl1CX5iSKoZJhXWLSeexxD7VaNOGNAoGBAJeEV78d2WljTxJ/cbuM\n2l/xaoZnlErgOHkdsMf3dWC3oSz5KrCbBHdEdGnoow1Ln0qUqjrknqKIBxF6AopX\n3Un9RbJlm3/k8iAsZLYpj0AEdr+hLzY22Jg9q3IzhIaDr31SmwyC3COjD0Fc5xeq\nsBDzMxMPRrg3TtAoW0Vcujm/AoGARRVLnxkMEG6C/1P074Zq5oHkoOOp1LzT/0+z\nY7SLJBRIEIBddz58zdJvaV+oeHmRyIctJGpR0zaa9EvpXVV7YVK4mzCvBlG8ArIC\nhH/lTYlKjiP89m0SWpT5V4CWzWbv+AuKk5gcoDhXnm5MFmVZjeCt6/vPBBXbj/xY\nQ/H8+ekCgYB36iuoWdihQDPb0wP3iUCs3/nfZjX7huVCon1MMXXvyKZS7lvgkUp2\nrhyV2A/QcaxW9f/hFyAgzj/e16de8ypy/CoSsRkBdIsZlRs9SUw3mX9a7SBMC9Le\nLU1aPXAqPdcBNIlBFEgLt6A18ZYD3wwdH6F+Mqocge8WljnTBVrt+g==\n-----END RSA PRIVATE KEY-----\n",
            "require_https": false,
            "url": "https://192.168.163.3:8443",
            "jwt": {
              "signing_key": "-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEAqLM8nKKL6NcKHCk4/jgcaSFbz6APw7pbLTHb8nqezmTCs/R0\nspsXoUrwRuPrtBwwkzjc3SfGX6Lq2MouBa0FJMlw+o/Iq/+JHDnnH00rOjlBg62y\n5bL6ABBlKn0yh9HqnL5cwOArtd3J2xP87PEMykyR5ag1CfiVjwexOH1NgDUmw8pZ\n1kwILwtWmpDxFIB32fhaCMCcSXOvyFZJDOhj/IM8R2mAUNOz8vSmcCOWb/BLxjcj\ng5qsNLTBCnDOtmMC+EBX6eODZJ6g3aa5UnHAxskUCNDM3taBLQ+fIF3u+LZDeGdi\ny0Jv/xsEMHsgN9IiMwKWBSsLwHSnMDeaFT9PgwIDAQABAoIBAQCKoRea49wzD5sI\nPzvNdJCsN7R5rt+liNtqDUHgRbGAi76QIL9REi/d5HYE20ES9eNY5+5fclMKvhdc\n5O/izCag70R/Mm7GIKwsXMy3pTNzmh9jNPcA2Q2lxdNMkitW/0JbYfdYrB5fSg2Z\nkRhUIVXQXBG8dnh3ZCaKrdiNQjLQugcWdkwREhe4gObftBFppbERSUVPVplHwt2t\nsCcsrc6O5KkIvdU1wFzGr1bWZ0mOrw8dL9kopbA1q+lSF4dHjeBLxLrV9bY4dpBl\nsKf4EG048KHJgM80Cco4AwVBjTQDkY/0OHteTLjLh0Z9HelpEX1yBH7sBuDRCHpO\nHiIGuupZAoGBANZDCy7Ehz2AFSew94bHNaKGeUSthzVS76PxAgOod2c3sh4vEmYC\n//OtDZLu3W/xfJnqJC2qEb+vRexDkSwytNNxVHC5w7MW10pTUI51w5FB62s7okqG\n+9i6MbPMk9mT42/AlsXASs5PIK/EkQkm1hbJfzVx0QMIX804gr8crslPAoGBAMmQ\nFtgYYU1HRT8a1OP6jpwxorZ+FQVLtRmvegKBr1ybC/nODs/KQI5ol9oPYlucGLuP\ndkYQEpJjaCItm2a4OsuHfK61VEIchOTrv/oxNcmY7SSRXshKrqTib2w5sfnR4WYW\nx0F47MnUXs8sl1CX5iSKoZJhXWLSeexxD7VaNOGNAoGBAJeEV78d2WljTxJ/cbuM\n2l/xaoZnlErgOHkdsMf3dWC3oSz5KrCbBHdEdGnoow1Ln0qUqjrknqKIBxF6AopX\n3Un9RbJlm3/k8iAsZLYpj0AEdr+hLzY22Jg9q3IzhIaDr31SmwyC3COjD0Fc5xeq\nsBDzMxMPRrg3TtAoW0Vcujm/AoGARRVLnxkMEG6C/1P074Zq5oHkoOOp1LzT/0+z\nY7SLJBRIEIBddz58zdJvaV+oeHmRyIctJGpR0zaa9EvpXVV7YVK4mzCvBlG8ArIC\nhH/lTYlKjiP89m0SWpT5V4CWzWbv+AuKk5gcoDhXnm5MFmVZjeCt6/vPBBXbj/xY\nQ/H8+ekCgYB36iuoWdihQDPb0wP3iUCs3/nfZjX7huVCon1MMXXvyKZS7lvgkUp2\nrhyV2A/QcaxW9f/hFyAgzj/e16de8ypy/CoSsRkBdIsZlRs9SUw3mX9a7SBMC9Le\nLU1aPXAqPdcBNIlBFEgLt6A18ZYD3wwdH6F+Mqocge8WljnTBVrt+g==\n-----END RSA PRIVATE KEY-----\n",
              "verification_key": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAqLM8nKKL6NcKHCk4/jgc\naSFbz6APw7pbLTHb8nqezmTCs/R0spsXoUrwRuPrtBwwkzjc3SfGX6Lq2MouBa0F\nJMlw+o/Iq/+JHDnnH00rOjlBg62y5bL6ABBlKn0yh9HqnL5cwOArtd3J2xP87PEM\nykyR5ag1CfiVjwexOH1NgDUmw8pZ1kwILwtWmpDxFIB32fhaCMCcSXOvyFZJDOhj\n/IM8R2mAUNOz8vSmcCOWb/BLxjcjg5qsNLTBCnDOtmMC+EBX6eODZJ6g3aa5UnHA\nxskUCNDM3taBLQ+fIF3u+LZDeGdiy0Jv/xsEMHsgN9IiMwKWBSsLwHSnMDeaFT9P\ngwIDAQAB\n-----END PUBLIC KEY-----\n"
            },
            "user": {
              "authorities": [
                "openid",
                "scim.me",
                "password.write",
                "uaa.user",
                "profile",
                "roles",
                "user_attributes",
                "bosh.admin",
                "bosh.read",
                "bosh.*.admin",
                "bosh.*.read",
                "clients.read",
                "clients.write"
              ]
            },
            "clients": {
              "bosh_cli": {
                "authorized-grant-types": "password,refresh_token",
                "override": true,
                "scope": "openid,bosh.admin,bosh.read,bosh.*.admin,bosh.*.read",
                "authorities": "uaa.none",
                "refresh-token-validity": 86400,
                "access-token-validity": 600,
                "secret": "",
                "allowedproviders": null
              },
              "ops_manager": {
                "authorized-grant-types": "client_credentials",
                "override": true,
                "scope": "",
                "authorities": "bosh.admin",
                "refresh-token-validity": 86400,
                "access-token-validity": 600,
                "secret": "blank-password"
              },
              "login": {
                "authorized-grant-types": "password,authorization_code",
                "autoapprove": true,
                "override": true,
                "scope": "bosh.admin,scim.write,clients.write,scim.read,clients.read",
                "authorities": "",
                "refresh-token-validity": 86400,
                "access-token-validity": 600,
                "secret": "uaa-login-client-password"
              }
            },
            "scim": {
              "users": [
                "director|director-password|bosh.admin",
                "admin|blank-password|bosh.admin,scim.write,clients.write,scim.read,clients.read"
              ]
            }
          },
          "uaadb": {
            "address": "127.0.0.1",
            "db_scheme": "postgresql",
            "port": 5432,
            "databases": [
              {
                "name": "uaa",
                "tag": "uaa"
              }
            ],
            "roles": [
              {
                "name": "postgres",
                "password": "postgres-password",
                "tag": "admin"
              }
            ]
          },
          "vcenter": {
            "address": "192.168.163.131",
            "user": "user",
            "password": "password",
            "datacenters": [
              {
                "name": "vsphere-datacenter",
                "vm_folder": "pivotal_cf_vms_test-installation-guid",
                "template_folder": "pivotal_cf_templates_test-installation-guid",
                "disk_path": "pivotal_cf_disk_test-installation-guid",
                "allow_mixed_datastores": true,
                "datastore_pattern": "^(vsphere\\-datastore)$",
                "persistent_datastore_pattern": "^(vsphere\\-datastore)$",
                "clusters": [
                  {
                    "vsphere-cluster": {
                    }
                  }
                ]
              }
            ]
          }
        }
      }
    ],
    "cloud_provider": {
      "template": {
        "name": "vsphere_cpi",
        "release": "bosh-vsphere-cpi"
      },
      "mbus": "https://vcap:agent-password@192.168.163.3:6868",
      "properties": {
        "agent": {
          "mbus": "https://vcap:agent-password@0.0.0.0:6868"
        },
        "blobstore": {
          "provider": "local",
          "path": "/var/vcap/micro_bosh/data/cache"
        },
        "ntp": [
          "us.pool.ntp.org"
        ],
        "vcenter": {
          "address": "192.168.163.131",
          "user": "user",
          "password": "password",
          "datacenters": [
            {
              "name": "vsphere-datacenter",
              "vm_folder": "pivotal_cf_vms_test-installation-guid",
              "template_folder": "pivotal_cf_templates_test-installation-guid",
              "disk_path": "pivotal_cf_disk_test-installation-guid",
              "allow_mixed_datastores": true,
              "datastore_pattern": "^(vsphere\\-datastore)$",
              "persistent_datastore_pattern": "^(vsphere\\-datastore)$",
              "clusters": [
                {
                  "vsphere-cluster": {
                  }
                }
              ]
            }
          ]
        },
        "env": {
        }
      }
    }
  }
}
```

#### HTTP Request

`GET /api/v0/staged/director/manifest`

Allows you to generate a BOSH director manifest. 

<aside class="warning">
Using this manifest outside OpsManager will prevent you from using OpsManager in the future.
</aside>



##Deployed BOSH Director
###Getting a list of available credentials


```shell
curl "https://example.com/api/v0/deployed/director/credentials" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "credential_ids": [
    "vm_credentials",
    "agent_credentials",
    "registry_credentials",
    "director_credentials",
    "nats_credentials",
    "redis_credentials",
    "postgres_credentials",
    "blobstore_credentials",
    "health_monitor_credentials",
    "uaa_admin_user_credentials",
    "uaa_login_client_credentials",
    "uaa_jwt_key"
  ]
}
```

#### HTTP Request

`GET /api/v0/deployed/director/credentials`

Use this endpoint to discover available types of credentials.

### Listing an rsa_key credential

```shell
curl "https://example.com/api/v0/deployed/director/credentials/uaa_jwt_key" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "credential": {
    "type": "rsa_cert_credentials",
    "value": {
      "private_key_pem": "-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEAqLM8nKKL6NcKHCk4/jgcaSFbz6APw7pbLTHb8nqezmTCs/R0\nspsXoUrwRuPrtBwwkzjc3SfGX6Lq2MouBa0FJMlw+o/Iq/+JHDnnH00rOjlBg62y\n5bL6ABBlKn0yh9HqnL5cwOArtd3J2xP87PEMykyR5ag1CfiVjwexOH1NgDUmw8pZ\n1kwILwtWmpDxFIB32fhaCMCcSXOvyFZJDOhj/IM8R2mAUNOz8vSmcCOWb/BLxjcj\ng5qsNLTBCnDOtmMC+EBX6eODZJ6g3aa5UnHAxskUCNDM3taBLQ+fIF3u+LZDeGdi\ny0Jv/xsEMHsgN9IiMwKWBSsLwHSnMDeaFT9PgwIDAQABAoIBAQCKoRea49wzD5sI\nPzvNdJCsN7R5rt+liNtqDUHgRbGAi76QIL9REi/d5HYE20ES9eNY5+5fclMKvhdc\n5O/izCag70R/Mm7GIKwsXMy3pTNzmh9jNPcA2Q2lxdNMkitW/0JbYfdYrB5fSg2Z\nkRhUIVXQXBG8dnh3ZCaKrdiNQjLQugcWdkwREhe4gObftBFppbERSUVPVplHwt2t\nsCcsrc6O5KkIvdU1wFzGr1bWZ0mOrw8dL9kopbA1q+lSF4dHjeBLxLrV9bY4dpBl\nsKf4EG048KHJgM80Cco4AwVBjTQDkY/0OHteTLjLh0Z9HelpEX1yBH7sBuDRCHpO\nHiIGuupZAoGBANZDCy7Ehz2AFSew94bHNaKGeUSthzVS76PxAgOod2c3sh4vEmYC\n//OtDZLu3W/xfJnqJC2qEb+vRexDkSwytNNxVHC5w7MW10pTUI51w5FB62s7okqG\n+9i6MbPMk9mT42/AlsXASs5PIK/EkQkm1hbJfzVx0QMIX804gr8crslPAoGBAMmQ\nFtgYYU1HRT8a1OP6jpwxorZ+FQVLtRmvegKBr1ybC/nODs/KQI5ol9oPYlucGLuP\ndkYQEpJjaCItm2a4OsuHfK61VEIchOTrv/oxNcmY7SSRXshKrqTib2w5sfnR4WYW\nx0F47MnUXs8sl1CX5iSKoZJhXWLSeexxD7VaNOGNAoGBAJeEV78d2WljTxJ/cbuM\n2l/xaoZnlErgOHkdsMf3dWC3oSz5KrCbBHdEdGnoow1Ln0qUqjrknqKIBxF6AopX\n3Un9RbJlm3/k8iAsZLYpj0AEdr+hLzY22Jg9q3IzhIaDr31SmwyC3COjD0Fc5xeq\nsBDzMxMPRrg3TtAoW0Vcujm/AoGARRVLnxkMEG6C/1P074Zq5oHkoOOp1LzT/0+z\nY7SLJBRIEIBddz58zdJvaV+oeHmRyIctJGpR0zaa9EvpXVV7YVK4mzCvBlG8ArIC\nhH/lTYlKjiP89m0SWpT5V4CWzWbv+AuKk5gcoDhXnm5MFmVZjeCt6/vPBBXbj/xY\nQ/H8+ekCgYB36iuoWdihQDPb0wP3iUCs3/nfZjX7huVCon1MMXXvyKZS7lvgkUp2\nrhyV2A/QcaxW9f/hFyAgzj/e16de8ypy/CoSsRkBdIsZlRs9SUw3mX9a7SBMC9Le\nLU1aPXAqPdcBNIlBFEgLt6A18ZYD3wwdH6F+Mqocge8WljnTBVrt+g==\n-----END RSA PRIVATE KEY-----\n",
      "public_key_pem": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAqLM8nKKL6NcKHCk4/jgc\naSFbz6APw7pbLTHb8nqezmTCs/R0spsXoUrwRuPrtBwwkzjc3SfGX6Lq2MouBa0F\nJMlw+o/Iq/+JHDnnH00rOjlBg62y5bL6ABBlKn0yh9HqnL5cwOArtd3J2xP87PEM\nykyR5ag1CfiVjwexOH1NgDUmw8pZ1kwILwtWmpDxFIB32fhaCMCcSXOvyFZJDOhj\n/IM8R2mAUNOz8vSmcCOWb/BLxjcjg5qsNLTBCnDOtmMC+EBX6eODZJ6g3aa5UnHA\nxskUCNDM3taBLQ+fIF3u+LZDeGdiy0Jv/xsEMHsgN9IiMwKWBSsLwHSnMDeaFT9P\ngwIDAQAB\n-----END PUBLIC KEY-----\n"
    }
  }
}
```

#### HTTP Request

`GET /api/v0/deployed/director/credentials/:id`


### Listing a simple_credential

```shell
curl "https://example.com/api/v0/deployed/director/credentials/agent_credentials" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "credential": {
    "type": "simple_credentials",
    "value": {
      "identity": "vcap",
      "password": "agent-password"
    }
  }
}
```


#### HTTP Request

`GET /api/v0/deployed/director/credentials/:id`



###Viewing status
###Fetching a manifest

```shell
curl "https://example.com/api/v0/deployed/director/manifest" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "name": "p-bosh-installation-name",
  "releases": [
    {
      "name": "bosh",
      "url": "file:///tmp/FactoryGirl-Tempest::PackageLibrary-FACTORY20160412-125-20sejo/internal_releases/bosh"
    },
    {
      "name": "bosh-vsphere-cpi",
      "url": "file:///tmp/FactoryGirl-Tempest::PackageLibrary-FACTORY20160412-125-20sejo/internal_releases/cpi"
    },
    {
      "name": "uaa",
      "url": "file:///tmp/FactoryGirl-Tempest::PackageLibrary-FACTORY20160412-125-20sejo/internal_releases/uaa"
    }
  ],
  "networks": [
    {
      "name": "default",
      "type": "manual",
      "subnets": [
        {
          "netmask": "255.255.255.0",
          "dns": [
            "192.168.163.1"
          ],
          "gateway": "192.168.163.2",
          "range": "192.168.163.0/24",
          "cloud_properties": {
            "name": "vsphere-network"
          }
        }
      ]
    }
  ],
  "disk_pools": [
    {
      "name": "director_disk_pool",
      "disk_size": 51200,
      "cloud_properties": {
        "type": "thin"
      }
    }
  ],
  "resource_pools": [
    {
      "name": "director_resource_pool",
      "network": "default",
      "stemcell": {
        "url": "file:///tmp/FactoryGirl-Tempest::PackageLibrary-FACTORY20160412-125-20sejo/stemcells/bosh-stemcell-3215-vsphere-esxi-ubuntu-trusty-go_agent.tgz"
      },
      "cloud_properties": {
        "cpu": 2,
        "disk": 51200,
        "ram": 4096,
        "datacenters": [
          {
            "name": "vsphere-datacenter",
            "clusters": [
              {
                "vsphere-cluster": {
                }
              }
            ]
          }
        ]
      },
      "env": {
        "bosh": {
          "password": "$6$vm-salt-12345687$nWtgnl1OMN2ZYBP4KhIrjuSAKJ968h43goOBQBBkaGX9vJlK2DL5QanzPSppfEogEIF7MzxFHR.6xLKVe1olr."
        }
      }
    }
  ],
  "jobs": [
    {
      "name": "bosh",
      "instances": 1,
      "templates": [
        {
          "name": "postgres",
          "release": "bosh"
        },
        {
          "name": "nats",
          "release": "bosh"
        },
        {
          "name": "redis",
          "release": "bosh"
        },
        {
          "name": "director",
          "release": "bosh"
        },
        {
          "name": "health_monitor",
          "release": "bosh"
        },
        {
          "name": "uaa",
          "release": "uaa"
        },
        {
          "name": "vsphere_cpi",
          "release": "bosh-vsphere-cpi"
        },
        {
          "name": "blobstore",
          "release": "bosh"
        }
      ],
      "resource_pool": "director_resource_pool",
      "persistent_disk_pool": "director_disk_pool",
      "networks": [
        {
          "name": "default",
          "static_ips": [
            "192.168.163.3"
          ],
          "default": [
            "dns",
            "gateway"
          ]
        }
      ],
      "properties": {
        "env": {
        },
        "nats": {
          "address": "127.0.0.1",
          "user": "nats",
          "password": "nats-password"
        },
        "redis": {
          "listen_address": "127.0.0.1",
          "address": "127.0.0.1",
          "password": "redis-password"
        },
        "postgres": {
          "host": "127.0.0.1",
          "user": "postgres",
          "password": "postgres-password",
          "database": "bosh",
          "additional_databases": [
            "uaa"
          ],
          "adapter": "postgres"
        },
        "blobstore": {
          "address": "192.168.163.3",
          "port": 25250,
          "provider": "dav",
          "director": {
            "user": "blobstore",
            "password": "blobstore-password"
          },
          "agent": {
            "user": "blobstore",
            "password": "blobstore-password"
          }
        },
        "director": {
          "address": "192.168.163.3",
          "name": "p-bosh-installation-name",
          "cpi_job": "vsphere_cpi",
          "user_management": {
            "provider": "uaa",
            "uaa": {
              "url": "https://192.168.163.3:8443",
              "public_key": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAqLM8nKKL6NcKHCk4/jgc\naSFbz6APw7pbLTHb8nqezmTCs/R0spsXoUrwRuPrtBwwkzjc3SfGX6Lq2MouBa0F\nJMlw+o/Iq/+JHDnnH00rOjlBg62y5bL6ABBlKn0yh9HqnL5cwOArtd3J2xP87PEM\nykyR5ag1CfiVjwexOH1NgDUmw8pZ1kwILwtWmpDxFIB32fhaCMCcSXOvyFZJDOhj\n/IM8R2mAUNOz8vSmcCOWb/BLxjcjg5qsNLTBCnDOtmMC+EBX6eODZJ6g3aa5UnHA\nxskUCNDM3taBLQ+fIF3u+LZDeGdiy0Jv/xsEMHsgN9IiMwKWBSsLwHSnMDeaFT9P\ngwIDAQAB\n-----END PUBLIC KEY-----\n"
            }
          },
          "max_threads": 3,
          "db": {
            "host": "127.0.0.1",
            "user": "postgres",
            "password": "postgres-password",
            "database": "bosh",
            "additional_databases": [
              "uaa"
            ],
            "adapter": "postgres"
          },
          "trusted_certs": null,
          "ssl": {
            "key": "-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEAqLM8nKKL6NcKHCk4/jgcaSFbz6APw7pbLTHb8nqezmTCs/R0\nspsXoUrwRuPrtBwwkzjc3SfGX6Lq2MouBa0FJMlw+o/Iq/+JHDnnH00rOjlBg62y\n5bL6ABBlKn0yh9HqnL5cwOArtd3J2xP87PEMykyR5ag1CfiVjwexOH1NgDUmw8pZ\n1kwILwtWmpDxFIB32fhaCMCcSXOvyFZJDOhj/IM8R2mAUNOz8vSmcCOWb/BLxjcj\ng5qsNLTBCnDOtmMC+EBX6eODZJ6g3aa5UnHAxskUCNDM3taBLQ+fIF3u+LZDeGdi\ny0Jv/xsEMHsgN9IiMwKWBSsLwHSnMDeaFT9PgwIDAQABAoIBAQCKoRea49wzD5sI\nPzvNdJCsN7R5rt+liNtqDUHgRbGAi76QIL9REi/d5HYE20ES9eNY5+5fclMKvhdc\n5O/izCag70R/Mm7GIKwsXMy3pTNzmh9jNPcA2Q2lxdNMkitW/0JbYfdYrB5fSg2Z\nkRhUIVXQXBG8dnh3ZCaKrdiNQjLQugcWdkwREhe4gObftBFppbERSUVPVplHwt2t\nsCcsrc6O5KkIvdU1wFzGr1bWZ0mOrw8dL9kopbA1q+lSF4dHjeBLxLrV9bY4dpBl\nsKf4EG048KHJgM80Cco4AwVBjTQDkY/0OHteTLjLh0Z9HelpEX1yBH7sBuDRCHpO\nHiIGuupZAoGBANZDCy7Ehz2AFSew94bHNaKGeUSthzVS76PxAgOod2c3sh4vEmYC\n//OtDZLu3W/xfJnqJC2qEb+vRexDkSwytNNxVHC5w7MW10pTUI51w5FB62s7okqG\n+9i6MbPMk9mT42/AlsXASs5PIK/EkQkm1hbJfzVx0QMIX804gr8crslPAoGBAMmQ\nFtgYYU1HRT8a1OP6jpwxorZ+FQVLtRmvegKBr1ybC/nODs/KQI5ol9oPYlucGLuP\ndkYQEpJjaCItm2a4OsuHfK61VEIchOTrv/oxNcmY7SSRXshKrqTib2w5sfnR4WYW\nx0F47MnUXs8sl1CX5iSKoZJhXWLSeexxD7VaNOGNAoGBAJeEV78d2WljTxJ/cbuM\n2l/xaoZnlErgOHkdsMf3dWC3oSz5KrCbBHdEdGnoow1Ln0qUqjrknqKIBxF6AopX\n3Un9RbJlm3/k8iAsZLYpj0AEdr+hLzY22Jg9q3IzhIaDr31SmwyC3COjD0Fc5xeq\nsBDzMxMPRrg3TtAoW0Vcujm/AoGARRVLnxkMEG6C/1P074Zq5oHkoOOp1LzT/0+z\nY7SLJBRIEIBddz58zdJvaV+oeHmRyIctJGpR0zaa9EvpXVV7YVK4mzCvBlG8ArIC\nhH/lTYlKjiP89m0SWpT5V4CWzWbv+AuKk5gcoDhXnm5MFmVZjeCt6/vPBBXbj/xY\nQ/H8+ekCgYB36iuoWdihQDPb0wP3iUCs3/nfZjX7huVCon1MMXXvyKZS7lvgkUp2\nrhyV2A/QcaxW9f/hFyAgzj/e16de8ypy/CoSsRkBdIsZlRs9SUw3mX9a7SBMC9Le\nLU1aPXAqPdcBNIlBFEgLt6A18ZYD3wwdH6F+Mqocge8WljnTBVrt+g==\n-----END RSA PRIVATE KEY-----\n",
            "cert": "-----BEGIN CERTIFICATE-----\nMIIC3zCCAcegAwIBAgIBADANBgkqhkiG9w0BAQUFADAzMQswCQYDVQQGEwJVUzEQ\nMA4GA1UECgwHUGl2b3RhbDESMBAGA1UEAwwJc29tZS1uYW1lMB4XDTEzMDcwOTIy\nMTI1MVoXDTE1MDcwOTIyMTI1MVowMzELMAkGA1UEBhMCVVMxEDAOBgNVBAoMB1Bp\ndm90YWwxEjAQBgNVBAMMCXNvbWUtbmFtZTCCASIwDQYJKoZIhvcNAQEBBQADggEP\nADCCAQoCggEBALny6vikcrf/3Do/Nq2Sh/ji8d8dhACUIOaf+ml2cWuRKuP6TcQC\nohbLBHlbm3l5QqD/lvz1EaSy168SVPIsy34sAcioYv+7oOIAHqS4gCVEb7AQrsKb\nZIFUmp7J9qCmkJtyImvViQUUJrWOSaQi3eCAh8uSHyelBNPdaHnj0k8WufKp6hzK\npAr6Xv8OgFSwjD0+XROiTyRpvsQoTm8/XtdhAFpwjTnqR9gxlXqzDqmBRzWduO/M\nyctwF9gtggUp31USmqo5fVC+nr1wh6a/JlbUETtcRhk8jR/FnAVHLSJC4+FZqhmK\nDcemvIfEJaKCNqhvytRLI01l+0p1pm8cqxUCAwEAATANBgkqhkiG9w0BAQUFAAOC\nAQEAru0hKd5gd1WDS6AUrIa8AYCWUrGHMd5P63FWB0KyUnfIDTX4tegHTF+olOxA\nkrR4IRVgFbu3u0pnURFn2N1Et4pZwvW9PEamwkIGHEpYmASOiUZqvrthx/WpUaeu\n+xQIWa1S140v4wa/27UTakAuR+GnA6StJSIRBEBa7hafqpeLGPugZVWRtY3m/OIF\nLICs2U2X8P86RMUWgdtM9//x3t6O7IJzhrSKRkZDmSWAv6EbS/aTpXOPpJFpJtT8\n0aETgAhauKhyp6CeajL3Nc3FfoIONK427VbfIGKJ1Qw7OwTA4N0VPpETiGN7KrfD\nU4mSCEKQ0cIypQAm9rkPboHfwg==\n-----END CERTIFICATE-----\n"
          }
        },
        "hm": {
          "director_account": {
            "ca_cert": "-----BEGIN CERTIFICATE-----\nMIIC+zCCAeOgAwIBAgIBADANBgkqhkiG9w0BAQUFADAfMQswCQYDVQQGEwJVUzEQ\nMA4GA1UECgwHUGl2b3RhbDAeFw0xNjA0MTExNTE0MzRaFw0yMDA0MTIxNTE0MzRa\nMB8xCzAJBgNVBAYTAlVTMRAwDgYDVQQKDAdQaXZvdGFsMIIBIjANBgkqhkiG9w0B\nAQEFAAOCAQ8AMIIBCgKCAQEApJOxyQUh/OwHVnWxjf39aNQ61DPyFozobajQarwp\n6dHYWwZYj2nNtgEWmkrpbagTU0rwP8QzpUUpf3NUCmPwLEerLpQVEg4LTIMpQyKR\nXPjEWrU0QB08QOZwxP2H2x2IS6Qn9obKuBtExwhTvaCUoDqZhaZycqYIZ6yV4ppN\nnKEP5HcDXpj02ZA4lvkCh/mDFxYJUjJrLnK+zMfALOcX7QohL/iOENEkG0qmaYqI\n/e7tcAw2O7h95sGlRkcUQ2JghhfbwZQkAR6YA6k2EYPdnkMLhoJOwlL+oexJubVS\nlA6CzNVOcuG4Bec3Zg43oJtR1sV7z8a8Tgri+F2NYhsccQIDAQABo0IwQDAdBgNV\nHQ4EFgQUkP10tFC5mzdcncEK0NWBQXTiTg4wDwYDVR0TAQH/BAUwAwEB/zAOBgNV\nHQ8BAf8EBAMCAQYwDQYJKoZIhvcNAQEFBQADggEBAHefgUo4C+kG4EOzXPcMwKY3\nCtBIUq+zVfLEuu7QBxJ0LEspoBt080MLeKcrkrSdDG6FZeNSTvSbYR5adWoEhK3/\nxB2RraY/780oE6Paptqo4f4H38+rPj18zmzsMiSIXkimoLmsAXe7ZGV72Zsmz9hm\nYDaSuSBxMlSUWytmqLacajgeGFdyAs2KRJamTHqeGSxLGyqpDWw023aLqSOXSaWg\nD2jUfGNd7PwpsBTa7jwlDEVapUqHdQmNbMaZqon40dlTwEi0JOp4zWtg8g8AFNjG\n7B1T481JoeZuPVMsD+A/jgnerwKxm9+hzsyLLGVnsafksB5S6vbmdjKt0nhHHzU=\n-----END CERTIFICATE-----\n",
            "user": "health_monitor",
            "password": "health_monitor"
          },
          "resurrector_enabled": false,
          "pagerduty_enabled": false,
          "pagerduty": {
            "service_key": null,
            "http_proxy": null
          },
          "email_notifications": false,
          "email_recipients": [

          ],
          "smtp": {
            "from": null,
            "host": null,
            "port": 25,
            "domain": null,
            "tls": false,
            "user": null,
            "password": null
          }
        },
        "agent": {
          "mbus": "nats://nats:nats-password@192.168.163.3:4222"
        },
        "ntp": [
          "us.pool.ntp.org"
        ],
        "login": {
          "protocol": "https",
          "branding": {
            "company_name": "Pivotal",
            "product_logo": "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",
            "square_logo": "iVBORw0KGgoAAAANSUhEUgAAAGwAAABsCAYAAACPZlfNAAAAAXNSR0IArs4c6QAABYtJREFUeAHtnVtsFFUYx7/d3ruWotUKVIkNaCw02YgJGBRTMd4CokUejD4QH4gxQcIDeHnBmPjkhSghUYLGe3ywPtAHNCo0QgkWwi2tXG2V1kIpLXTbLt1tS9dzlmzSJssZhv32zDk7/2km2znn7Pd9+/vt2Z2dmW0D9Obat4gCiwiLBQQSLflSViAQeN6Can1fYiJBFPQ9BcsAQBiEWUbAsnIxwyDMMgKWlYsZBmGWEbCsXMwwCLOMgGXlYoZBmGUELCsXMwzCLCNgWbmYYRBmGQHLysUMgzDLCFhWLmYYhFlGwLJyMcMgzDIClpWLGQZhlhGwrFzMMAizjIBl5WKGQZhlBCwrV1xbb96y59V1VFJQmLawQNrWa43x8XEaHo1fW+Oj1H8lSqf6eulEbw+dvNhLvcNDinvb0WWksAdm3UWhwiJ2gt2RAWo80UY7jrdSU8cZGrt6lT1HtgMaKSxbD7qqfDq99tAjyTUSG6FP9v1BH+3dTUPxeLZSssf17U5HeXEJbXr8aerY+A6tf7iOxFeu2OFmI6BvhaVgVoRCtHl5PTW8/AoV5xekmo299b2wlJn6+WFqWrOWKkpDqSYjbyFskpZFs++hL1e9NKnFvF+t3OmQOwzdkcgUmnnBABXm5Ys1j8qKisVadFPvS8tramn1goX09eEDU+KbsmGlsMbjbbT6x++UDOVORGXoFppXOYMerLqbVsyrpcWzqykYdH5R+fjZlcnd/8sjV5Q5vOh0rt6LqhhyJsQ3uC+ID8ry89aHYtf90W1bKLzlffr19EnH6HIP8oXasOM4LwbkrLB0MP+6cJ6e+eoz+vTP5nTdU9peDC+Ysm3Khq+ESehy5r3e2ECHu7uUDuqq59Id4iXVtMV3wqSACSHt3V2/KF3I97qayjuVY7zo9KUwCfq3M6coNjamZD6zrFzZ70Wnb4XFxseoK3JZyXzWtGnKfi86fStMwu6LRpXMZ5RBmBKQ7k75XqZa8gLmPZ/Nq0hFkLnvttJSZUT5Oc60xbfC5CGs6lsrlD56hgaV/V50+lbYkuo5VFygPp3SMwxhXjwp0+bcsGRp2vZU48TEBB09153aNObWlzNMHo1/6r4apYTmsx10MTqsHONFp5VH6zMBtWbhYtq6YpVjiJ/ajjmO8WKAL4QFxamWZffPT1678dicex05D4jTKj8cO+Q4zosBOSXs7bonktci5ovjgPIUye3ieo3wzKrk+TC5faPLGz83On6ovtFY3ONySth7Ty67qbPMk6Hu+edv+vzg/slNRv3uy52O6xk40HWW6r/94nrdRrTn1AzLhOju9tP03DfbKTo6mkmYrN/X98L6xQHgTb/vpG0t+5LnybJOPMMEvhXWOXCJvj9yiD7Yu4sGRkYyxKjv7r4RJi+Na+05Rwf/66SG1qO0v/NffZQZM+WUsI07d1BC/MTE144GYzHxJYcYDYq1vb/f8WQlI9OshsopYZubm7IKy4Tg2K03wYKLGiDMBSwThkKYCRZc1ABhLmCZMBTCTLDgogYIcwHLhKEQZoIFFzVAmAtYJgyFMBMsuKgBwlzAMmEohJlgwUUNEOYClglDIcwECy5qgDAXsEwYCmEmWHBRA4S5gGXCUAgzwYKLGow84yyvuyhR/GW19kt9Lh5ibg01UtjS7VtzizLjo8FLIiNMHaEgTAdlxhwQxghTRygI00GZMQeEMcLUEQrCdFBmzAFhjDB1hIIwHZQZc0AYI0wdoSBMB2XGHBDGCFNHKAjTQZkxB4QxwtQRCsJ0UGbMAWGMMHWEgjAdlBlzQBgjTB2hIEwHZcYcEMYIU0coCNNBmTEHhDHC1BEKwnRQZswBYYwwdYSCMB2UGXNAGCNMHaEgTAdlxhziUu1Ei8M/+WFMh1CZEUi0/A+j7hNSB5Wo2wAAAABJRU5ErkJggg==",
            "footer_legal_text": "©2016 Pivotal Software, Inc. All Rights Reserved",
            "footer_links": null
          },
          "saml": {
          }
        },
        "uaa": {
          "admin": {
            "client_secret": "blank-password"
          },
          "disableInternalAuth": false,
          "sslCertificate": "-----BEGIN CERTIFICATE-----\nMIIC3zCCAcegAwIBAgIBADANBgkqhkiG9w0BAQUFADAzMQswCQYDVQQGEwJVUzEQ\nMA4GA1UECgwHUGl2b3RhbDESMBAGA1UEAwwJc29tZS1uYW1lMB4XDTEzMDcwOTIy\nMTI1MVoXDTE1MDcwOTIyMTI1MVowMzELMAkGA1UEBhMCVVMxEDAOBgNVBAoMB1Bp\ndm90YWwxEjAQBgNVBAMMCXNvbWUtbmFtZTCCASIwDQYJKoZIhvcNAQEBBQADggEP\nADCCAQoCggEBALny6vikcrf/3Do/Nq2Sh/ji8d8dhACUIOaf+ml2cWuRKuP6TcQC\nohbLBHlbm3l5QqD/lvz1EaSy168SVPIsy34sAcioYv+7oOIAHqS4gCVEb7AQrsKb\nZIFUmp7J9qCmkJtyImvViQUUJrWOSaQi3eCAh8uSHyelBNPdaHnj0k8WufKp6hzK\npAr6Xv8OgFSwjD0+XROiTyRpvsQoTm8/XtdhAFpwjTnqR9gxlXqzDqmBRzWduO/M\nyctwF9gtggUp31USmqo5fVC+nr1wh6a/JlbUETtcRhk8jR/FnAVHLSJC4+FZqhmK\nDcemvIfEJaKCNqhvytRLI01l+0p1pm8cqxUCAwEAATANBgkqhkiG9w0BAQUFAAOC\nAQEAru0hKd5gd1WDS6AUrIa8AYCWUrGHMd5P63FWB0KyUnfIDTX4tegHTF+olOxA\nkrR4IRVgFbu3u0pnURFn2N1Et4pZwvW9PEamwkIGHEpYmASOiUZqvrthx/WpUaeu\n+xQIWa1S140v4wa/27UTakAuR+GnA6StJSIRBEBa7hafqpeLGPugZVWRtY3m/OIF\nLICs2U2X8P86RMUWgdtM9//x3t6O7IJzhrSKRkZDmSWAv6EbS/aTpXOPpJFpJtT8\n0aETgAhauKhyp6CeajL3Nc3FfoIONK427VbfIGKJ1Qw7OwTA4N0VPpETiGN7KrfD\nU4mSCEKQ0cIypQAm9rkPboHfwg==\n-----END CERTIFICATE-----\n",
          "sslPrivateKey": "-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEAqLM8nKKL6NcKHCk4/jgcaSFbz6APw7pbLTHb8nqezmTCs/R0\nspsXoUrwRuPrtBwwkzjc3SfGX6Lq2MouBa0FJMlw+o/Iq/+JHDnnH00rOjlBg62y\n5bL6ABBlKn0yh9HqnL5cwOArtd3J2xP87PEMykyR5ag1CfiVjwexOH1NgDUmw8pZ\n1kwILwtWmpDxFIB32fhaCMCcSXOvyFZJDOhj/IM8R2mAUNOz8vSmcCOWb/BLxjcj\ng5qsNLTBCnDOtmMC+EBX6eODZJ6g3aa5UnHAxskUCNDM3taBLQ+fIF3u+LZDeGdi\ny0Jv/xsEMHsgN9IiMwKWBSsLwHSnMDeaFT9PgwIDAQABAoIBAQCKoRea49wzD5sI\nPzvNdJCsN7R5rt+liNtqDUHgRbGAi76QIL9REi/d5HYE20ES9eNY5+5fclMKvhdc\n5O/izCag70R/Mm7GIKwsXMy3pTNzmh9jNPcA2Q2lxdNMkitW/0JbYfdYrB5fSg2Z\nkRhUIVXQXBG8dnh3ZCaKrdiNQjLQugcWdkwREhe4gObftBFppbERSUVPVplHwt2t\nsCcsrc6O5KkIvdU1wFzGr1bWZ0mOrw8dL9kopbA1q+lSF4dHjeBLxLrV9bY4dpBl\nsKf4EG048KHJgM80Cco4AwVBjTQDkY/0OHteTLjLh0Z9HelpEX1yBH7sBuDRCHpO\nHiIGuupZAoGBANZDCy7Ehz2AFSew94bHNaKGeUSthzVS76PxAgOod2c3sh4vEmYC\n//OtDZLu3W/xfJnqJC2qEb+vRexDkSwytNNxVHC5w7MW10pTUI51w5FB62s7okqG\n+9i6MbPMk9mT42/AlsXASs5PIK/EkQkm1hbJfzVx0QMIX804gr8crslPAoGBAMmQ\nFtgYYU1HRT8a1OP6jpwxorZ+FQVLtRmvegKBr1ybC/nODs/KQI5ol9oPYlucGLuP\ndkYQEpJjaCItm2a4OsuHfK61VEIchOTrv/oxNcmY7SSRXshKrqTib2w5sfnR4WYW\nx0F47MnUXs8sl1CX5iSKoZJhXWLSeexxD7VaNOGNAoGBAJeEV78d2WljTxJ/cbuM\n2l/xaoZnlErgOHkdsMf3dWC3oSz5KrCbBHdEdGnoow1Ln0qUqjrknqKIBxF6AopX\n3Un9RbJlm3/k8iAsZLYpj0AEdr+hLzY22Jg9q3IzhIaDr31SmwyC3COjD0Fc5xeq\nsBDzMxMPRrg3TtAoW0Vcujm/AoGARRVLnxkMEG6C/1P074Zq5oHkoOOp1LzT/0+z\nY7SLJBRIEIBddz58zdJvaV+oeHmRyIctJGpR0zaa9EvpXVV7YVK4mzCvBlG8ArIC\nhH/lTYlKjiP89m0SWpT5V4CWzWbv+AuKk5gcoDhXnm5MFmVZjeCt6/vPBBXbj/xY\nQ/H8+ekCgYB36iuoWdihQDPb0wP3iUCs3/nfZjX7huVCon1MMXXvyKZS7lvgkUp2\nrhyV2A/QcaxW9f/hFyAgzj/e16de8ypy/CoSsRkBdIsZlRs9SUw3mX9a7SBMC9Le\nLU1aPXAqPdcBNIlBFEgLt6A18ZYD3wwdH6F+Mqocge8WljnTBVrt+g==\n-----END RSA PRIVATE KEY-----\n",
          "require_https": false,
          "url": "https://192.168.163.3:8443",
          "jwt": {
            "signing_key": "-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEAqLM8nKKL6NcKHCk4/jgcaSFbz6APw7pbLTHb8nqezmTCs/R0\nspsXoUrwRuPrtBwwkzjc3SfGX6Lq2MouBa0FJMlw+o/Iq/+JHDnnH00rOjlBg62y\n5bL6ABBlKn0yh9HqnL5cwOArtd3J2xP87PEMykyR5ag1CfiVjwexOH1NgDUmw8pZ\n1kwILwtWmpDxFIB32fhaCMCcSXOvyFZJDOhj/IM8R2mAUNOz8vSmcCOWb/BLxjcj\ng5qsNLTBCnDOtmMC+EBX6eODZJ6g3aa5UnHAxskUCNDM3taBLQ+fIF3u+LZDeGdi\ny0Jv/xsEMHsgN9IiMwKWBSsLwHSnMDeaFT9PgwIDAQABAoIBAQCKoRea49wzD5sI\nPzvNdJCsN7R5rt+liNtqDUHgRbGAi76QIL9REi/d5HYE20ES9eNY5+5fclMKvhdc\n5O/izCag70R/Mm7GIKwsXMy3pTNzmh9jNPcA2Q2lxdNMkitW/0JbYfdYrB5fSg2Z\nkRhUIVXQXBG8dnh3ZCaKrdiNQjLQugcWdkwREhe4gObftBFppbERSUVPVplHwt2t\nsCcsrc6O5KkIvdU1wFzGr1bWZ0mOrw8dL9kopbA1q+lSF4dHjeBLxLrV9bY4dpBl\nsKf4EG048KHJgM80Cco4AwVBjTQDkY/0OHteTLjLh0Z9HelpEX1yBH7sBuDRCHpO\nHiIGuupZAoGBANZDCy7Ehz2AFSew94bHNaKGeUSthzVS76PxAgOod2c3sh4vEmYC\n//OtDZLu3W/xfJnqJC2qEb+vRexDkSwytNNxVHC5w7MW10pTUI51w5FB62s7okqG\n+9i6MbPMk9mT42/AlsXASs5PIK/EkQkm1hbJfzVx0QMIX804gr8crslPAoGBAMmQ\nFtgYYU1HRT8a1OP6jpwxorZ+FQVLtRmvegKBr1ybC/nODs/KQI5ol9oPYlucGLuP\ndkYQEpJjaCItm2a4OsuHfK61VEIchOTrv/oxNcmY7SSRXshKrqTib2w5sfnR4WYW\nx0F47MnUXs8sl1CX5iSKoZJhXWLSeexxD7VaNOGNAoGBAJeEV78d2WljTxJ/cbuM\n2l/xaoZnlErgOHkdsMf3dWC3oSz5KrCbBHdEdGnoow1Ln0qUqjrknqKIBxF6AopX\n3Un9RbJlm3/k8iAsZLYpj0AEdr+hLzY22Jg9q3IzhIaDr31SmwyC3COjD0Fc5xeq\nsBDzMxMPRrg3TtAoW0Vcujm/AoGARRVLnxkMEG6C/1P074Zq5oHkoOOp1LzT/0+z\nY7SLJBRIEIBddz58zdJvaV+oeHmRyIctJGpR0zaa9EvpXVV7YVK4mzCvBlG8ArIC\nhH/lTYlKjiP89m0SWpT5V4CWzWbv+AuKk5gcoDhXnm5MFmVZjeCt6/vPBBXbj/xY\nQ/H8+ekCgYB36iuoWdihQDPb0wP3iUCs3/nfZjX7huVCon1MMXXvyKZS7lvgkUp2\nrhyV2A/QcaxW9f/hFyAgzj/e16de8ypy/CoSsRkBdIsZlRs9SUw3mX9a7SBMC9Le\nLU1aPXAqPdcBNIlBFEgLt6A18ZYD3wwdH6F+Mqocge8WljnTBVrt+g==\n-----END RSA PRIVATE KEY-----\n",
            "verification_key": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAqLM8nKKL6NcKHCk4/jgc\naSFbz6APw7pbLTHb8nqezmTCs/R0spsXoUrwRuPrtBwwkzjc3SfGX6Lq2MouBa0F\nJMlw+o/Iq/+JHDnnH00rOjlBg62y5bL6ABBlKn0yh9HqnL5cwOArtd3J2xP87PEM\nykyR5ag1CfiVjwexOH1NgDUmw8pZ1kwILwtWmpDxFIB32fhaCMCcSXOvyFZJDOhj\n/IM8R2mAUNOz8vSmcCOWb/BLxjcjg5qsNLTBCnDOtmMC+EBX6eODZJ6g3aa5UnHA\nxskUCNDM3taBLQ+fIF3u+LZDeGdiy0Jv/xsEMHsgN9IiMwKWBSsLwHSnMDeaFT9P\ngwIDAQAB\n-----END PUBLIC KEY-----\n"
          },
          "user": {
            "authorities": [
              "openid",
              "scim.me",
              "password.write",
              "uaa.user",
              "profile",
              "roles",
              "user_attributes",
              "bosh.admin",
              "bosh.read",
              "bosh.*.admin",
              "bosh.*.read",
              "clients.read",
              "clients.write"
            ]
          },
          "clients": {
            "bosh_cli": {
              "authorized-grant-types": "password,refresh_token",
              "override": true,
              "scope": "openid,bosh.admin,bosh.read,bosh.*.admin,bosh.*.read",
              "authorities": "uaa.none",
              "refresh-token-validity": 86400,
              "access-token-validity": 600,
              "secret": "",
              "allowedproviders": null
            },
            "ops_manager": {
              "authorized-grant-types": "client_credentials",
              "override": true,
              "scope": "",
              "authorities": "bosh.admin",
              "refresh-token-validity": 86400,
              "access-token-validity": 600,
              "secret": "blank-password"
            },
            "login": {
              "authorized-grant-types": "password,authorization_code",
              "autoapprove": true,
              "override": true,
              "scope": "bosh.admin,scim.write,clients.write,scim.read,clients.read",
              "authorities": "",
              "refresh-token-validity": 86400,
              "access-token-validity": 600,
              "secret": "uaa-login-client-password"
            }
          },
          "scim": {
            "users": [
              "director|director-password|bosh.admin",
              "admin|blank-password|bosh.admin,scim.write,clients.write,scim.read,clients.read"
            ]
          }
        },
        "uaadb": {
          "address": "127.0.0.1",
          "db_scheme": "postgresql",
          "port": 5432,
          "databases": [
            {
              "name": "uaa",
              "tag": "uaa"
            }
          ],
          "roles": [
            {
              "name": "postgres",
              "password": "postgres-password",
              "tag": "admin"
            }
          ]
        },
        "vcenter": {
          "address": "192.168.163.131",
          "user": "user",
          "password": "password",
          "datacenters": [
            {
              "name": "vsphere-datacenter",
              "vm_folder": "pivotal_cf_vms_test-installation-guid",
              "template_folder": "pivotal_cf_templates_test-installation-guid",
              "disk_path": "pivotal_cf_disk_test-installation-guid",
              "allow_mixed_datastores": true,
              "datastore_pattern": "^(vsphere\\-datastore)$",
              "persistent_datastore_pattern": "^(vsphere\\-datastore)$",
              "clusters": [
                {
                  "vsphere-cluster": {
                  }
                }
              ]
            }
          ]
        }
      }
    }
  ],
  "cloud_provider": {
    "template": {
      "name": "vsphere_cpi",
      "release": "bosh-vsphere-cpi"
    },
    "mbus": "https://vcap:agent-password@192.168.163.3:6868",
    "properties": {
      "agent": {
        "mbus": "https://vcap:agent-password@0.0.0.0:6868"
      },
      "blobstore": {
        "provider": "local",
        "path": "/var/vcap/micro_bosh/data/cache"
      },
      "ntp": [
        "us.pool.ntp.org"
      ],
      "vcenter": {
        "address": "192.168.163.131",
        "user": "user",
        "password": "password",
        "datacenters": [
          {
            "name": "vsphere-datacenter",
            "vm_folder": "pivotal_cf_vms_test-installation-guid",
            "template_folder": "pivotal_cf_templates_test-installation-guid",
            "disk_path": "pivotal_cf_disk_test-installation-guid",
            "allow_mixed_datastores": true,
            "datastore_pattern": "^(vsphere\\-datastore)$",
            "persistent_datastore_pattern": "^(vsphere\\-datastore)$",
            "clusters": [
              {
                "vsphere-cluster": {
                }
              }
            ]
          }
        ]
      },
      "env": {
      }
    }
  }
}
```

#### HTTP Request

`GET /api/v0/deployed/director/manifest`


##Available Products

An available product is a product that has been uploaded into Ops Manager, or is available for download from Pivotal Network. Available products must be added to the Staged products namespace before configuration changes can be made.

### Uploading a product

```shell
curl "https://example.com/api/v0/available_products" -F 'product[file]=@/path/to/component.zip' -X POST \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{}
```

#### HTTP Request

`POST /api/v0/available_products`

### Checking for product updates

```shell
curl "https://example.com/api/v0/pivotal_network/available_product_updates" -X GET \
  -H "Content-Type: application/json" \
  -d '{"product_name": "pivnet-product-name"}' \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```


```
Example Response
```

```http
HTTP/1.1 200 OK
{
  "versions": [
    "1.1.0",
    "1.0.10"
  ]
}
```

#### HTTP Request

`GET /api/v0/pivotal_network/available_product_updates`

<aside class="warning">
This endpoint will not have any valuable information unless you have configured a Pivotal Network API token.
</aside>

### Fetching EULA content for a given product


```shell
curl "https://example.com/api/v0/pivotal_network/eulas?product_id=example-product-guid" \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "eula": "Legalese..."
}
```

#### HTTP Request

`GET /api/v0/pivotal_network/eulas?product_id=example-product-name`

This retrieves the EULA for the version of the requested product  



### Accepting EULA for a given product

```shell
curl "https://example.com/api/v0/pivotal_network/eulas?product_id=example-product-guid&accept=true" \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK
```
`PUT /api/v0/pivotal_network/eulas?product_id=example-product-name&accept=true`


This accepts the EULA for the version of the requested product, on the 
Pivotal Network

<aside class="notice">
When running UAAC curl commands in a shell or a script, ensure the ampersand is escaped with single quotes.
</aside>


### Downloading a product from Pivotal Network

```shell
curl "https://example.com/api/v0/pivotal_network/download" -X POST \
  -H "Content-Type: application/json" \
  -d '{"product_name": "pivnet-product-name", "version": "1.2.3"}' \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK
{
  "success": true
}
```

#### HTTP Request

`POST /api/v0/pivotal_network/download`

<aside class="warning">
You must have a Pivotal Network API token set for this endpoint to work. You must also have accepted the EULA for the provided version of the product.
</aside>



### Checking for stemcell updates

```shell
curl "https://example.com/api/v0/pivotal_network/stemcell_updates" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```


```
Example Response
```

```http
HTTP/1.1 200 OK
{
  "stemcell_updates": [
    {
      "stemcell_version": "100.10",
      "release_id": 100,
      "products": [
        {
          "product_id": "product1-id"
        },
        {
          "product_id": "product2-id"
        }
      ]
    },
    {
      "stemcell_version": "200.10",
      "release_id": 200,
      "products": [
        {
          "product_id": "product3-id"
        }
      ]
    }
  ]
}
```

#### HTTP Request

`GET /api/v0/pivotal_network/stemcell_updates`

<aside class="warning">
This endpoint will not have any valuable information unless you have configured a Pivotal Network API token.
</aside>




### Listing all available products

```shell
curl "https://example.com/api/v0/available_products" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

[
  {
    "name": "p-bosh",
    "product_version": "1.7.0.0"
  },
  {
    "name": "dummy",
    "product_version": "1.0.0.0"
  }
]

```

#### HTTP Request

`GET /api/v0/available_products`


### Deleting unused products



```shell
curl "https://example.com/api/v0/available_products" -d '' -X DELETE \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN" \
	-H "Content-Type: application/x-www-form-urlencoded"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{}
```

#### HTTP Request

`DELETE /api/v0/available_products`

##Staged Products

Staged Products are products that [have been added](#adding-an-available-product) to the Ops Manager Installation. The Staged namespace represents the desired state of the installation. Changes can be deployed by triggering the [installations controller](#installations).

### Listing all staged products


```shell
curl "https://example.com/api/v0/staged/products" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

[
  {
    "installation_name": "component-type1-installation-name",
    "guid": "component-type1-guid",
    "type": "component-type1"
  },
  {
    "installation_name": "p-bosh-installation-name",
    "guid": "p-bosh-guid",
    "type": "p-bosh"
  }
]
```

#### HTTP Request

`GET /api/v0/staged/products`


### Adding an available product

```shell
curl "https://example.com/api/v0/staged/products" -d 'name=component-type1&product_version=1.0.0.1' -X POST \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN" \
	-H "Content-Type: application/x-www-form-urlencoded"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

```

#### HTTP Request

`POST /api/v0/staged/products`


#### Query Parameters

Parameter  | Description
---------- | -----------
name	| The name of the product as specified in the product template, e.g. 'cf' or 'p-mysql'
product_version	| The version of the product as specified in the product template, e.g. '1.2.0.0'

### Removing products

```shell
curl "https://example.com/api/v0/staged/products/component-type1-guid" -d '' -X DELETE \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN" \
	-H "Content-Type: application/x-www-form-urlencoded"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "component": {
    "guid": "component-type1-guid"
  }
}
```

#### HTTP Request

`DELETE /api/v0/staged/products/:id`



#### Query Parameters

Parameter  | Description
---------- | -----------
id | The guid of the product to be removed from the installation


### Upgrading a product

```shell
curl "https://example.com/api/v0/staged/products/dummy-guid" -d 'to_version=2.0.0.0-alpha' -X PUT \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN" \
	-H "Content-Type: application/x-www-form-urlencoded"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{}
```

#### HTTP Request

`PUT /api/v0/staged/products/:id`

#### Query Parameters

Parameter  | Description
---------- | -----------
id | 	The guid of the product to upgrade
to_version | Version to which the product will be upgraded

### Retrieving a list of jobs

```shell
curl "https://example.com/api/v0/staged/products/product-type1-guid/jobs/" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"

```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "jobs": [
    {
      "name": "job-1-name",
      "guid": "job-1-guid",
    }
  ]
}
```

#### HTTP Request

`GET /api/v0/staged/products/:product_guid/jobs`

This endpoint returns a list of all jobs associated with a product.

### Retrieving resource configuration for a job

```shell
curl "https://example.com/api/v0/staged/products/product-type1-guid/jobs/example-job-guid/resource_config" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"

```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "instances": 1,
  "persistent_disk": {
     "size_mb": "20480"
   }
  "instance_type": {
    "name": "m1.medium"
  }
}
```

#### HTTP Request

`GET /api/v0/staged/products/:product_guid/jobs/:job_id/resource_config`

This endpoint returns the compute and disk configuration for a job.

### Configuring resources for a job

```shell
curl "https://example.com/api/v0/staged/products/product-type1-guid/jobs/example-job-guid/resource_config" -X PUT \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN" \
	--data '{
    "instances": 1,
    "persistent_disk": {
       "size_mb": "20480"
     }
    "instance_type": {
      "name": "m1.medium"
    }
  }'

```

```
Example Response
```

```http
HTTP/1.1 200 OK

{}
```

#### HTTP Request

`PUT /api/v0/staged/products/:product_guid/jobs/:job_id/resource_config`

This endpoint allows setting compute and disk configuration for a job.

###Listing currently assigned networks and azs
```shell
curl "https://example.com/api/v0/staged/products/product-type1-guid/networks_and_azs" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"

```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "networks_and_azs": {
    "singleton_availability_zone": {
      "name": "az-one"
    },
    "other_availability_zones": [
      { "name": "az-two" },
      { "name": "az-three" }
    ],
    "network": {
      "name": "network-one"
    }
  }
}
```

#### HTTP Request

`GET /api/v0/staged/products/:product_guid/networks_and_azs`

This endpoint returns the current network and AZ assignment.

###Listing available networks and azs
###Configuring networks and azs

```shell
curl "https://example.com/api/v0/staged/products/product-type1-guid/networks_and_azs" -X PUT \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN" \
	--data '{
    "networks_and_azs": {
      "singleton_availability_zone": {
        "name": "az-one"
      },
      "other_availability_zones": [
        { "name": "az-two" },
        { "name": "az-three" }
      ],
      "network": {
        "name": "network-one"
      }
    }
  }'

```

```
Example Response
```

```http
HTTP/1.1 200 OK

{}
```

#### HTTP Request

`PUT /api/v0/staged/products/:product_guid/networks_and_azs`

This endpoint allows assigning AZs and networks.

<aside class="warning">
Subnets in the specified network must align with the specified availability zones
</aside>



###Viewing currently selected errands

```shell
curl "https://example.com/api/v0/staged/products/product-type1-guid/errands" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"

```

```
Example Response
```

```http
HTTP/1.1 200 OK
{
	"errands": [
    {
      "name": "errand-1",
      "post_deploy": false
    },
    {
      "name": "errand-2",
      "pre_delete": true
    },
    {
      "name": "shared-errand",
      "post_deploy": false,
      "pre_delete": true
    }
	]
}
```

#### HTTP Request

`GET /api/v0/staged/products/:product_guid/errands`

Errands allowed to run as post_deploy or pre_delete are determined by the product template. 

The presence of the 'post_deploy' or 'pre_delete' key in the response indicates the product author's intent.

The boolean value indicates whether the errand is enabled for that lifecycle event by the operator.

###Configuring errands


```shell
curl "https://example.com/api/v0/staged/products/product-type1-guid/errands" -X PUT \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN" \
	-H "Content-Type: application/json" \
	-d '{
    "errands": [
      {
        "name": "example-errand",
        "post_deploy": true,
        "pre_delete": true
      }
    ]
  }'

```

```
Example Response
```

```http
HTTP/1.1 200 OK
{}
```

#### HTTP Request

`PUT /api/v0/staged/products/:product_guid/errands`

Set enabled or disabled list of errands to run.

###Viewing product properties
```shell
curl "https://example.com/api/v0/staged/products/product-type1-guid/properties" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"

```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "properties": {
    "example_selector": {
      "type": "selector",
      "configurable": true,
      "credential": false,
      "value": "Pizza",
      "options": {
        "Pizza": {
          "pepperoni": {
            "type": "boolean",
            "configurable": true,
            "credential": false,
            "value": false
          },
          "pineapple": {
            "type": "boolean",
            "configurable": true,
            "credential": false,
            "value": false
          },
          "other_toppings": {
            "type": "string",
            "configurable": true,
            "credential": false,
            "value": null
          }
        },
        "Filet Mignon": {
          "rarity_dropdown": {
            "type": "dropdown_select",
            "configurable": true,
            "credential": false,
            "value": "rare"
          },
          "secret_sauce": {
            "type": "secret",
            "configurable": true,
            "credential": true,
            "value": {
              "secret": "***"
            }
          }
        },
        "Beverage": {
          "cola": {
            "type": "string",
            "configurable": true,
            "credential": false,
            "value": null
          }
        }
      }
    },
    "example_collection": {
      "type": "collection",
      "configurable": true,
      "credential": false,
      "value": [
        {
          "guid": {
            "type": "uuid",
            "configurable": false,
            "credential": false,
            "value": "2d6beadc-bf05-41a0-829f-a5d44db14815"
          },
          "album": {
            "type": "string",
            "configurable": false,
            "credential": false,
            "value": "Christmas Carols"
          },
          "artist": {
            "type": "string",
            "configurable": false,
            "credential": false,
            "value": "Ops Manatee"
          },
          "explicit": {
            "type": "boolean",
            "configurable": false,
            "credential": false,
            "value": true
          },
          "secret_meaning": {
            "type": "secret",
            "configurable": true,
            "credential": true,
            "value": {
              "secret": "***"
            }
          }
        }
      ]
    },
    "web_server.static_ips": {
      "type": "ip_ranges",
      "configurable": true,
      "credential": false,
      "value": null
    },
    "web_server.generated_rsa_cert_credentials": {
      "type": "rsa_cert_credentials",
      "configurable": false,
      "credential": true,
      "value": {
        "private_key_pem": "***"
      }
    },
    "web_server.generated_rsa_pkey_credentials": {
      "type": "rsa_pkey_credentials",
      "configurable": false,
      "credential": true,
      "value": {
        "private_key_pem": "***"
      }
    },
    "web_server.generated_salted_credentials": {
      "type": "salted_credentials",
      "configurable": false,
      "credential": true,
      "value": {
        "password": "***",
        "salt": "***"
      }
    },
    "web_server.generated_simple_credentials": {
      "type": "simple_credentials",
      "configurable": false,
      "credential": true,
      "value": {
        "password": "***"
      }
    },
    "web_server.generated_secret": {
      "type": "secret",
      "configurable": false,
      "credential": true,
      "value": {
        "secret": "***"
      }
    },
    "web_server.generated_uuid": {
      "type": "uuid",
      "configurable": false,
      "credential": false,
      "value": null
    },
    "web_server.configured_secret": {
      "type": "secret",
      "configurable": true,
      "credential": true,
      "value": {
        "secret": "***"
      }
    },
    "web_server.configured_simple_credentials": {
      "type": "simple_credentials",
      "configurable": true,
      "credential": true,
      "value": {
        "password": "***"
      }
    },
    "web_server.configured_rsa_cert_credentials": {
      "type": "rsa_cert_credentials",
      "configurable": true,
      "credential": true,
      "value": {
        "private_key_pem": "***"
      }
    },
    "web_server.example_string_with_placeholder": {
      "type": "string",
      "configurable": true,
      "credential": false,
      "value": null
    },
    "web_server.example_string": {
      "type": "string",
      "configurable": true,
      "credential": false,
      "value": "Hello world"
    },
    "web_server.example_migrated_integer": {
      "type": "integer",
      "configurable": true,
      "credential": false,
      "value": 1
    },
    "web_server.example_boolean": {
      "type": "boolean",
      "configurable": true,
      "credential": false,
      "value": true
    },
    "web_server.example_dropdown": {
      "type": "dropdown_select",
      "configurable": true,
      "credential": false,
      "value": "kiwi"
    },
    "web_server.example_domain": {
      "type": "domain",
      "configurable": true,
      "credential": false,
      "value": "www.example.com"
    },
    "web_server.example_wildcard_domain": {
      "type": "wildcard_domain",
      "configurable": true,
      "credential": false,
      "value": "example.com"
    },
    "web_server.example_string_list": {
      "type": "string_list",
      "configurable": true,
      "credential": false,
      "value": "a,list,of,strings"
    },
    "web_server.example_text": {
      "type": "text",
      "configurable": true,
      "credential": false,
      "value": "some_text"
    },
    "web_server.example_ldap_url": {
      "type": "ldap_url",
      "configurable": true,
      "credential": false,
      "value": "ldap://example.com"
    },
    "web_server.example_email": {
      "type": "email",
      "configurable": true,
      "credential": false,
      "value": "foo@example.com"
    },
    "web_server.example_http_url": {
      "type": "http_url",
      "configurable": true,
      "credential": false,
      "value": "http://www.example.com"
    },
    "web_server.example_ip_address": {
      "type": "ip_address",
      "configurable": true,
      "credential": false,
      "value": "192.168.0.1"
    },
    "web_server.example_ip_ranges": {
      "type": "ip_ranges",
      "configurable": true,
      "credential": false,
      "value": "1.1.1.1-1.1.1.4,2.2.2.1-2.2.2.4"
    },
    "web_server.example_multi_select_options": {
      "type": "multi_select_options",
      "configurable": true,
      "credential": false,
      "value": [
        "earth",
        "jupiter"
      ]
    },
    "web_server.example_network_address_list": {
      "type": "network_address_list",
      "configurable": true,
      "credential": false,
      "value": "1.1.1.1,example.com,foo.bar.example.com"
    },
    "web_server.example_network_address": {
      "type": "network_address",
      "configurable": true,
      "credential": false,
      "value": "1.1.1.1"
    },
    "web_server.example_port": {
      "type": "port",
      "configurable": true,
      "credential": false,
      "value": 1111
    },
    "web_server.example_smtp_authentication": {
      "type": "smtp_authentication",
      "configurable": true,
      "credential": false,
      "value": "plain"
    },
    "web_server.client_certificate": {
      "type": "ca_certificate",
      "configurable": true,
      "credential": false,
      "value": null
    }
  }
}


```

#### HTTP Request

`GET /api/v0/staged/products/:product_guid/properties`

This endpoint returns a list of all of the product's properties, along with currently set values.

### Updating a simple property

```shell
# Simple Property
curl "https://example.com/api/v0/staged/products/product-type1-guid/properties" -X PUT \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN" \
	--data '{
    'properties': 
      {'top-level-property': 'valid-data', 'a-job.job-property': 'new-job-data'
  }'

```

```
Example Response
```

```http
HTTP/1.1 200 OK

{}
```

#### HTTP Request

`PUT /api/v0/staged/products/:product_guid/properties`


### Updating a hashed property

```shell
# Hashed Property
curl "https://example.com/api/v0/staged/products/product-type1-guid/properties" -X PUT \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN" \
	--data '{
	'properties': {
        'a-job.job-property': {'identity': 'username', 'password': 'new-password'}
    }
  }'

```

```
Example Response
```

```http
HTTP/1.1 200 OK

{}
```

#### HTTP Request

`PUT /api/v0/staged/products/:product_guid/properties`


### Updating a selector property

```shell
# Selector Property
curl "https://example.com/api/v0/staged/products/product-type1-guid/properties" -X PUT \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN" \
	--data '{
	'properties': {
        'top-level-property': {
           'value': 'pizza_option',
           'options': {
             'pizza_option': {
               'cheese': {
                 'value': true
               },
               'login_info': {
                 'value': {
                   'identity': 'Rey',
                   'password': 'BB-8',
                 }
               }
             }
           }
        }
    }
  }'

```

```
Example Response
```

```http
HTTP/1.1 200 OK

{}
```

#### HTTP Request

`PUT /api/v0/staged/products/:product_guid/properties`


### Updating a collection property

```shell
# Collection Property
curl "https://example.com/api/v0/staged/products/product-type1-guid/properties" -X PUT \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN" \
	--data '{
	'properties': {
        'top-level-property':{
            [
                {
                   'guid' => '66f94d18-e02f-4717-a8ac-121f2cead19c',
                   'name' => 'jesse',
                   'my-secret' => {'secret' => 'bar'},
                }
            ]    
        }
    }
  }'

```

```
Example Response
```

```http
HTTP/1.1 200 OK

{}
```

#### HTTP Request

`PUT /api/v0/staged/products/:product_guid/properties`

##Deployed Products

The Deployed namespace represents the actual state of the installation and various deployment-specific attributes can be retrieved here. 

### Viewing available credentials

```shell
curl "https://example.com/api/v0/deployed/products/product-guid/credentials" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "credentials": [
    ".properties.some-credentials",
    ".my-job.some-credentials"
  ]
}
```

#### HTTP Request

`GET /api/v0/deployed/products/:product_guid/credentials`



This endpoint returns a list of references for credential properties for the given deployed product, except for VM credentials. These references can be used to get the credentials themselves using the credentials endpoint.


#### Query Parameters

Parameter  | Description
---------- | -----------
product_guid | A product guid


### Fetching credentials

```shell
curl "https://example.com/api/v0/deployed/products/product-guid/credentials/.properties.some-credentials" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "credential": {
    "type": "simple_credentials",
    "value": {
      "identity": "carmen-sandiego",
      "password": "hiding-somewhere"
    }
  }
}

```

#### HTTP Request

`GET /api/v0/deployed/products/:product_guid/credentials/:credential_reference`


This endpoint returns the credentials for a specified credential reference as a hash.

#### Query Parameters

Parameter  | Description
---------- | -----------
credential_reference | The credential reference string
product_guid | A product guid


### Listing VM credentials for product jobs

```shell
curl "https://example.com/api/v0/deployed/products/component-type1-guid/vm_credentials" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

[
  {
    "name": "compilation-guid",
    "identity": "vcap1",
    "password": "vm-password1"
  },
  {
    "name": "job-type1-guid",
    "identity": "vcap1",
    "password": "vm-password1"
  },
  {
    "name": "credentials-job-guid",
    "identity": "vcap",
    "password": "vm-password"
  }
]

```

#### HTTP Request

`GET /api/v0/deployed/products/:product_guid/vm_credentials`


#### Query Parameters

Parameter  | Description
---------- | -----------
product_guid |	Product ID


### Retrieving status of product jobs

```shell
curl "https://example.com/api/v0/deployed/products/product-guid/status" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "status": [
    {
      "job-name": "web_server-7f841fc2af9c2b357cc4",
      "index": 0,
      "az_guid": "ee61aa1e420ed3fdf276",
      "az_name": "first-az",
      "ips": [
        "10.85.42.58"
      ],
      "cid": "vm-448ef313-86ee-4049-87cf-764ca2fa97e7",
      "load_avg": [
        "0.00",
        "0.01",
        "0.03"
      ],
      "cpu": {
        "sys": "0.1",
        "user": "0.2",
        "wait": "0.3"
      },
      "memory": {
        "kb": "60632",
        "percent": "6"
      },
      "swap": {
        "kb": "0",
        "percent": "0"
      },
      "system_disk": {
        "inode_percent": "31",
        "percent": "42"
      },
      "ephemeral_disk": {
        "inode_percent": "0",
        "percent": "1"
      },
      "persistent_disk": {
        "inode_percent": "0",
        "percent": "0"
      }
    }
  ]
}
```

#### HTTP Request

`GET /api/v0/deployed/products/:product_guid/status`


The information returned is based on the output of the `bosh vms` command, with some additional data added.

### Listing static IP assignments for product jobs


```shell
curl "https://example.com/api/v0/deployed/products/component-type1-guid/static_ips" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

[
  {
    "name": "job-type1-guid-partition-default-az-guid",
    "ips": [
      "192.168.163.4"
    ]
  },
  {
    "name": "credentials-job-guid-partition-default-az-guid",
    "ips": [
      "192.168.163.7"
    ]
  }
]

```

#### HTTP Request

`GET /api/v0/deployed/products/:product_guid/static_ips`

#### Query Parameters

Parameter  | Description
---------- | -----------
product_guid |	Product ID


### Enqueueing log downloads for a given job

```shell
curl "https://example.com/api/v0/deployed/products/product-type1-guid/jobs/job-example-1-guid/logs" -X POST \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"

```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "id": "3453589567389"
}
```

#### HTTP Request

`POST /api/v0/deployed/products/:product_guid/jobs/:job_guid/logs`


This returns a task identifier for the async operation that performs log downloading from BOSH.

To track log download status, call `GET /api/v0/deployed/products/:product_guid/jobs/:job_guid/logs`



### Listing log download tasks for a given job


```shell
curl "https://example.com/api/v0/deployed/products/component-type1-guid/jobs/job-example-guid/logs" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "tasks": [
    {
      "guid": "3854e98d1378",
      "status": "downloaded",
      "timestamp": "2016-04-21 17:32:10 UTC"
    },
    {
      "guid": "b550456bddbc",
      "status": "downloaded",
      "timestamp": "2016-04-21 17:32:51 UTC"
    },
    {
      "guid": "816ae3784f94",
      "status": "downloaded",
      "timestamp": "2016-04-21 18:08:43 UTC"
    }
  ]
}
```

#### HTTP Request

`GET /api/v0/deployed/products/:product_guid/jobs/:job_guid/logs`


ZIP files for tasks in the 'downloaded' stage are available at `/api/v0/deployed/products/:product_guid/jobs/:job_guid/logs/:task_guid`



### Download ZIP file with logs

```shell
curl -o logs.zip "https://example.com/api/v0/deployed/products/product-type1-guid/jobs/job-example-1-guid/logs/task-guid-example" \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"

```

```
Example Response

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 70035    0 70035    0     0   348k      0 --:--:-- --:--:-- --:--:--  348k
```



#### HTTP Request

`POST /api/v0/deployed/products/:product_guid/jobs/:job_guid/logs/:task_id`



##Stemcells
###Uploading stemcells
```shell
curl "http://example.com/api/v0/stemcells" \
  -H "Authorization: bearer [token]" \
  -F 'stemcell[file]=@/path/to/stemcell/bosh-stemcell-2749-vsphere-esxi-centos-7-go_agent.tgz'
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{}
```

#### HTTP Request

`POST /api/v0/stemcells`

Importing a stemcell

#### Query Parameters

Parameter  | Description
---------- | -----------
stemcell[file] | Stemcell file


##Disk types

### Returning all disk types

```shell
curl "https://example.com/api/v0/disk_types" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK
{
  "disk_types": [
    {
      "name": "1024",
      "builtin": true,
      "size_mb": 1024
    },
    {
      "name": "2048",
      "builtin": true,
      "size_mb": 2048
    },
    {
      "name": "5120",
      "builtin": true,
      "size_mb": 5120
    }
  ]
}
```

#### HTTP Request

`GET /api/v0/disk_types`

When overridden by custom types, this endpoint returns the custom types and the response will
include the dates that the custom disk types were created and modified.


### Deleting all custom disk types

```shell
curl "https://example.com/api/v0/disk_types" -d '' -X DELETE \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK
```

#### HTTP Request

`DELETE /api/v0/disk_types`

Returns available disk types to the default list

### Overriding defaults with custom disk types

```shell
curl "https://example.com/api/v0/disk_types" -d '{"disk_types":[{"size_mb":999},{"size_mb":888},{"size_mb":777}]}' -X PUT \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN" \
	-H "Content-Type: application/json"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "disk_types": [
    {
      "size_mb": 999
    },
    {
      "size_mb": 888
    },
    {
      "size_mb": 777
    }
  ]
}
```

#### HTTP Request

`PUT /api/v0/disk_types`


    
When overridden, the default types will be replaced by operator provided sizes. Operators can repeatedly update the list of available sizes, and any jobs using no-longer-available-sizes will be returned to the default of “automatic”.

##VM types
### Returning all vm types
```shell
curl "https://example.com/api/v0/vm_types" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "vm_types": [
    {
      "name": "nano",
      "ram": 512,
      "cpu": 1,
      "ephemeral_disk": 1024,
      "builtin": true
    },
    {
      "name": "micro",
      "ram": 1024,
      "cpu": 1,
      "ephemeral_disk": 2048,
      "builtin": true
    },
    {
      "name": "small.disk",
      "ram": 2048,
      "cpu": 1,
      "ephemeral_disk": 16384,
      "builtin": true
      }
  ]
}
```

#### HTTP Request

`GET /api/v0/vm_types`

When not overridden by custom types, this endpoint returns all the default vm types for your IaaS
    
### Overriding defaults with custom VM types

```shell
curl "https://example.com/api/v0/vm_types" -d '{"vm_types":[{"name":"mytype","cpu":1,"ram":1024,"ephemeral_disk":1024},{"name":"bigger","cpu":2,"ram":2048,"ephemeral_disk":2048}]}' -X PUT \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN" \
	-H "Content-Type: application/json"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "vm_types": [
    {
      "name": "mytype",
      "cpu": 1,
      "ram": 1024,
      "ephemeral_disk": 1024
    },
    {
      "name": "bigger",
      "cpu": 2,
      "ram": 2048,
      "ephemeral_disk": 2048
    }
  ]
}
```

#### HTTP Request

`PUT /api/v0/vm_types`
    
When overridden, the default types will be replaced by operator provided sizes. 
Operators can repeatedly update the list of available sizes, and any jobs using no-longer-available-sizes will be returned to the default of “automatic”.
    

### Deleting all custom vm types

```shell
curl "https://example.com/api/v0/vm_types" -d '' -X DELETE \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN" \
	-H "Content-Type: application/json"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

```

#### HTTP Request

`DELETE /api/v0/vm_types`


    
Returns available vm types to the default list

##Installation Asset Collection
### Exporting an installation asset collection

```shell
curl "https://example.com/api/v0/installation_asset_collection" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"

```

```
Example Response
```

```http
HTTP/1.1 200 OK

{}
```

#### HTTP Request

`GET /api/v0/installation_asset_collection`

### Importing an installation asset collection

```shell
curl "https://example.com/api/v0/installation_asset_collection" \
 -F 'installation[file]=@/path/to/installation.zip' -F 'passphrase=passphrase' -X POST 

```

```
Example Response
```

```http
HTTP/1.1 200 OK

{}
```

#### HTTP Request

`POST /api/v0/installation_asset_collection`



Ops Manager is now protected by Cloud Foundry UAA for security and multi-user support.

When upgrading from a pre 1.7 version of Ops Manager, a username is automatically created for you, and is set to “admin”. Your password is unchanged. If you are importing a 1.7 or newer installation of Ops Manager, both the username and the password are carried over to the new installation.

In addition to usernames and passwords, Ops Manager will prompt users for a common decryption passphrase upon reboot. The decryption passphrase is currently the same as your password. Change the decryption passphrase before sharing it with other users.

### Resetting an installation

```shell
curl "https://example.com/api/v0/installation_asset_collection" -d '' -X DELETE \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN" \
	-H "Content-Type: application/x-www-form-urlencoded"

```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "install": {
    "id": 12
  }
}
```

#### HTTP Request

`DELETE /api/v0/installation_asset_collection`

This endpoint allows you to return your Ops Manager to its initial state. All products and BOSH configuration settings will be lost. Files uploaded or downloaded to the "Available Products" namespace will continue to be available. Hitting this endpoint does not reset your UAA login server and only affects Ops Manager, BOSH, and products installed on them.

#ADVANCED
##UAA Settings
###Viewing token expiration times

```shell
curl "https://example.com/api/v0/uaa/tokens_expiration" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "tokens_expiration": {
    "access_token_expiration": 100,
    "refresh_token_expiration": 1200
  }
}
```

#### HTTP Request

`GET /api/v0/uaa/tokens_expiration`

This endpoint allows you to view the currently set expiration times for UAA access and refresh tokens.

###Changing token expiration times

```shell
curl "https://example.com/api/v0/uaa/tokens_expiration" -d 'tokens_expiration[access_token_expiration]=200&tokens_expiration[refresh_token_expiration]=1400' -X PUT \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN" \
	-H "Content-Type: application/x-www-form-urlencoded"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{}
```

#### HTTP Request

`PUT /api/v0/uaa/tokens_expiration`

Changes the current access & refresh token expirations for Ops Manager UAA and restarts the UAA

#### Query Parameters

Parameter  | Description
---------- | -----------
tokens_expiration	| FOO



##Metadata
###Migrating metadata

```shell
curl "https://example.com/api/v0/metadata/migrate" -F 'metadata[file]=@/path/to/component-type1.yml' -X POST \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "metadata": "---\nname: component-type1\nproduct_version: 1.0.0.0\nmetadata_version: '1.7'\nreleases:\n- name: component-type1-release-name\n  file: component-type1-release-file\n  version: component-type1-release-version\n  md5: component-type1-release-md5\nlabel: component-type1-label\ndescription: component-type1-description\nrank: 1\nprovides_product_versions:\n- name: component-type1\n  version: 1.0.0.0\nrequires_product_versions:\n- name: component-type2\n  version: \"~> 1.0.0\"\nserial: true\nform_types:\n- name: job-type1\n  label: job-type1-label\n  description: job-type1-description\n  property_inputs:\n  - reference: \".job-type1.property-definition1\"\n    label: property-definition1-label\n- name: job-type3\n  label: job-type3-label\n  description: job-type3-description\n  property_inputs:\n  - reference: \".job-type3.http-url\"\n    label: HTTP URL\n  - reference: \".job-type3.domain\"\n    label: Domain\n  - reference: \".job-type3.ip-ranges\"\n    label: IP Ranges\n  - reference: \".job-type3.ip-address\"\n    label: IP Address\n  - reference: \".job-type3.email\"\n    label: E-mail\n  - reference: \".job-type3.port\"\n    label: Port\n  - reference: \".job-type3.integer\"\n    label: Integer\n  - reference: \".job-type3.boolean\"\n    label: Boolean\n  - reference: \".job-type3.string\"\n    label: String\n  - reference: \".job-type3.smtp-authentication\"\n    label: SMTP Authentication\n  - reference: \".job-type3.network-address\"\n    label: Network Address\n  - reference: \".job-type3.simple-credentials\"\n    label: Simple credentials\n  - reference: \".job-type3.rsa-cert-credentials\"\n    label: RSA PEM and Certificate\n  - reference: \".job-type3.ca-certificate\"\n    label: CA Certificate PEM\n  - reference: \".job-type3.checkboxes\"\n    label: Checkboxes\n  - reference: \".job-type3.erlang-config\"\n    label: Erlang Configuration\njob_types:\n- name: job-type1\n  resource_label: job-type1-resource-label\n  resource_definitions:\n  - name: ram\n    type: integer\n    label: RAM\n    configurable: true\n    default: 1\n  - name: ephemeral_disk\n    type: integer\n    label: Ephemeral Disk\n    configurable: true\n    default: 2\n  - name: persistent_disk\n    type: integer\n    label: Persistent Disk\n    configurable: true\n    default: 3\n    constraints:\n      min: 1\n  - name: cpu\n    type: integer\n    label: CPU\n    configurable: true\n    default: 4\n  static_ip: 1\n  dynamic_ip: 0\n  max_in_flight: 1\n  property_blueprints:\n  - name: property-definition1\n    type: domain\n    configurable: true\n  - name: property-definition2\n    type: string\n    configurable: false\n  - name: property-definition3\n    type: secret\n    configurable: false\n  manifest: |\n    job_name: job-type1\n    properties:\n      property1: (( property-definition1.value ))\n      property2: (( .job-type1.property-definition2.typed_value.value ))\n      property3: (( .job-type1.property-definition3.typed_value.value ))\n  templates:\n  - name: job-type1-template\n    release: component-type1-release-name\n  instance_definition:\n    configurable: true\n    default: 1\n  single_az_only: false\n- name: job-type2\n  resource_label: job-type2-resource-label\n  resource_definitions:\n  - name: ram\n    type: integer\n    label: RAM\n    configurable: true\n    default: 1024\n  - name: ephemeral_disk\n    type: integer\n    label: Ephemeral Disk\n    configurable: true\n    default: 2048\n  - name: persistent_disk\n    type: integer\n    label: Persistent Disk\n    configurable: true\n    default: 8192\n    constraints:\n      min: 1\n  - name: cpu\n    type: integer\n    label: CPU\n    configurable: true\n    default: 1\n  static_ip: 1\n  dynamic_ip: 0\n  max_in_flight: 1\n  property_blueprints: []\n  manifest: |\n    job_name: job-type2\n  templates:\n  - name: job-type2-template\n    release: component-type1-release-name\n  instance_definition:\n    default: 1\n  single_az_only: false\n- name: job-type3\n  resource_label: job-type3-resource-label\n  resource_definitions:\n  - name: ram\n    type: integer\n    label: RAM\n    configurable: true\n    default: 1024\n  - name: ephemeral_disk\n    type: integer\n    label: Ephemeral Disk\n    configurable: true\n    default: 2048\n  - name: persistent_disk\n    type: integer\n    label: Persistent Disk\n    configurable: true\n    default: 8192\n    constraints:\n      min: 1\n  - name: cpu\n    type: integer\n    label: CPU\n    configurable: true\n    default: 1\n  static_ip: 1\n  dynamic_ip: 0\n  max_in_flight: 1\n  property_blueprints:\n  - name: http-url\n    type: http_url\n    configurable: true\n    default: http://default.example.com\n  - name: domain\n    type: domain\n    configurable: true\n    default: default.domain.com\n  - name: ip-ranges\n    type: ip_ranges\n    configurable: true\n    default: 1.2.3.4-1.2.3.10,2.3.4.5-2.3.4.9\n  - name: ip-address\n    type: ip_address\n    configurable: true\n    default: 1.2.3.4\n  - name: email\n    type: email\n    configurable: true\n    default: email@example.com\n  - name: port\n    type: port\n    configurable: true\n    default: 80\n  - name: integer\n    type: integer\n    configurable: true\n    default: 32\n    constraints:\n      min: 1\n      max: 32\n  - name: boolean\n    type: boolean\n    configurable: true\n    default: false\n  - name: string\n    type: string\n    configurable: true\n    default: Some Text\n  - name: smtp-authentication\n    type: smtp_authentication\n    configurable: true\n    default: cram_md5\n  - name: network-address\n    type: network_address\n    configurable: true\n    default: 1.2.3.4\n  - name: simple-credentials\n    type: simple_credentials\n    configurable: true\n  - name: rsa-cert-credentials\n    type: rsa_cert_credentials\n    optional: true\n    configurable: true\n  - name: ca-certificate\n    type: ca_certificate\n    optional: true\n    configurable: true\n  - name: checkboxes\n    type: multi_select_options\n    configurable: true\n    optional: true\n    options:\n    - name: checkbox1\n      label: Checkbox 1\n    - name: checkbox2\n      label: Checkbox 2\n    - name: checkbox3\n      label: Checkbox 3\n  - name: erlang-config\n    type: text\n    configurable: true\n    optional: true\n  manifest: |\n    job_name: job-type3\n  templates:\n  - name: job-type3-template\n    release: component-type1-release-name\n  instance_definition:\n    default: 1\n  single_az_only: false\n- name: compilation\n  resource_label: compilation-resource-label\n  resource_definitions:\n  - name: ram\n    type: integer\n    label: RAM\n    configurable: true\n    default: 1024\n  - name: ephemeral_disk\n    type: integer\n    label: Ephemeral Disk\n    configurable: true\n    default: 2048\n  - name: persistent_disk\n    type: integer\n    label: Persistent Disk\n    configurable: true\n    default: 8192\n    constraints:\n      min: 1\n  - name: cpu\n    type: integer\n    label: CPU\n    configurable: true\n    default: 1\n  static_ip: 1\n  dynamic_ip: 0\n  max_in_flight: 1\n  instance_definition:\n    default: 1\n  single_az_only: false\noriginal_metadata_version: '1.6'\ndeprecated_tile_image: component-type1-image\nicon_image: \nstemcell_criteria:\n  os: ubuntu-trusty\n  version: '9000'\nminimum_version_for_upgrade: 0.0.0.0\n"
}
```

#### HTTP Request

`POST /api/v0/metadata/migrate`

#### Query Parameters

Parameter  | Description
---------- | -----------
metadata[file] |	Metadata file


##Manifests
###Generating a manifest for a staged product
```shell
curl "https://example.com/api/v0/staged/products/component-type1-guid/manifest" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "manifest": {
    "name": "component-type1-installation-name",
    "director_uuid": "ignore",
    "releases": [
      {
        "name": "release-17",
        "version": "2"
      }
    ],
    "networks": [
      {
        "name": "net-subnet-guid",
        "subnets": [
          {
            "range": "192.168.163.0/24",
            "gateway": "192.168.163.2",
            "dns": [
              "192.168.163.1"
            ],
            "static": [

            ],
            "reserved": [
              "192.168.163.1",
              "192.168.163.3-192.168.163.7",
              "192.168.163.9-192.168.163.254"
            ],
            "cloud_properties": {
              "name": "vsphere-network"
            }
          }
        ]
      }
    ],
    "resource_pools": [

    ],
    "compilation": {
      "reuse_compilation_vms": true,
      "workers": 1,
      "network": "net-subnet-guid",
      "cloud_properties": {
        "ram": 1024,
        "disk": 2048,
        "cpu": 1,
        "datacenters": [
          {
            "clusters": [
              {
                "vsphere-cluster": {
                }
              }
            ]
          }
        ]
      }
    },
    "update": {
      "canaries": 1,
      "canary_watch_time": "30000-300000",
      "update_watch_time": "30000-300000",
      "max_in_flight": 1,
      "max_errors": 2,
      "serial": false
    },
    "jobs": [

    ],
    "disk_pools": [

    ]
  }
}
```

#### HTTP Request

`GET /api/v0/staged/products/:product_guid/manifest`

To view the manifest for a product, replace :product_guid with the appropriate guid. 
<aside class="warning">
Using this manifest outside OpsManager will prevent you from using OpsManager in the future.
</aside>

###Retrieving manifest for a deployed product

```shell
curl "https://example.com/api/v0/deployed/products/component-type1-guid/manifest" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "name": "component-type1-installation-name",
  "director_uuid": "ignore",
  "releases": [
    {
      "name": "component-type1-release-name",
      "version": "component-type1-release-version"
    }
  ],
  "networks": [
    {
      "name": "default",
      "subnets": [
        {
          "range": "192.168.163.0/24",
          "gateway": "192.168.163.2",
          "dns": [
            "192.168.163.3",
            "192.168.163.1"
          ],
          "static": [
            "192.168.163.4",
            "192.168.163.5",
            "192.168.163.6",
            "192.168.163.7"
          ],
          "reserved": [
            "192.168.163.1",
            "192.168.163.3",
            "192.168.163.10-192.168.163.100",
            "192.168.163.103-192.168.163.254"
          ],
          "cloud_properties": {
            "name": "vsphere-network"
          }
        }
      ]
    }
  ],
  "resource_pools": [
    {
      "name": "job-type1-installation-name-partition-default-az-guid",
      "stemcell": {
        "name": "component-type1-stemcell-name",
        "version": "component-type1-stemcell-version"
      },
      "network": "default",
      "size": 1,
      "cloud_properties": {
        "ram": 1,
        "disk": 2,
        "cpu": 4,
        "datacenters": [
          {
            "clusters": [
              {
                "vsphere-cluster": {
                  "resource_pool": null
                }
              }
            ]
          }
        ]
      },
      "env": {
        "bosh": {
          "password": "vm-password-hashed"
        }
      }
    },
    {
      "name": "job-type2-installation-name-partition-default-az-guid",
      "stemcell": {
        "name": "component-type1-stemcell-name",
        "version": "component-type1-stemcell-version"
      },
      "network": "default",
      "size": 1,
      "cloud_properties": {
        "ram": 1024,
        "disk": 2048,
        "cpu": 1,
        "datacenters": [
          {
            "clusters": [
              {
                "vsphere-cluster": {
                  "resource_pool": null
                }
              }
            ]
          }
        ]
      },
      "env": {
        "bosh": {
          "password": "vm-password-hashed"
        }
      }
    },
    {
      "name": "job-type3-installation-name-partition-default-az-guid",
      "stemcell": {
        "name": "component-type1-stemcell-name",
        "version": "component-type1-stemcell-version"
      },
      "network": "default",
      "size": 1,
      "cloud_properties": {
        "ram": 1024,
        "disk": 2048,
        "cpu": 1,
        "datacenters": [
          {
            "clusters": [
              {
                "vsphere-cluster": {
                  "resource_pool": null
                }
              }
            ]
          }
        ]
      },
      "env": {
        "bosh": {
          "password": "vm-password-hashed"
        }
      }
    }
  ],
  "compilation": {
    "workers": 1,
    "network": "default",
    "cloud_properties": {
      "ram": 1024,
      "disk": 2048,
      "cpu": 1
    }
  },
  "update": {
    "canaries": 1,
    "canary_watch_time": "30000-300000",
    "update_watch_time": "30000-300000",
    "max_in_flight": 1,
    "max_errors": 2,
    "serial": false
  },
  "jobs": [
    {
      "name": "job-type1-installation-name-partition-default-az-guid",
      "template": "job-type1-template",
      "release": "component-type1-release-name",
      "lifecycle": "service",
      "resource_pool": "job-type1-installation-name-partition-default-az-guid",
      "instances": 1,
      "persistent_disk": 3,
      "networks": [
        {
          "name": "default",
          "static_ips": [
            "192.168.163.4"
          ],
          "default": [
            "dns",
            "gateway"
          ]
        }
      ],
      "update": {
        "max_in_flight": 5,
        "canaries": 2,
        "serial": false
      },
      "properties": {
        "job_name": "job-type1"
      }
    },
    {
      "name": "job-type2-installation-name-partition-default-az-guid",
      "template": "job-type2-template",
      "release": "component-type1-release-name",
      "lifecycle": "service",
      "resource_pool": "job-type2-installation-name-partition-default-az-guid",
      "instances": 1,
      "persistent_disk": 8192,
      "networks": [
        {
          "name": "default",
          "static_ips": [
            "192.168.163.5"
          ],
          "default": [
            "dns",
            "gateway"
          ]
        }
      ],
      "update": {
        "max_in_flight": 1
      },
      "properties": {
        "job_name": "job-type2"
      }
    },
    {
      "name": "job-type3-installation-name-partition-default-az-guid",
      "template": "job-type3-template",
      "release": "component-type1-release-name",
      "lifecycle": "service",
      "resource_pool": "job-type3-installation-name-partition-default-az-guid",
      "instances": 1,
      "persistent_disk": 8192,
      "networks": [
        {
          "name": "default",
          "static_ips": [
            "192.168.163.6"
          ],
          "default": [
            "dns",
            "gateway"
          ]
        }
      ],
      "update": {
        "max_in_flight": 1
      },
      "properties": {
        "job_name": "job-type3"
      }
    }
  ]
}
```

#### HTTP Request

`GET /api/v0/deployed/products/:product_guid/manifest`
   
To view the manifest for a product, replace :product_guid with the appropriate guid.


##Base Release URL
### Get active base releases url
```shell
curl "https://example.com/api/v0/staged/products/product-type1-guid/base_releases_url" \
    -H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "base_releases_url": "https://example.com/releases"
}
```

#### HTTP Request

`GET /api/v0/staged/products/:id/base_releases_url`

Light tiles contain pointers to installation binaries and these pointers can be changed in circumstances where the default location is inaccessible (e.g. in an airgapped or firewalled network). 

### Update active base releases url

```shell
curl "https://example.com/api/v0/staged/products/product-type1-guid/base_releases_url" \
    -X PUT \
    -H "Authorization: Bearer UAA_ACCESS_TOKEN" \
    -H 'Content-type: application/json'
    -d '{
        "base_releases_url": "https://mirror.example.com/releases"
    }'
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "base_releases_url": "https://mirror.example.com/releases"
}
```

#### HTTP Request

`PUT /api/v0/staged/products/:id/base_releases_url`

When base_releases_url is set, the default pointers are ignored and BOSH attempts to download releases from the location specified.

#### Query Parameters

Parameter          | Description
------------------ | -----------
base_releases_url  | New base releases url


### Reset active base releases url

Resets to the default specified in the product template.

```shell
curl "https://example.com/api/v0/staged/products/product-type1-guid/base_releases_url" \
    -X DELETE \
    -H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "base_releases_url": "https://example.com/releases"
}
```

####HTTP Request

`DELETE /api/v0/staged/products/:id/base_releases_url`

##Sessions
###Logging out all active users

Only one user can be active in PCF Ops Manager at a time. 
For API users, we consider a user to be active during the period between 
their last request and when their token expires. 
This endpoint will make inactive all API users and log out all UI users, including yourself,
allowing a new user to log in or make API requests.

```shell
curl "https://example.com/api/v0/sessions" -d '' -X DELETE \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN" \
	-H "Content-Type: application/x-www-form-urlencoded"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{}
```

#### HTTP Request

`DELETE /api/v0/sessions`

##Unlock
###Unlocking with the encryption passphrase
When the application reboots after initial setup, it requires an operator to enter the decryption
passphrase once to unlock its internal datastore.

```shell
curl "https://example.com/api/v0/unlock" -d 'passphrase=passphrase' -X PUT \
	-H "Content-Type: application/x-www-form-urlencoded"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

passphrase=passphrase
```

#### HTTP Request

`PUT /api/v0/unlock`


#### Query Parameters

Parameter  | Description
---------- | -----------
passphrase	| Decryption passphrase


##Security
### Returning the Root CA Certificate 

This returns the public key of the Root CA Certificate

```shell
curl "https://example.com/api/v0/security/root_ca_certificate" -X GET 
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "root_ca_certificate_pem": "-----BEGIN CERTIFICATE-----\nMIIC+zCCAeOgAwIBAgIBADANBgkqhkiG9w0BAQUFADAfMQswCQYDVQQGEwJVUzEQ\nMA4GA1UECgwHUGl2b3RhbDAeFw0xNjA0MTExNTE0NTFaFw0yMDA0MTIxNTE0NTFa\nMB8xCzAJBgNVBAYTAlVTMRAwDgYDVQQKDAdQaXZvdGFsMIIBIjANBgkqhkiG9w0B\nAQEFAAOCAQ8AMIIBCgKCAQEAru6dVTEFWsA0SNg2peiQVOcDu/xM9RtKc8YOqio6\nTsouA5pMHbGtvHOYVhuYZZPsN3X5mTdPOb27y3mgyw/eRrN6ycTMmYG9MLZUBNu7\nAUe+JKjupS5h73Txo62nkRUeDpf+4w+ZrMDwQqjeWZ6+FusVyyo+DrP88jRiymxy\nl/XBqBrfs40Sq8plwP42hZI6fGSdtAGbWIGmha3vwvrlaWpkyfBUOdvf2aLVlu8u\nTpzyTQ6fOnjTNP3KolKPUzvOhmRDBEC02jGy7oNvJR67bd0ZbPJzqepHFgrFmB/Z\n5zAyL08EoGD2eb3J3KRqMrSGC75CO/n490iT32kQ92EMxwIDAQABo0IwQDAdBgNV\nHQ4EFgQU23Zk5rl6JqAVIyyn7c5kHpqU2vQwDwYDVR0TAQH/BAUwAwEB/zAOBgNV\nHQ8BAf8EBAMCAQYwDQYJKoZIhvcNAQEFBQADggEBAEFSudPNo5j86kpN/qXDyNpS\ndW+ERkBi+5HY56LG68V2Xp4B/L/rLCqMeS8kSWcTp+lA5mgciwgZbBlqHF+/Rvet\nuoLNz7L/HC1zadhjmj9bWnkoiXdrQFlTXasW7nmB81gZr2VDhRchsstGiVSTST2v\n7YjHC34GGHC6wqXXhtb85kGQQmwwh1K3snzreHrlf7O/mKVkTKcMBRHOWTuFUCOM\nPPx/ZdKGHd/6lBUaKJOJxr+5S8+DW6NORduxZn+N9QiK8fvGZIFzU8Xd6cr2iWSz\nVElVm2rLaHK1Z/WYqUEsLwJGDbaS7+g8D8InZteKh4DNIQIK+e1rt5rDMl8sbsI=\n-----END CERTIFICATE-----\n"
}
```

#### HTTP Request

`GET /api/v0/security/root_ca_certificate`


## Diagnostic Report
###Viewing the diagnostic report
Retrieve a diagnostic report with general information about the state of your Ops Manager.

```shell
curl "https://example.com/api/v0/diagnostic_report" -X GET \
	-H "Authorization: Bearer UAA_ACCESS_TOKEN"
```

```
Example Response
```

```http
HTTP/1.1 200 OK

{
  "versions": {
    "installation_schema_version": "1.8",
    "metadata_version": "1.8",
    "release_version": "1.8.0.0",
    "javascript_migrations_version": "v1"
  },
  "generation_time": "2016-04-22T18:06:46Z",
  "infrastructure_type": "vsphere",
  "director_configuration": {
    "bosh_recreate_on_next_deploy": false,
    "resurrector_enabled": false,
    "blobstore_type": "local",
    "max_threads": null,
    "database_type": "internal",
    "ntp_servers": [],
    "hm_pager_duty_enabled": false,
    "hm_emailer_enabled": false,
    "vm_password_type": "generate"
  },
  "releases": [
    "example-release-14.tgz",
  ],
  "stemcells": [
    "bosh-stemcell-3215-vsphere-esxi-ubuntu-trusty-go_agent.tgz",
  ],
  "product_templates": [
    "e08002f028a5.yml"
  ],
  "added_products": {
    "deployed": [],
    "staged": [
      {
        "name": "p-bosh",
        "version": "1.8.0.0",
        "stemcell": "bosh-stemcell-3215-vsphere-esxi-ubuntu-trusty-go_agent.tgz"
      },
      {
        "name": "example-product",
        "version": "1.8.0.0-alpha",
        "stemcell": "bosh-stemcell-3215-vsphere-esxi-ubuntu-trusty-go_agent.tgz"
      }
    ]
  }
}
```

#### HTTP Request

`GET /api/v0/diagnostic_report`



