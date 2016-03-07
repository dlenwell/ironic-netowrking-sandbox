Devstack + Ironic + Arista V-EOS
================================
Instructions for standing up devstack with ironic and a virtual
Arista switch. This will allow you to use Devstack and ironic with
both virtual (ssh driver) and physical nodes.
_________
Prerequisites:
--------------
- You will require a computer with plenty of memory and cpus..
   (*minimum of 4 cores and 16gb of ram*)
- You will need a basic understanding of linux, virsh, and devstack
- Ubuntu trusty

**Ready? 1,2,3 GO!**

**1.) Register with Arista and download stuff.**

- register link:  https://www.arista.com/en/user-registration
- download page: https://www.arista.com/en/support/software-download
    - vEOS-lab-4.14.9.1M.vmdk
    - Aboot-veos-2.1.0.iso

**2.) Create brbm bridge**

Do this manually so that the veos vm can already be connected to it
before devstack, stacks.
	
There is a bridge template in the ironic repository, use the
virsh `net-define` command to define the bridge on your system from
the template.

	virsh # net-define ./ironic/devstack/tools/ironic/templates/brbm.xml

You can verify the bridge is active with the net-list command.

	virsh # net-list --all
	 Name                 State      Autostart     Persistent
	----------------------------------------------------------
	 brbm                 inactive   no            no
	 default              active     yes           yes

You'll need to make sure it is started before the next step.. if it says
"inactive" under "State" then go ahead and kick it with:

    net-start brbm 

You can also tell it to autostart if you plan on this env being a little
more persistent with:

    net-autostart brbm  


**3.) Define a Virtual machine for v-eos**

it will boot to the Arista V-Eos image we downloaded earlier.
	
There are many ways to do this. Lots of folks will have their own opinions.
I like to create an xml file and use virsh to manually control vm's in my 
development env.. 

Here is a sample of the xml file that I used here:

    <domain type='kvm'>
      <name>veos</name>
      <memory unit='Gb'>2</memory>
      <currentMemory unit='Gb'>2</currentMemory>
      <vcpu placement='static'>1</vcpu>
      <resource>
        <partition>/machine</partition>
      </resource>
      <os>
        <type arch='x86_64' machine='pc-i440fx-1.5'>hvm</type>
        <boot dev='cdrom'/>
        <boot dev='hd'/>
      </os>
      <features>
        <acpi/>
        <apic/>
        <pae/>
      </features>
      <clock offset='utc'/>
      <on_poweroff>destroy</on_poweroff>
      <on_reboot>restart</on_reboot>
      <on_crash>restart</on_crash>
      <devices>
        <emulator>/usr/bin/qemu-system-x86_64</emulator>
        <disk type='file' device='disk'>
          <driver name='qemu' type='vmdk'/>
          <source file='/files/veos/vEOS-lab-4.14.9.1M.vmdk'/>
          <target dev='hda' bus='ide'/>
          <alias name='ide0-0-0'/>
          <address type='drive' controller='0' bus='0' target='0' unit='0'/>
        </disk>
        <disk type='file' device='cdrom'>
          <driver name='qemu' type='raw'/>
          <source file='/files/veos/Aboot-veos-2.1.0.iso'/>
          <target dev='hdc' bus='ide'/>
          <readonly/>
          <alias name='ide0-1-0'/>
          <address type='drive' controller='0' bus='1' target='0' unit='0'/>
        </disk>
        <controller type='usb' index='0'>
          <alias name='usb0'/>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x2'/>
        </controller>
        <controller type='pci' index='0' model='pci-root'>
          <alias name='pci0'/>
        </controller>
        <controller type='ide' index='0'>
          <alias name='ide0'/>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x1'/>
        </controller>
        <interface type='bridge'>
          <source bridge='brbm'/>
          <model type='e1000'/>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
          <virtualport type='openvswitch' />
        </interface>
        <serial type='pty'>
          <source path='/dev/pts/4'/>
          <target port='0'/>
          <alias name='serial0'/>
        </serial>
        <console type='pty' tty='/dev/pts/4'>
          <source path='/dev/pts/4'/>
          <target type='serial' port='0'/>
          <alias name='serial0'/>
        </console>
        <input type='mouse' bus='ps2'/>
        <graphics type='vnc' port='5900' autoport='yes' listen='localhost'>
          <listen type='address' address='localhost'/>
        </graphics>
        <memballoon model='virtio'>
          <alias name='balloon0'/>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
        </memballoon>
      </devices>
      <seclabel type='dynamic' model='apparmor' relabel='yes'/>
    </domain>
	
**Note the interface section:**

        <interface type='bridge'>
          <source bridge='brbm'/>
          <model type='e1000'/>
          <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
          <virtualport type='openvswitch' />   <-- that line is important
        </interface>

Then use the `define` command in virsh:

    virsh # define veos.xml 
    Domain veos defined from veos.xml

    virsh # start veos
    Domain veos started

Now you can use a vnc client to connect to the vm and insure that it
starts correctly and poke around the terminal to make sure its connected
to the right networks.

4.) Devstack + Ironic 

Use [this](http://docs.openstack.org/developer/ironic/dev/dev-quickstart.html#deploying-ironic-with-devstack) handy guide.

Sample `local.conf`:

    [[local|localrc]]
    # Credentials
    ADMIN_PASSWORD=password
    DATABASE_PASSWORD=password
    RABBIT_PASSWORD=password
    SERVICE_PASSWORD=password
    SERVICE_TOKEN=password
    SWIFT_HASH=password
    SWIFT_TEMPURL_KEY=password

    # Enable Ironic plugin
    enable_plugin ironic git://git.openstack.org/openstack/ironic

    # Enable Neutron which is required by Ironic and disable nova-network.
    disable_service n-net
    disable_service n-novnc
    enable_service q-svc
    enable_service q-agt
    enable_service q-dhcp
    enable_service q-l3
    enable_service q-meta
    enable_service neutron

    # Enable Swift for agent_* drivers
    enable_service s-proxy
    enable_service s-object
    enable_service s-container
    enable_service s-account

    # Disable Heat
    disable_service heat h-api h-api-cfn h-api-cw h-eng

    # Disable Cinder
    disable_service cinder c-sch c-api c-vol

    # Swift temp URL's are required for agent_* drivers.
    SWIFT_ENABLE_TEMPURLS=True

    # Create 3 virtual machines to pose as Ironic's baremetal nodes.
    IRONIC_VM_COUNT=3
    IRONIC_VM_SSH_PORT=22
    IRONIC_BAREMETAL_BASIC_OPS=True
    IRONIC_DEPLOY_DRIVER_ISCSI_WITH_IPA=True

    # Enable Ironic drivers.
    IRONIC_ENABLED_DRIVERS=fake,agent_ssh,agent_ipmitool,pxe_ssh,pxe_ipmitool

    # Change this to alter the default driver for nodes created by devstack.
    # This driver should be in the enabled list above.
    IRONIC_DEPLOY_DRIVER=pxe_ssh

    # The parameters below represent the minimum possible values to create
    # functional nodes.
    IRONIC_VM_SPECS_RAM=1024
    IRONIC_VM_SPECS_DISK=10

    # Size of the ephemeral partition in GB. Use 0 for no ephemeral partition.
    IRONIC_VM_EPHEMERAL_DISK=0

    # To build your own IPA ramdisk from source, set this to True
    IRONIC_BUILD_DEPLOY_RAMDISK=False

    VIRT_DRIVER=ironic

    # By default, DevStack creates a 10.0.0.0/24 network for instances.
    # If this overlaps with the hosts network, you may adjust with the
    # following.
    NETWORK_GATEWAY=10.1.0.1
    FIXED_RANGE=10.1.0.0/24
    FIXED_NETWORK_SIZE=256

    # Log all output to files
    LOGFILE=/opt/stack/devstack.log
    LOGDIR=/opt/stack/logs
    IRONIC_VM_LOG_DIR=/opt/stack/ironic-bm-logs

Once your local.conf file is how you like it Go ahead and stack it up.

	./stack.sh

**5.) The Arista ml2 mechanical driver**

Arista doesn't seem to have a pre-developed devstack plugin we are
going to install it manually. That process is detailed
[here](https://wiki.openstack.org/wiki/Arista-neutron-ml2-driver) .

- Clone the mech driver from git into the proper place
 
        git clone https://github.com/sukhdevkapur/neutron.git /opt/stack/neutron/neutron/plugins/ml2/drivers/arista/

- Edit neutron configs to enable the arista mech driver
    -  /etc/neutron/neutron.conf
    -  /etc/neutron/plugins/ml2/ml2_conf.ini
    
- Edit /etc/neutron/plugins/ml2/ml2_conf_arista.ini:

        [ml2_arista]
        # (StrOpt) EOS IP address. This is required field. If not set, all
        #          communications to Arista EOS will fail
        #
        # eapi_host =
        # Example: eapi_host = 192.168.0.1
        #
        # (StrOpt) EOS command API username. This is required field.
        #          if not set, all communications to Arista EOS will fail.
        #
        # eapi_username =
        # Example: arista_eapi_username = admin
        #
        # (StrOpt) EOS command API password. This is required field.
        #          if not set, all communications to Arista EOS will fail.
        #
        # eapi_password =
        # Example: eapi_password = my_password
        #
        # (StrOpt) Defines if hostnames are sent to Arista EOS as FQDNs
        #          ("node1.domain.com") or as short names ("node1"). This is
        #          optional. If not set, a value of "True" is assumed.
        #
        # use_fqdn =
        # Example: use_fqdn = True
        #
        # (IntOpt) Sync interval in seconds between Quantum plugin and EOS.
        #          This field defines how often the synchronization is performed.
        #          This is an optional field. If not set, a value of 180 seconds
        #          is assumed.
        #
        # sync_interval =
        # Example: sync_interval = 60
        #
        # (StrOpt) Defines Region Name that is assigned to this OpenStack Controller.
        #          This is useful when multiple OpenStack/Neutron controllers are
        #          managing the same Arista HW clusters. Note that this name must
        #          match with the region name registered (or known) to keystone
        #          service. Authentication with Keysotne is performed by EOS.
        #          This is optional. If not set, a value of "RegionOne" is assumed.
        #
        # region_name =
        # Example: region_name = RegionOne

- Restart neutron.. this time referencing the additional config file

        cd <neutron_path> && python <neutron_path>/bin/neutron-server --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini --config-file /etc/neutron/plugins/ml2/ml2_conf_arista.ini

You should be in business... you can log into the arista switch with vnc to
get a better look at what is happening when you do stuff within openstack.

___

extra credit
------------

If you want to connect this stack with ironic to a real
physical network with bare metal nodes and take advantage of the ml2 network
isolation. You can link a trunk port to a switch that supports 802.1q (dynamic
vlans) and bridge it into your Arista virtual eos switch.

**Setup interface for link to external network** *(still in progress)*

In order to connect the v-eos switch to a physical network we just create new
bridge that is linked to the interface that is connected to a trunk port on
physical switch that supports 802.1q for both the BMC network and the tennant
network.

For each network you want to bridge in:

- Take down the interface in question.. 

        sudo ifdown eth1 

- edit the interfaces file	

        sudo vi  /etc/network/interface 
	
- Comment out the portion of the file that defines the interface in question

	    #auto ethx
	    #iface ethx inet dhcp

- add the following:

    	auto brbmten
    	iface brbmten inet manual
       		bridge_ports ethx
    		bridge_stp off   
    		bridge_fd 0
    		bridge_maxwait 0

- Save the file and exit your editor
- turn the new interface on

	    sudo ifup brbmten

- verify that the new bridge is configured correctly

		> brctl show
		bridge name	bridge id		STP enabled	interfaces
		brbmten		8000.xx		    no			ethx

Now you can edit the xml file for the veos vm we worked on above. Adding interface tags for each of the other networks you want then undefine and define the xml file again. 
