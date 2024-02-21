# Deploy SGX support in openstack devstack

## Install:

1) Clone the devstack. Create an appropriate ``local.conf`` with access to network interfaces, passwords and everything else you like. Add ``LIBS_FROM_GIT=python-novaclient,python-openstackclient`` to the end of the ``local.conf``. Use ``export FORCE=yes`` and ``stack.sh`` to install the devstack on a recent Ubuntu version.
1) In ``nova/`` do ``git checkout 3209f6551652cff7bef0b9d9719ab940dd05a0f8``
1) Apply patches to ``nova/``, ``python-novaclient/`` and ``python-openstackclient/``
1) Set ``security_driver = "none"`` in ``/etc/libvirt/qemu.conf``
1) ``sudo systemctl restart libvirtd``
1) Add ``sgx_epc_mb = <TOTAL EPC MB>`` to the ``[libvirt]`` section in ``/etc/nova/nova-cpu.conf`` (in this example 64000)
1) ``sudo systemctl restart "devstack@*"``
1) ``source openstack.rc``
1) ``openstack keypair create --public-key <path to key>.pub cloud-key``
1) ``openstack flavor create --ram 4096 --vcpus 2 --disk 20 --property resources:CUSTOM_SGX_EPC_MB=3 --property trait:HW_CPU_X86_SGX='required' sgx-2c-4g`` (only 3MB SGX memory is allocated in this example)
1) ``openstack image create --file <file path of SGX-enabled vm image> --disk-format <file format> --container-format bare --property usage_type='common' --property os_distro=others --property hw_qemu_guest_agent=no --property os_admin_user=root --property trait:HW_CPU_X86_SGX='required' --public sgx-vm-image``
1) ``openstack server create --flavor sgx-2c-4g --network private --image sgx-vm-image  --key-name cloud-key sgx-virtual-instance``

Test executable:
``https://github.com/ethernity-cloud/mvp-pox-node/raw/master/utils/linux/test-sgx``

## Troubleshooting:

* ``nova.exception.NoValidHost: No valid host was found.`` -> 
    * A possible race condition was triggered. Try again just to be sure.
    * If the error persists, check whether the ``n-cpu`` service is down and restart it if needed.
    * If the error persists, check if the ``CUSTOM_SGX_EPC_MB`` is created, and available in the provider inventory with ``openstack resource provider inventory list <UUID>``. Also check if the reported values are appropriate.
    * If the error still persists, check the logs of ``n-cpu`` ``n-sch`` and ``n-super-cond`` with ``journalctl -u devstack@n-cpu.service --since=today | tail -n 1000 | less``

* Horizon hangs in ``Spawning`` or ``Shutting down`` state -> Check status of and restart ``devstack@n-cpu`` service if needed.

* If ``n-cpu`` crashes constantly with ``update conflict: Inventory for 'CUSTOM_SGX_EPC_MB' on resource provider`` -> Make sure ``/etc/nova/nova-cpu.conf`` contains a valid value for ``sgx_epc_mb``

* ``nova.exception.MaxRetriesExceeded: Exceeded maximum number of retries.`` -> This can have **a lot** of reasons and horizon is not gonna tell you which. You can try to find out by reading the nova-cpu service logs using ``journalctl -u devstack@n-cpu.service --since=today | tail -n 1000 | less``. If that is not conclusive try to read other logs by replacing the service name with ``n-api``, ``n-sch``, ``n-super-cond`` etc.

* ``nova.exception.HypervisorUnavailable: Connection to the hypervisor is broken on host`` -> Libvirt config is broken. Check ``/var/log/libvirt/libvirtd.log``. Possibliy related to AppArmor (or SELinux). Check config and restart ``libvirtd`` service

The following part is WIP:

* ``libvirt.libvirtError: internal error: cannot update AppArmor profile`` ->
    * You have disabled AppArmor, but libvirt still wants to use it. Either set ``security_driver = "none"`` in ``/etc/libvirt/qemu.conf``, or enable AppArmor.
    * OR: Some broken apparmor profiles are loaded. Check ``sudo aa-status``. Delete and unload unused ``/etc/apparmor.d/libvirt/`` profiles...

* ``unsupported configuration: Security driver apparmor not enabled``
    * AppArmor is disabled and should be enabled
    * OR: The libvirtd enforcements are unloaded ->
```
sudo aa-complain /etc/apparmor.d/usr.sbin.libvirtd
sudo aa-enforce /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
sudo systemctl restart libvirtd
```
