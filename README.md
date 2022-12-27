# Configuring-Junos-Devices-using-Netconf-and-YANG-Model
* In this wiki, I will explain how to configure Junos devices using NETCONF and YANG Model
* Introduction about YANG can be obtained from the [document](https://www.juniper.net/documentation/us/en/software/junos/netconf/topics/concept/netconf-yang-modules-overview.html#:~:text=YANG%20data%20models%20comprise%20modules,and%20constraints%20on%20that%20data.)
* Introduction about NETCONF can be obtained from the [document](https://www.juniper.net/documentation/en_US/bti-series/bti78004.4/topics/concept/c-7800-swicg-about-netconf.html)
## Getting YANG Model From Junos Device
```
file make-directory /var/tmp/yang
show system schema module all output-directory /var/tmp/yang
start shell
cd /var/tmp
tar czf yang.tgz yang/*
```
* Downloading YANG Models on the MGMT Client

```
scp user@junos-device-ip:/var/tmp/yang.tgz  .
tar -xzf yang.tgz
```

## Installing Necessary Tools on the Mgmt Client (Ubuntu 20.04)
```
sudo apt-get install yang-tools
sudo apt install python3-pip
pip3 install netconf-console2
```
## Adding Required Config in Junos
* Junos config generated via Netconf RPC calls will not have required Namespace in the generated config so it can't be validated against YANG model
* Add following  config in Junos Device

```
set system services netconf rfc-compliant
set system services netconf yang-compliant
```
## Getting Config from Junos Device and Validating it against Relvant YANG Model 
* We can get config from the Junos device as xml output, which is the default data structure for Netconf RPC or as CLI curly brackets hierarchical format or set format.
* An example of each above named method  is given below.
* Output will be shown on mgmt client terminal and it can directed to file using ">" operator
```
netconf-console2 --host ${junos-device-mgmt-ip} --port 830 --user ${user-name} --password ${password} --rpc get-config-xml.xml
netconf-console2 --host ${junos-device-mgmt-ip} --port 830 --user ${user-name} --password ${password} --rpc get-config-set.xml
netconf-console2 --host ${junos-device-mgmt-ip} --port 830 --user ${user-name} --password ${password} --rpc get-config-cli.xml
```
* Getting physical interface Config 

```
netconf-console2 --host ${junos-device-mgmt-ip} --port 830 --user ${user-name} --password ${password} --rpc get-config-ifd.xml

<?xml version='1.0' encoding='UTF-8'?>
<nc:rpc-reply xmlns:junos="http://xml.juniper.net/junos/22.4R0/junos" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:62097e6b-e1f5-4712-b38a-ed665366dda7">
<configuration xmlns="http://yang.juniper.net/junos/conf/root" junos:changed-seconds="1672101257" junos:changed-localtime="2022-12-26 16:34:17 PST">
    <interfaces xmlns="http://yang.juniper.net/junos/conf/interfaces">
        <interface>
            <name>et-0/0/1</name>
            <description>remote:C1-R2:et-0/0/1</description>
            <mtu>9216</mtu>
            <unit>
                <name>0</name>
                <family>
                    <inet>
                        <address>
                            <name>192.168.1.2/31</name>
                        </address>
                    </inet>
                    <iso>
                    </iso>
                    <inet6>
                        <address>
                            <name>fc00:0:0:1::/127</name>
                        </address>
                    </inet6>
                    <mpls>
                    </mpls>
                </family>
            </unit>
        </interface>
    </interfaces>
</configuration>
</nc:rpc-reply>
```
* Change the format of above output as per following.
```
cat > ifd-config.xml << EOF
<configuration xmlns="http://yang.juniper.net/junos/conf/root">
    <interfaces xmlns="http://yang.juniper.net/junos/conf/interfaces">
        <interface>
            <name>et-0/0/1</name>
            <description>remote:C1-R2:et-0/0/1</description>
            <mtu>9216</mtu>
            <unit>
                <name>0</name>
                <family>
                    <inet>
                        <address>
                            <name>192.168.1.2/31</name>
                        </address>
                    </inet>
                    <iso>
                    </iso>
                    <inet6>
                        <address>
                            <name>fc00:0:0:1::/127</name>
                        </address>
                    </inet6>
                    <mpls>
                    </mpls>
                </family>
            </unit>
        </interface>
    </interfaces>
</configuration>
EOF
```
* Use "yanglint" to compare the above formatted output with the available YANG model.
```
yanglint --strict ifd-config.xml junos-conf-interfaces@2022-01-01.yang
```
* Above operation did not generate  any error message so the physical interface et-0/0/1 config is validated against the junos-conf-interfaces@2022-01-01.yang YANG model.

## Config Push Operation into Junos Device
* Netconf client can upload config in Junos device either in xml format,  set format or Junos Config hierarchy
* Preparing Config in xml format 
```
<edit-config>
    <target>
      <candidate />
    </target>
    <default-operation>merge</default-operation>
    <config>
	    <configuration>
               <put your text config here/>
           </configuration>
    </config
 </edit-config>
```
* Prepare config in Junos Config hierarchy or Set format

```
<edit-config>
    <target>
      <candidate />
    </target>
    <default-operation>merge</default-operation>
    <config-text>
	    <configuration-text>
               <put your text config here/>
           </configuration-text>
    </config-text>
 </edit-config>
```

## Example Config for et-0/0/1 interface with slight change in Interface Description
```
cat > config-ifd-prep << EOF
<edit-config>
    <target>
      <candidate />
    </target>
    <default-operation>merge</default-operation>
    <config-text>
	    <configuration-text>
               interfaces {
    et-0/0/1 {
        description test;
        mtu 9216;
        unit 0 {
            family inet {
                address 192.168.1.2/31;
            }
            family iso;
            family inet6 {
                address fc00:0:0:1::/127;
            }
            family mpls;
        }
    }
}
           </configuration-text>
    </config-text>
 </edit-config>
EOF
```
## Pushing Config
```
netconf-console2 --host ${junos-device-mgmt-ip} --port 830 --user ${user-name} --password ${password}  --rpc config-ifd-prep.xml
<?xml version='1.0' encoding='UTF-8'?>
<nc:rpc-reply xmlns:junos="http://xml.juniper.net/junos/22.4R0/junos" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:ad836d03-3805-4d88-b247-d2e57a432a43">
<nc:ok/>
</nc:rpc-reply>
```
## Compare Candidate Config with Running Config

```
netconf-console2 --host ${junos-device-mgmt-ip} --port 830 --user ${user-name} --password ${password}  --rpc compare-candidate.xml
<?xml version='1.0' encoding='UTF-8'?>
<nc:rpc-reply xmlns:junos="http://xml.juniper.net/junos/22.4R0/junos" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:fcb9b41a-2b97-4343-a0c7-e20ed51163e9">
<configuration xmlns="http://xml.juniper.net/xnm/1.1/xnm" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" xmlns:yang="urn:ietf:params:xml:ns:yang:1">
    <interfaces>
        <interface>
            <name>et-0/0/1</name>
            <description nc:operation="delete"/>
            <description nc:operation="create">TEST</description>
        </interface>
    </interfaces>
</configuration>
</nc:rpc-reply>
```
## Config Validatation
```
netconf-console2 --host ${junos-device-mgmt-ip} --port 830 --user ${user-name} --password ${password} --rpc validate-config.xml
<?xml version='1.0' encoding='UTF-8'?>
<nc:rpc-reply xmlns:junos="http://xml.juniper.net/junos/22.4R0/junos" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:9c7c0688-0c7b-4ae6-b530-e37e0460334e">
<nc:ok/>
</nc:rpc-reply>
```
## Config Commit
```
netconf-console2 --host ${junos-device-mgmt-ip} --port 830 --user ${user-name} --password ${password} --rpc commit-config.xml
<?xml version='1.0' encoding='UTF-8'?>
<nc:rpc-reply xmlns:junos="http://xml.juniper.net/junos/22.4R0/junos" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:ec6ec732-3e6e-43fc-84bd-7ec346bf4095">
<nc:ok/>
</nc:rpc-reply>
```
## Config Rollback

```
netconf-console2 --host ${junos-device-mgmt-ip} --port 830 --user ${user-name} --password ${password}  --rpc discard-changes.xml
<?xml version='1.0' encoding='UTF-8'?>
<nc:rpc-reply xmlns:junos="http://xml.juniper.net/junos/22.4R0/junos" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:409c7a96-af34-4b15-84af-2d03ec8141d5">
<nc:ok/>
</nc:rpc-reply>
```
## Closing the Session

```
netconf-console2 --host ${junos-device-mgmt-ip} --port 830 --user ${user-name} --password ${password} --rpc close-session.xml
<?xml version='1.0' encoding='UTF-8'?>
<nc:rpc-reply xmlns:junos="http://xml.juniper.net/junos/22.4R0/junos" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:e0eeffa9-bd86-4396-98b5-f35719a4ab23">
<nc:ok/>
</nc:rpc-reply>
Operation failed: SessionCloseError - Unexpected session close
IN_BUFFER: `b'<!-- session end at 2022-12-26 20:26:35 PST -->\n'`
```
