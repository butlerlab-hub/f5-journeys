# JOURNEYS - BIG-IP upgrade and migration utility

----
## Contents:
- [Description](#description)
- [Journey: Full config migration](#journey-full-config-migration)
- [Journey: Application Service migration](#journey-application-service-migration)
- [Configuration Migration Considerations](#configuration-migration-considerations)
- [JOURNEYS Setup Requirements](#journeys-setup-requirements)
- [Usage](#usage)
- [Feature Details](#feature-details)
- [Contributing](#contributing)

----
## Description
JOURNEYS is an application designed to assist F5 Customers with migrating a BIG-IP configuration to a new F5 device and enable new ways of migrating.

Supported journeys:
+ Full Config migration - migrating a BIG-IP configuration from any version starting at 11.5.0 to a higher one, including VELOS systems.
+ Application Service migration - migrating mission critical Applications and their dependencies to a new AS3 configuration and deploying it to a BIG-IP instance of choice.

## Journey: Full Config migration
Supported features:
+ Loading UCS or UCS+AS3 source configurations
+ Flagging source configuration feature parity gaps and fixing them with provided built-in solutions
+ Load validation
+ Deployment of the updated configuration to a destination device, including VELOS VM tenants
+ Post-migration diagnostics
+ Generating detailed PDF reports at every stage of the journey

Full config non-VELOS BIG-IP migrations are supported for software paths according to the following matrix:

|                 |          |         |         |   DEST  |         |         |         |
| ----------------| -------- |-------- |-------- |---------|-------- |-------- |-------- |
|                 |**X**     |**11.x** |**12.x** |**13.x** |**14.x** |**15.x** |**16.x** |
|                 | **<11.5**|  **X**  |  **X**  |  **X**  |  **X^** |  **X^** |         |
|                 | **12.x** |         |  **X**  |  **X**  |  **X**  |  **X^** |         |   
|      **SRC**    | **13.x** |         |         |  **X**  |  **X**  |  **X**  |         | 
|                 | **14.x** |         |         |         |  **X**  |  **X**  |  **X**  |
|                 | **15.x** |         |         |         |         |  **X**  |  **X**  |
|                 | **16.x** |         |         |         |         |         |  **X**  |


**X^** - an exception compared to the supported upgrade paths listed in the official document [K13845](https://support.f5.com/csp/article/K13845) (upgrade allowed only if the source configuration is upgraded to the latest available maintenance release).


> WARNING: Migrating Application Services using keys stored in FIPS cards is not supported at the moment, unless the user can restore the FIPS keys with the original (non-fips) ones on the destination platform.

### Known parity gaps
JOURNEYS finds the following configuration elements in the source configuration, which are no longer supported by the BIG-IP running on the destination platform. For every incompatible element found, there will be one or more automatic possible solutions provided.

#### Common issues

+ **Syslog** - solves [syslog configuration parsing error](https://support.f5.com/csp/article/K96275603) possible during an upgrade to a destination BIG-IP with version 13.1.0 or higher
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: Syslog
   * Available mitigations:
      * (**default**) Adjust `sys syslog` configuration depending on source version 
        * Change all characters `[` to `\\[` in field `include` (source version 12.1.5 or higher)
        * Change all characters `[` to `\\\\[` in field `include` (source version lower than 12.1.5)
      
      * Delete unsupported object
         * Remove  `sys syslog` object containing improperly escaped `[` in section `include`

   </details>
+ **TRUNK** - trunks cannot be defined on the BIG-IP VCMP guest.
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: Trunk
   * Affected BIG-IP versions: 15.x.x, 16.x.x
   * Available mitigations:
      * (**default**) Delete unsupported objects
         * Remove any `net trunk` configuration objects
   </details>
+ **CompatibilityLevel** - determines the level of platform hardware capability for device DoS/DDoS prevention. Details on the feature and the respective levels can be found [here](https://techdocs.f5.com/en-us/bigip-15-1-0/big-ip-system-dos-protection-and-protocol-firewall-implementations/detecting-and-preventing-system-dos-and-ddos-attacks.html).
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: CompatibilityLevel
   * Affected BIG-IP versions: all
   * Available mitigations:
      * (**default**) Set the device compatibility level to level supported by destination BIG-IP platform. 
      Possible scenarios:
          * Set the device compatibility level to 0
             * Adjust the `level` value in the configuration object `sys conmpatibility-level` to 0
             * Enforce software processing of the DoS protection feature by setting an appropriate BigDB database value
             * Decreasing compatibility from level 2 to level 0 requires removal `dos network-whitelist` if it's declaration contains field `extended-entries`
          * Set the device compatibility level to 1
             * Adjust the `level` value in the configuration object `sys conmpatibility-level` to 1
             * Decreasing compatibility from level 2 to level 1 requires removal `dos network-whitelist` if it's declaration contains field `extended-entries`
          * Set the device compatibility level to 2
             * Adjust the `level` value in the configuration object `sys conmpatibility-level` to 2
   </details>
   
#### Velos specific issues
+ **AAM** - Application Acceleration Manager is not supported on VELOS platform.
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: AAM
   * Affected VELOS BIG-IP versions: all
   * Available mitigations:
      * (**default**) Delete unsupported objects
         * Find any `ltm policy` objects used in virtuals used inside `wam applications`. Remove policies which contain a `wam enable` clause
         * In the same virtuals, remove any profiles of type `web-acceleration`
         * Deprovision the `am` module
   </details>
+ **CGNAT** - VELOS platform does not support [Carrier Grade NAT](https://techdocs.f5.com/en-us/bigip-15-0-0/big-ip-cgnat-implementations.html) configuration.
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: CGNAT
   * Affected VELOS BIG-IP versions: 14.1.x
   * Available mitigations:
      * (**default**) Delete unsupported objects
         * Remove any of the following configuration objects: `ltm lsn-pool`, `ltm profile pcp`, `ltm virtual` with `lsn` type and any references to them
         * Disable the `cgnat` feature module
   </details>
+ **ClassOfService** - the CoS feature is not supported on the VELOS platform.
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: ClassOfService
   * Affected VELOS BIG-IP versions: all
   * Available mitigations:
      * (**default**) Delete unsupported objects
         * Remove any of the following configuration objects: `net cos`
   </details>
+ **CompatibilityLevel** - determines the level of platform hardware capability for device DoS/DDoS prevention. VELOS currently supports only level 0 and 1 (the latter if software DoS processing mode is enabled). Details on the feature and the respective levels can be found [here](https://techdocs.f5.com/en-us/bigip-15-1-0/big-ip-system-dos-protection-and-protocol-firewall-implementations/detecting-and-preventing-system-dos-and-ddos-attacks.html).
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: CompatibilityLevel
   * Affected VELOS BIG-IP versions: 14.1.x
   * Available mitigations:
      * (**default**) Set the device compatibility level to 0
         * Adjust the `level` value in the configuration object `sys conmpatibility-level` to 0
      * Set the device compatibility level to 1
         * Adjust the `level` value in the configuration object `sys conmpatibility-level` to 1
         * Enforce software processing of the DoS protection feature by setting an appropriate BigDB database value
   </details>
+ **DoubleTagging** - indicates support for the IEEE 802.1QinQ standard, informally known as Double Tagging or Q-in-Q, which VELOS does not have as for now. More info on the feature can be found [here](https://techdocs.f5.com/en-us/bigip-14-1-0/big-ip-tmos-routing-administration-14-1-0/vlans-vlan-groups-and-vxlan.html).
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: DoubleTagging
   * Affected VELOS BIG-IP versions: all
   * Available mitigations:
      * (**default**) Delete unsupported parameters
         * Remove any `customer-tag` entries in `net vlan` objects
   </details>
+ **DeviceGroup** - "Device trust can cause UCS load errors on VELOS devices, and device groups are not automatically removed while using the reset-trust option. User will need to manually reconfigure HA after loading the UCS on new devices.
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: DeviceGroup
   * Affected VELOS BIG-IP versions: all
   * Available mitigations:
      * (**default**) Delete unsupported objects
         * Remove any `cm device-group` configuration objects and any references to them
   </details>
+ **MGMT-DHCP** - on VELOS mgmt-dhcp configuration is handled mostly on a partition level.
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: MGMT-DHCP
   * Affected VELOS BIG-IP versions: all
   * Available mitigations:
      * (**default**) Disable management DHCP
         * Set the `sys global-settings` `mgmt-dhcp` value to `disabled`
   </details>
+ **MTU** - since VELOS currently does not support jumbo frames, we have to limit mtu to 1500.
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: MTU
   * Affected VELOS BIG-IP versions: all
   * Available mitigations:
      * (**default**) Change MTU values to the maximum allowed
         * Set any found MTU values in `net vlan` configuration objects to 1500
   </details>
+ **PEM** - although keeping the PEM configuration in the configuration files will not cause load errors (it will be discarded when loading the UCS), as for now PEM is not functionally supported by VELOS. JOURNEYS removes PEM elements from the configuration to avoid confusion.
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: PEM
   * Affected VELOS BIG-IP versions: 14.1.x
   * Available mitigations:
      * (**default**) Remove unsupported objects
         * Disable provisioning of the PEM module
         * Remove any non-default configuration objects from the `pem` module
         * Remove any iRules using the following PEM-specific expressions: `CLASSIFICATION::`, `CLASSIFY::`, `PEM::`, `PSC::`
   </details>
+ **RoundRobin** - Round Robin DAG configuration is not available on VELOS guest systems.
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: RoundRobin
   * Affected VELOS BIG-IP versions: all
   * Available mitigations:
      * (**default**) Remove unsupported objects
         * Remove all settings other than `dag-ipv6-prefix-len` from the `net dag-globals` configuration object
   </details>
+ **SPDAG** - [source/destination DAG](https://techdocs.f5.com/en-us/bigip-15-1-0/big-ip-service-provider-generic-message-administration/generic-message-example/generic-message-example/about-dag-modes-for-bigip-scalability/sourcedestination-dag-sp-dag.html) is not supported on VELOS.
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: SPDAG
   * Affected VELOS BIG-IP versions: 14.1.x
   * Available mitigations:
      * (**default**) Set unsupported vlan parameters to default
         * Change `cmp-hash` values in all `net vlan` configuration objects to `default`
      * Remove all vlans having unsupported parameters
         * Remove all vlans having non-default `cmp-hash` values, as well as any references to them
   </details>
+ **sPVA** - some security Packet Velocity Acceleration (PVA) features do not have hardware support on VELOS - these either must be removed or set to use software mode.
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: sPVA
   * Affected VELOS BIG-IP versions: 14.1.x
   * Available mitigations:
      * (**default**) Delete unsupported objects
         * Remove `address-lists` defined in any `security dos network-whitelist` configuration objects
         * For any of the removed fields above, remove any referring top level `address-list` objects
         * Set `sys compatibility-level` to 0
      * Set the device compatibility level to 1
         * Adjust the `level` value in the configuration object `sys conmpatibility-level` to 1
         * Enforce software processing of the DoS protection feature by setting an appropriate BigDB database value
   </details>
+ **TRUNK** - on VELOS Trunks cannot be defined on the BIG-IP level.
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: TRUNK
   * Affected VELOS BIG-IP versions: all
   * Available mitigations:
      * (**default**) Delete unsupported objects
         * Remove any `net trunk` configuration objects
   </details>
+ **VirtualWire** - the [virtual-wire feature](https://techdocs.f5.com/kb/en-us/products/big-ip_ltm/manuals/product/big-ip-system-configuring-for-layer-2-transparency-14-0-0/01.html) is not supported on VELOS systems.
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: VirtualWire
   * Affected VELOS BIG-IP versions: all
   * Available mitigations:
      * (**default**) Delete unsupported objects
         * Remove any `net trunk` configuration objects
   </details>
+ **VlanGroup** - on VELOS [vlan-groups](https://techdocs.f5.com/en-us/bigip-14-1-0/big-ip-tmos-routing-administration-14-1-0/vlans-vlan-groups-and-vxlan.html) cannot be defined on the BIG-IP level.
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: VlanGroup
   * Affected VELOS BIG-IP versions: all
   * Available mitigations:
      * (**default**) Delete unsupported objects
         * Remove any `net vlan-group` configuration objects
   </details>
+ **VLANMACassignment** - solves an issue with mac assignment set to `vmw-compat` that can happen when migrating from a BIG-IP Virtual Edition.
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: VLANMACassignment
   * Affected VELOS BIG-IP versions: all
   * Available mitigations:
      * (**default**) Modify unsupported value to `unique`
         * Set the `share-single-mac` value in `ltm global-settings` and the respective BigDB database value to `unique`
      * Modify unsupported value to `global`
         * Set the `share-single-mac` value in `ltm global-settings` and the respective BigDB database value to `global`
   </details>
+ **WildcardWhitelist** - a part of sPVA - extended-entries field in network-whitelist objects is not supported.
   <details><summary>Details</summary>
   
   * JOURNEYS issue ID: WildcardWhitelist
   * Affected VELOS BIG-IP versions: 14.1.x
   * Available mitigations:
      * (**default**) Delete unsupported objects
         * Remove any `security network-whitelist` objects containing an `extended-entries` key
   </details>

**JOURNEYS does not support** feature parity gaps that:

+ Reside outside a UCS archive to be migrated, e.g. in a host UCS (not in a guest UCS):
    + Crypto/Compression Guest Isolation - Dedicated/Shared SSL-mode for guests is not supported on VELOS. [Feature details.](https://support.f5.com/csp/article/K22363295)
    + Traffic Rate Limiting (affects vCMP guests only) - assigning a traffic profile for vcmp guests is currently not supported on a VELOS tenant.

+ Do not cause any config load failures on VELOS:

    + [Secure Vault](https://support.f5.com/csp/article/K73034260) - keys will be instead stored on the tenant file system.
    + Several sPVA features which do not support hardware processing, where software support will occur instead (DDoS HW Vectors (Device and VS), Device/VS Block List, Device Vector Bad Actor (Greylist))
    + Wildcard SYN cookie protection - as above, software processing will replace hardware one.

----
### BIG-IP Prerequisites

Mandatory steps before running Full Config migration in JOURNEYS:

1. **Master key transfer** - to allow handling encrypted objects, before running JOURNEYS, you need to set a device master key password on both Source and Destination Systems. There are two ways to do this:

    1. Copy the Source System master key with f5mku and re-key master key on the Destination System:

       >Important: Whenever possible, use the documented tmsh commands for master and unit key manipulation.
       >Only use the f5mku command with assistance from F5 Support when no tmsh commands exist to perform
       > the task you want.

       - Obtain the master key from the Source System by entering the following commands:
         ```
         f5mku -K
         ```
         and copy the output. The command output appears similar to the following example:
         ```
         oruIVCHfmVBnwGaSR/+MAA==
         ```
       - Install the master-key that you copied in the previous step to the Destination System using the
         following syntax:
         ```
         f5mku -r <key_value>
         ```

    1.  Set master-key password before saving the source UCS file
        - Set the device master key password on the Source System by entering the following commands
          (**remember this password** because you'll need to use the same password on the destination device)
          ```
          tmsh modify sys crypto master-key prompt-for-password
          tmsh save sys config
          ```

        - Set the master key password on the Destination System by entering the following commands, using the password remembered in the previous step:
          ```
          tmsh modify sys crypto master-key prompt-for-password
          tmsh save sys config
           ```

   For more details, please refer to:
      - [Platform-migrate option overview: K82540512](https://support.f5.com/csp/article/K82540512#p1)
      - [Installing UCS files containing encrypted passwords or passphrases: K9420](https://support.f5.com/csp/article/K9420)

1. **SSH public keys migration**
   * SSH public keys for passwordless authentication may stop work after UCS migration, since the UCS file may not contain SSH public keys for users.
   * If the version is affected by the problem, then:
      - all key files have to be migrated manually from the Source System to the Destination System
      - /etc/ssh directory has to be added to the UCS backup configuration of the Source System
   * For more details on how to manually migrate SSH keys and verify if your version is affected by the problem, please read:
      - K22327083: UCS backup files do not include the /etc/ssh/ directory
      https://support.f5.com/csp/article/K22327083
      - K17318: Public key SSH authentication may fail after installing a UCS
      https://support.f5.com/csp/article/K17318
      - K13454: Configuring SSH public key authentication on BIG-IP systems (11.x - 15.x)
      https://support.f5.com/csp/article/K13454

1. **Destination System preparation for JOURNEYS**
   1. Destination BIG-IP should be in Active state.
   1. VLANs, trunks and interfaces should already be configured on vCMP and VELOS systems.
      For more details, please refer to:
      - [Platform-migrate option overview: K82540512](https://support.f5.com/csp/article/K82540512#p1)

#### BIG-IP account prerequisites
To ensure all JOURNEYS features work properly, an account with Administrator role and advanced shell (bash) access is
required on both source and target hosts. It can be `root` or any other account. For auditing purposes, a separate account
for migration might be desired.

 > IMPORTANT: Due to the above, certain features of JOURNEYS - specifically the ones requiring ssh access to the machine - are not available on BIG-IPs running in the **Appliance mode**. The user will be required to perform manual variants of these steps instead.

---
## Journey: Application Service Migration

Supported features:
+ Loading UCS source configurations.
+ Grouping Virtuals into Applications and Tenants, basing on internal dependencies and set preferences.
+ Configuration editor for Virtuals with an easy access to updated AS3 conversion previews.
+ Converting selected applications into a deployable AS3 declaration using [f5-as3-config-converter](https://github.com/f5devcentral/f5-as3-config-converter/).
+ Deployment of the updated configuration to a specified destination device via AS3.
+ Post-migration diagnostics.

### Configuration object grouping

1. **Virtual** - smallest object recognized by JOURNEYS. It includes a single `ltm virtual-server` (represented by a Service in AS3) object and any others referenced by it directly or indirectly - monitors, pools, profiles, etc.

2. **Application** - A group of AS3 objects, including the virtual mentioned above. Logic for initial grouping of virtuals into apps can be manually set via preferences in GUI before loading the ucs. Represented by `folders` on the BIG-IP level.

3. **Tenant** - A group of AS3 applications. By default JOURNEYS creates one tenant per application. `Common` is a special reserved name, represeting objects shared between tenants. Represented by `partitions` on the BIG-IP level.

### Application conversion status
Each virtual separately gets assigned a status based on the f5-as3-config-converter response. Possible statuses are as follows:

* Green - All virtual objects appear to convert properly.
* Red - Virtual configuration includes some objects that are currently marked as `unsupported` by f5-acc-config-converter, and are considered undeployable.
* Black - Error during a virtual config conversion attempt.

> Note: If one or more of your apps have a red or black status, you may attempt to use a newer version of f5-acc-config-converter by editing the image version inside the `docker-compose.yml` file. Otherwise, please open an issue on [f5devcentral](https://github.com/f5devcentral/f5-journeys/issues) and include configuration contents from the problematic app.

### Additional notes

#### Deployable objects
Not all required configuration files can be included inside the resulting AS3 json, and need to be installed on the Destination System prior to sending the declaration itself. 

If deploying manually, JOURNEYS will prepare a package containing all of the required files alongside with a list of commands to perform on the Destination System.

If deploying via JOURNEYS, the application will install the files automatically.

#### Known issues

* JOURNEYS does not support keys generated using a physical FIPS card. If there are any applications referring to these keys, deployment of them will fail.

----
## Configuration Migration Considerations

### BIG-IP device swap
To minimize downtime, F5 recommends deploying the new hardware alongside existing BIG-IP deployment.

F5 recommends the following procedure for moving production traffic to a new device:

1. Deploy the target device, trying to keep physical connections as close as possible to the old BIG-IP (respective interfaces assigned to the same physical networks)
1. Remove interfaces to existing VLANs on the **new hardware** (this will impact **all** tenants on VELOS/vCMP guests).
   <br/>There are three options to do this:
    1.  Disabling interfaces.
    1.  Physically unplugging the network cables.
    1.  Disabling the port on the switch connected to the destination platform.
1. Deploy JOURNEYS-generated config to a new BIG-IP. Note that some validators like LTM module comparison are expected to fail as Virtual Servers will be down.
1. If the configuration was deployed successfully, review the system status. If the status is "REBOOT REQUIRED", perform the reboot before shutting down interfaces on the old BIG-IP system.
1. If you use BIG-IQ, refer to the [article K15938](https://support.f5.com/csp/article/K15938) to discover your new BIG-IP device.
1. Shutdown interfaces on the **old BIG-IP system** (this will impact **all** source BIG-IPs). 
   <br/>There are three options to do this:
    1. Disabling interfaces.
    1. Physically unplugging the network cables.
    1. Disabling the port on the switch connected to the old BIG-IP system.
1. Re-add interfaces to existing VLANs on the **new hardware** (this will impact **all** tenants on VELOS/vCMP guests).
    <br/>There are three options to do this:
    1. Re-enable interfaces on the destination platform.
    1. Physically unplug the network cables on the source platform, plug the network cables on the destination platform.
    1. Enable the port on the switch connected to the destination platform.

### SPDAG/VlanGroup mitigation
If SPDAG or VlanGroup removal mitigation is applied, and a conflicted object is configured on a Virtual Server, JOURNEYS **will remove all VLANs assigned for that particular Virtual Server** - not only the conflicted one.
This is done to ensure that JOURNEYS does not produce an invalid configuration (Virtual Servers cannot share identifiers, as they need to be unique).

### Restoring backup UCS in case of migration failure
Deployment progress can be tracked on the Summary page.
If UCS load fails, it is strongly advised to manually load the backup UCS archive created during the deployment process before any other action is taken on the Destination System.

   1. Log to the Destination System:
      ```
      ssh username@<destination-system-ip>
      ```
   1. Use the command to load the backup UCS:
      ```
      tmsh load sys ucs <backup_ucs_name*>
      ```
      *The backup UCS archive name is displayed on the Summary page in the Detailed Results in a log entry that should look as follows:
      ```
      Create backup of the Destination System: /var/local/ucs/journey_ucs_backup_20211012-103937.ucs
      ```

----
## JOURNEYS Setup Requirements
* Install [Docker](https://docs.docker.com/get-docker/)
* Install [Docker Compose](https://docs.docker.com/compose/install/)

## Usage

### Fetching JOURNEYS

```
git clone https://github.com/f5devcentral/f5-journeys.git
cd f5-journeys
```

### Preparing the environment
1. Create a directory for all of JOURNEYS operations - modifying configs, logging, etc.

   You can use any directory in place of `/tmp/journeys`. 
   ```
   mkdir /tmp/journeys
   ```
   > Note: keep in mind that the default migration directory is placed in the `/tmp` directory, which is cleaned up upon a system reboot. If longer session persistence is required, please modify this directory path via the .env file to a non-temporary one.

1. Prepare an environment file for docker-compose
   ```
   cp sample.env .env
   ```
   If you are using a different working directory than the one shown in the point above, `WORKING_DIRECTORY` variable has to be updated in the `.env` file

1. Fetch services included in the docker-compose configuration file
   ```
   docker-compose pull
   ```

1. Print sha digest of downloaded images
   ```
   docker image ls f5devcentral/f5-bigip-journeys-app --digests
   ```

1. Examine if image listed in docker-compose journeys/celery-worker has exactly the same digest that can be found on docker hub repository with the same tag
   ```
   https://hub.docker.com/r/f5devcentral/f5-bigip-journeys-app/tags?page=1&ordering=last_updated
   ```
   > Note: do not use the application if digests does not match.

1. Start services included in the docker-compose configuration file
   ```
   docker-compose up -d
   ```
   > Note: services included in the default docker-compose.yaml file do not allow usage of the `perapp` functionalities. To use them, please refer to [perapp documentation](PERAPP.md).

1. Open your web browser, navigate to:
   ```
   https://localhost:8443
   ```
   and accept self-signed cert to run the application. The certificate itself can be verified by comparing it with the one logged by the main JOURNEYS container.
   ```
   docker logs f5-journeys_journeys_1 2>&1 | grep 'BEGIN CERTIFICATE'
   ```




## Feature details

### Diagnostics
Upon deployment of UCS to the Destination System, user will be prompted to select several diagnose methods. These collect information relevant to your system condition and compare its state and configuration with the Source BIG-IP System.

Please keep in mind that only a few of the diagnostics methods can be run without providing a Source System. If the UCS was provided manually and no such system was provided, unavailable options will be locked out.

> IMPORTANT: User will always be prompted for source and destination credentials right before deployment even if Source System is offline. Destination credentials are valid before loading UCS archive on the target system. Once the UCS is loaded, source credentials (stored at UCS archive provided by a user or downloaded from the Source System) are required to log in.

Details of the diagnostic checks can be evaluated in a resulting pdf report.

<details>
<summary>Diagnose methods</summary>

* ### MCP status check

   Area:| error detection
   -----|-----

   Checks if values of returned fields are correct.
   This method uses `tmsh show sys mcp-state field-fmt` that can be executed manually.

* ### TMM status

   Area:| resource management
   -----|-----

   Function logs status of TMM. Requires manual evaluation.

* ### Prompt state

   Area:| error detection
   -----|-----

   Checks if prompt state is in active mode.

* ### Core dumps detection

   Area:| error detection
   -----|-----

   Checks if diagnostic core dumps were created.

* ### Database Comparison

   Area:| config migration
   -----|-----

   Compares two system DBs getting them from iControl endpoint for sys db. Requires manual evaluation.

* ### Memory footprint Comparison

   Area:| information, resource management
   -----|-----

   Compares information from `tmsh show sys provision` for both systems. Requires manual evaluation.

* ### Version Comparison

   Area:| information
   -----|-----

   Compares information from `tmsh show sys version` for both systems. Requires manual evaluation.

* ### Local Traffic Manager (LTM) module comparison checks

   Area:| config migration, resource management
   -----|-----

   Check lists of all defined LTM nodes and Virtual Servers configured in the new system.
   If both devices are on-line, it will check conformance of both configuration and resource availability.
   Requires manual evaluation.

* ### Log Watcher check

   Area:| error detection
   -----|-----

   Log watcher checks the difference in log files before and after UCS deployment.  This check looks for "ERR" and "CRIT" phrases (case-insensitive) that might appear in logs during the deployment. Current scope of log file lookup:
   - /var/log/ltm"
   - /var/log/apm"
   - /var/log/gtm"
   - /var/log/tmm"
   - /var/log/liveinstall.log
   - /var/log/asm"
   - /var/log/ts/bd.log
   - /var/log/ts/asm_config_server.log
   - /var/log/ts/pabnagd.log
   - /var/log/ts/db_upgrade.log
   - /var/log/daemon.log
   - /var/log/kern.log
   - /var/log/messages

   The log watcher output only includes these files, which appear on your system and are provisioned.
   Sample output:
   ```json
   Log watcher output:
   {
      "/var/log/ltm": [],
   }
   ```
   Empty list as a value means no lines containing phrases "ERR" and "CRIT" were found. However, there may be information about potential problems in logs.
   Therefore, this check requires manual evaluation.

* ### Ciphersuites check

   Area:| information
   -----|-----

   Compares information about ciphersuites for both systems. Requires manual evaluation.

* ### Cron entries check

   Area:| error detection
   -----|-----

   Compares cron data between Source and Destination Systems, checking for any unexpected changes.

* ### Platform migrate ignored objects

   Area:| information
   -----|-----

   Lists objects ignored due to the usage of `platform migrate` ucs load  option. Requires manual evaluation.

* ### Services check

   Area:| error detection
   -----|-----

   Verifies whether any services on the Destination System are stuck in a restart loop. If they are, attempt to fix the issue by rebooting the device.

   > WARNING: this check can include a reboot. Using it might significantly increase validation runtime, and can cause unexpected changes on the target device.

</details>

### Load validation

During the migration process, user will be provided with an option to validate if the configuration being migrated will load on the provided Destination System. This operation utilizes the BIG-IP built in `tmsh sys config verify` functionality to check whether the Destination BIG-IP finds any errors in the provided files.
This module temporarily modifies Destination configuration files for testing purposes. After moving on from this step, the Destination System will be reverted back to its original state.

> WARNING: Due to how `tmsh sys config verify` works, the load validation module has to temporarily heavily modify configuration files on the provided Destination System. For this reason, to avoid any potential loss of data, it is _not_ recommended to use this feature with a device already containing any non-basic configuration.

> WARNING: Please make sure the Destination device is licensed and provisioned with modules corresponding to the UCS configuration being migrated.

> WARNING: Load validation can produce false positive output if the configuration contains any parts subject to fix-up scripts built into the BIG-IP ucs load process. These fix-ups cannot be automatically incorporated in the validation module.

<details>
<summary>Load validation functional details</summary>

Since the `verify` command requires an unpacked and ready to load configuration on a live system, this module needs to perform a few preliminary steps before actual validation. The whole process looks as follows:

1. Modify the configuration files:
   * Remove several objects from the `net` module: `interface`, `stp`, `vlan`, `fdb vlan`, `trunk`
   * Remove any references to the removed objects
   * Disable user session limit
1. Generate a temporary UCS with the current configuration and upload it to the provided device
1. Generate a backup UCS
1. Extract a part of the UCS file on the provided host, including but not limited to:
   * Configuration files
   * Certificates and keys
   * Database files
1. If the migration includes an AS3 file, check and potentially install the appsvcs package
1. Call the `tmsh load sys config verify partitions all` command and check its output. 
Additionally, verify whether mcpd is in correct state after this operation.
1. If the migration includes an AS3 file, send a request with modified declaration having its action set as `dry-run` and verify the output
   * if declaration is AS3 format then only action is changed to `dry-run`
   * if declaration is ADC then AS3 section is added with action `dry-run` and other required settings
   https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/composing-a-declaration.html

After the validation is complete, the test host is then restored to the original state:
1. Restore the backup ucs
1. Remove any files that were extracted from the verified UCS file, but were not present originally on the system
1. Remove the test UCS


</details>

## Contributing

### Bug reporting

Let us know if something went wrong. By reporting issues, you support development of this project and get a chance of having it fixed soon.
Please use bug template available [here](https://github.com/f5devcentral/f5-journeys/issues/new?assignees=&labels=&template=bug_report.md&title=%5BBUG%5D) and attach the journeys.log file from the working directory (`/tmp/journeys` by default)

### Feature requests

Ideas for enhancements are welcome [here](https://github.com/f5devcentral/f5-journeys/issues/new?assignees=&labels=&template=feature_request.md&title=%5BFEAT%5D)
