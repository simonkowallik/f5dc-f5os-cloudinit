# F5OS API Interactions

Use these blocks with httpYac in order: create the cloud-init object, deploy the tenant, then verify it. Requires F5OS 2.0+, a BIG-IP 21.1+ tenant image, and a reachable DO RPM.

## HTTP client setup

Replace all lab addresses and credentials before running. The TLS override and plaintext passwords are for this lab only; keep real secrets out of version control. httpYac expands the `Basic {{user}} {{password}}` notation into an HTTP Basic auth header.

```http
// The F5OS management IP address, using port 443 with /api/.
@host = https://192.168.245.11/api/
// Basic auth is great for demos; in practice, use tokens and keep your secrets secret!
@user = admin
@password = Lab_F5OS_Admin
```

```http
// httpYac for VS Code - REST API client settings: disable TLS certificate verification with the metadata value below
# @no-reject-unauthorized

{{+
  // default HTTP headers for the F5OS REST API
  exports.F5OS_HTTP_headers = {
    'Content-Type': 'application/yang-data+json',
    'Accept': 'application/yang-data+json',
    'Authorization': 'Basic {{user}} {{password}}',
    'Connection': 'close'
  };
  // Function to generate a cloud-init artifact name, e.g. based on the tenant name
  // must consist only of lowercase letters (a–z) and digits (0–9) and be 1–32 characters long
  exports.GenCloudInitName = async function(val) {
    const input = String(val ?? '');
    const name = input.toLowerCase().replace(/[^a-z0-9]/g, '').slice(0, 26);
    const suffix = require('crypto').createHash('sha256').update(input).digest('hex').slice(0, 6);
    return name + suffix;
  };
  // String (e.g. YAML cloud-init artifact) to JSON string
  exports.toJsonString  = (str) => JSON.stringify(str ?? '');
}}
```


```http
@tenant_name = bigip-simon-devices-lab-local-102
```


## Cloud-init

### Cloud-init YAML

The YAML below is repeated verbatim in `cloudInitCfg` for the F5OS RESTCONF API request. `bootcmd` waits for MCPD, then adds a host entry for the lab artifact server; it does not configure DNS servers, but you could add that functionality if required. Change the artifact URL and host entry together for your environment.

The root password is `Initial!Root!123`.

```yaml
#cloud-config
chpasswd:
  # Set initial credentials
  # The plaintext admin credentials are NOT recommended but included for demonstrative purposes
  list: |
    admin:Admin_StartPW_356
    root:$6$ozX2YB1ynyheTwQG$WqWrhqniKK6EAhJfapeUt/onM.UUT6cBdvKGr/tg2LrJMN/SM8jFljYiJdfPDRKq26cU9JERNaDU/ftXyq2GF.
  # expire: true would force an immediate password change after logon, not great for automation
  expire: false

bootcmd:
  # https://clouddocs.f5.com/cloud/public/v1/shared/cloudinit.html#cloud-config-using-write-files-and-runcmd-modules-example
  # Run only once: the tmsh add commands are unnecessary to repeat on each boot.
  # Wait for MCPD before setting the host entry needed to fetch the DO RPM.
  - cloud-init-per once bootstrap_mcpd_wait bash -ec 'source /usr/lib/bigstart/bigip-ready-functions; wait_bigip_ready_config; tmsh modify sys global-settings remote-host add { internal-artifact-server.s3.example.com { hostname internal-artifact-server.s3.example.com addr 192.168.245.9 } }'

tmos_declared:
  enabled: true
  icontrollx_trusted_sources: false
  icontrollx_package_urls:
    # retrieve from https://my.f5.com/manage/s/downloads?productFamily=DO&productLine=Declarative+Onboarding
    # then store on an internal artifact server (e.g. S3 buckets or jump hosts)
    - http://internal-artifact-server.s3.example.com/F5artifacts/f5-declarative-onboarding-1.49.0-14.noarch.rpm
  do_declaration:
    schemaVersion: 1.49.0
    class: Device
    async: true
    label: "Simon's DO declaration to demo BIG-IP TMOS 21.1+ CloudInit on F5OS 2.0+"
    Common:
      class: Tenant
      myFavoriteSystemSettings:
        class: System
        hostname: simons-bigip.example.com
        guiSecurityBanner: true
        guiSecurityBannerText: |
            ****************************  W A R N I N G  *****************************
            *     Access to this system is restricted to authorized users only.      *
            **************************************************************************
      myFavoriteSystemDbKeys:
        class: DbVariables
        ui.advisory.enabled: true
        ui.advisory.color: orange
        ui.advisory.text: This is an advisory text with an orange background.
        # Do not run the setup wizard
        setup.run: false

```

### Create cloud-init for tenant

F5OS expects YAML user-data as a JSON string in `encrypted-data`; `toJsonString` handles the quoting. A deployed tenant prevents edits or deletion of its referenced object.

```http
{{
exports.cloudInitCfg = `#cloud-config
chpasswd:
  # Set initial credentials
  # The plaintext admin credentials are NOT recommended but included for demonstrative purposes
  list: |
    admin:Admin_StartPW_356
    root:$6$ozX2YB1ynyheTwQG$WqWrhqniKK6EAhJfapeUt/onM.UUT6cBdvKGr/tg2LrJMN/SM8jFljYiJdfPDRKq26cU9JERNaDU/ftXyq2GF.
  # expire: true would force an immediate password change after logon, not great for automation
  expire: false

bootcmd:
  # https://clouddocs.f5.com/cloud/public/v1/shared/cloudinit.html#cloud-config-using-write-files-and-runcmd-modules-example
  # Run only once: the tmsh add commands are unnecessary to repeat on each boot.
  # Wait for MCPD before setting the host entry needed to fetch the DO RPM.
  - cloud-init-per once bootstrap_mcpd_wait bash -ec 'source /usr/lib/bigstart/bigip-ready-functions; wait_bigip_ready_config; tmsh modify sys global-settings remote-host add { internal-artifact-server.s3.example.com { hostname internal-artifact-server.s3.example.com addr 192.168.245.9 } }'

tmos_declared:
  enabled: true
  icontrollx_trusted_sources: false
  icontrollx_package_urls:
    # retrieve from https://my.f5.com/manage/s/downloads?productFamily=DO&productLine=Declarative+Onboarding
    # then store on an internal artifact server (e.g. S3 buckets or jump hosts)
    - http://internal-artifact-server.s3.example.com/F5artifacts/f5-declarative-onboarding-1.49.0-14.noarch.rpm
  do_declaration:
    schemaVersion: 1.49.0
    class: Device
    async: true
    label: "Simon's DO declaration to demo BIG-IP TMOS 21.1+ CloudInit on F5OS 2.0+"
    Common:
      class: Tenant
      myFavoriteSystemSettings:
        class: System
        hostname: simons-bigip.example.com
        guiSecurityBanner: true
        guiSecurityBannerText: |
            ****************************  W A R N I N G  *****************************
            *     Access to this system is restricted to authorized users only.      *
            **************************************************************************
      myFavoriteSystemDbKeys:
        class: DbVariables
        ui.advisory.enabled: true
        ui.advisory.color: orange
        ui.advisory.text: This is an advisory text with an orange background.
        # Do not run the setup wizard
        setup.run: false
`;
}}

POST /data/f5-cloud-init:cloud-inits
...F5OS_HTTP_headers

{
    "f5-cloud-init:cloud-init": [
        {
            "name": "{{GenCloudInitName(tenant_name)}}",
            "config": {
                "user-data": {
                    "encrypted-data": {{toJsonString(cloudInitCfg)}}
                }
            }
        }
    ]
}

```

Run a GET to see how the cloud-init object name is generated (we have only seen the variable so far).

```http
GET /data/f5-cloud-init:cloud-inits
?fields=cloud-init/name
...F5OS_HTTP_headers

{
  "f5-cloud-init:cloud-inits": {
    "cloud-init": [
      {
        "name": "bigipsimondeviceslablocal1a0b15f"
      }
    ]
  }
}
```

## BIG-IP tenants - create with cloud-init

Adjust the image, management network, and resource sizes. `nodes: [1]` is for rSeries; check your platform's requirements before deploying. The POST starts the tenant but does not confirm that DO finished successfully!

```http
POST /data/f5-tenants:tenants
...F5OS_HTTP_headers

{
    "f5-tenants:tenant": [
        {
            "name": "{{tenant_name}}",
            "config": {
                "name": "{{tenant_name}}",
                "type": "BIG-IP",
                "nodes": [1],
                "image": "BIGIP-21.1.0.2-0.0.22.ALL-F5OS.tar.bundle",
                "cloud-init": "{{GenCloudInitName(tenant_name)}}",
                "mgmt-ip": "192.168.245.102",
                "prefix-length": "24",
                "gateway": "192.168.245.254",
                "vcpu-cores-per-node": "8",
                "memory": 29184,
                "storage": { "size": 90 },
                "running-state": "deployed"
            }
        }
    ]
}
```

Example GET on the tenant to display the configuration on the F5OS device.

```http
GET /data/f5-tenants:tenants/tenant={{tenant_name}}
?content=config&with-defaults=explicit
...F5OS_HTTP_headers
```

```json
{
  "f5-tenants:tenant": [
    {
      "name": "bigip-simon-devices-lab-local-102",
      "config": {
        "name": "bigip-simon-devices-lab-local-102",
        "type": "BIG-IP",
        "image": "BIGIP-21.1.0.2-0.0.22.ALL-F5OS.tar.bundle",
        "cloud-init": "bigipsimondeviceslablocal1a0b15f",
        "nodes": [1],
        "mgmt-ip": "192.168.245.102",
        "prefix-length": 24,
        "gateway": "192.168.245.254",
        "vcpu-cores-per-node": 8,
        "memory": "29184",
        "storage": { "size": 90 },
        "running-state": "deployed"
      }
    }
  ]
}
```

If you want to delete the tenant again, which is not uncommon when testing initial provisioning tasks:

```http
DELETE /data/f5-tenants:tenants/tenant={{tenant_name}}
...F5OS_HTTP_headers
```

Only after removing the tenant can the cloud-init object be deleted as well.

```http
DELETE /data/f5-cloud-init:cloud-inits/cloud-init={{GenCloudInitName(tenant_name)}}
...F5OS_HTTP_headers
```



## Check the tenant

It will take a while, probably 5-7 minutes before you see a successful response.

```http
GET https://192.168.245.102/mgmt/shared/declarative-onboarding
Authorization: Basic admin:Admin_StartPW_356
```
