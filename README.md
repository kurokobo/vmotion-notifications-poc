<!-- omit in toc -->
# [WIP] Demo: vSphere vMotion Notifications

Example implementation for vSphere vMotion Notifications.

<!-- omit in toc -->
## Table of contents

- [Requirements](#requirements)
- [Enable vSphere vMotion Notifications using helper script](#enable-vsphere-vmotion-notifications-using-helper-script)
  - [Initial setup for helper script](#initial-setup-for-helper-script)
  - [Per host settings](#per-host-settings)
  - [Per VM settings](#per-vm-settings)
- [Handle vSphere vMotion Notifications in guest OS](#handle-vsphere-vmotion-notifications-in-guest-os)
- [References](#references)

## Requirements

Refer documentation ([this](https://core.vmware.com/resource/vsphere-vmotion-notifications) and [this](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-vcenter-esxi-management/GUID-0540DF43-9963-4AF9-A4DB-254414DC00DA.html#how-to-configure-a-virtual-machine-for-vsphere-vmotion-notifications-3)) for details.

- vSphere **8.0** or later
- Virtual Hardware Version **20** or later
- VMware Tools (or Open VM Tools ) **11.0** or later

## Enable vSphere vMotion Notifications using helper script

To enable vSphere vMotion Notifications, **both VM and Host have to be configured**. This will be done by following three steps.

1. **Per Host**: Configure default timeout value for all hosts where VMs may run on by vMotion (`VmOpNotificationToApp.Timeout`).
1. **Per VM**: Enable vSphere vMotion Notifications per VM (`vmOpNotificationToAppEnabled`).
1. **Per VM**: Configure timeout value per VM (`vmOpNotificationTimeout`).

This repository includes helper script to modify these configurations.

### Initial setup for helper script

```bash
# Prepare Python environment
pip install -r helper/requirements.txt

# Prepare environment variables
export VMWARE_HOST="vcsa.example.com"
export VMWARE_USER="vmotion@vsphere.local"
export VMWARE_VALIDATE_CERTS="false"  # If required
read -sp "Password: " VMWARE_PASSWORD; export VMWARE_PASSWORD
```

### Per host settings

Modifying timeout value for **all hosts** where VMs may be running on by vMotion appears to be required, since the default timeout value is displayed as `1800` but **the actual default value is `0`**, and the smaller timeout value of the host and VM will be used to make vMotion delayed.

```bash
# Show current configuration for specific host
python helper/vmotion_notification.py host <HOST_NAME>

# Modify timeout for vMotion Notifications for specific host (VmOpNotificationToApp.Timeout = <VALUE>), e.g. 600
python helper/vmotion_notification.py host <HOST_NAME> --timeout <VALUE>
```

### Per VM settings

vSphere vMotion Notifications has to be enabled per VM. If timeout value is configured for VM, the smaller timeout value of the host and VM will be used to make vMotion delayed.

```bash
# Show current configuration (can NOT gather vmOpNotificationTimeout)
python helper/vmotion_notification.py vm <VM_NAME>

# Enable vMotion Notification (vmOpNotificationToAppEnabled = true)
python helper/vmotion_notification.py vm <VM_NAME> --enable

# Disable vMotion Notification (vmOpNotificationToAppEnabled = false)
python helper/vmotion_notification.py vm <VM_NAME> --disable

# Modify timeout for vMotion Notification (vmOpNotificationTimeout = <VALUE>), e.g. 120
python helper/vmotion_notification.py vm <VM_NAME> --timeout <VALUE>
```

## Handle vSphere vMotion Notifications in guest OS

Applications can use `vmtoolsd` (command line utility that installed with VMware Tools or Open VM Tools) to handle notifications.

```bash
# Register your applications for notifications
# Note that uniqueToken generated by this command is important to handle notifications and is only displayed at this time
$ vmtoolsd --cmd 'vm-operation-notification.register {"appName": "demo", "notificationTypes": ["sla-miss"]}'
{"version":"1.0.0", "result": true, "guaranteed": true, "uniqueToken": "525b5364-6caf-24a0-562c-87955647baa4", "notificationTimeoutInSec": 60 }

# List registered application
$ vmtoolsd --cmd 'vm-operation-notification.list'
{"version":"1.0.0", "result": true, "info": [     {"appName": "demo", "notificationTypes": ["sla-miss"]}] }

# Unregister your applications for notifications
$ vmtoolsd --cmd 'vm-operation-notification.unregister {"uniqueToken": "525b5364-6caf-24a0-562c-87955647baa4"}'
{"version":"1.0.0", "result": true }
```

Applications can check if your application is notified by following command. This command should be invoked periodically.

```bash
# Check if your application is notified
$ vmtoolsd --cmd 'vm-operation-notification.check-for-event {"uniqueToken": "525b5364-6caf-24a0-562c-87955647baa4"}'
```

Applications can get following events by above command. Note that only one event appears to be able to be retrieved in a single command execution, even if multiple events are queued.

```bash
# If your application IS NOT notified
{"version":"1.0.0", "result": true }

# If your application IS notified; vMotion started
# This notification means that your application needs to be ready for vMotion
# Note that operationId included in the output will be used to acknowledge this notification in later step
{"version":"1.0.0", "result": true, "eventType": "start",  "opType": "host-migration", "eventGenTimeInSec": 1666730185, "notificationTimeoutInSec": 120, "destNotificationTimeoutInSec": 120, "notificationTypes": ["sla-miss"],  "operationId": 6279072692702276663 }

# If your application IS notified; vMotion finished
{"version":"1.0.0",  "result": true, "eventType": "end",  "opType": "host-migration", "opStatus": "success", "eventGenTimeInSec": 1666730200, "notificationTypes": ["sla-miss"],  "operationId": 6279072692702276663 }

# Notificaion that just to inform the value for timeout has been updated
{"version":"1.0.0",  "result": true, "eventType": "timeout-change",  "eventGenTimeInSec": 1666736715, "notificationTimeoutInSec": 120, "newNotificationTimeoutInSec": 60, "notificationTypes": ["sla-miss"],  "operationId": 1666736715415723 }
```

To acknowledge the notification, invoke following command.

```bash
# Acknowledge the notification
$ vmtoolsd --cmd 'vm-operation-notification.ack-event {"operationId": 6279072692702276663, "uniqueToken": "525b5364-6caf-24a0-562c-87955647baa4"}'
{"version":"1.0.0", "result": true, "operationId": 6279072692702276663,  "ackStatus": "ack_received" }
```

## References

- [vSphere vMotion Notifications](https://core.vmware.com/resource/vsphere-vmotion-notifications)
- [Virtual Machine Conditions and Limitations for vSphere vMotion](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-vcenter-esxi-management/GUID-0540DF43-9963-4AF9-A4DB-254414DC00DA.html#how-to-configure-a-virtual-machine-for-vsphere-vmotion-notifications-3)
- [vSphere Web Services API - VMware API Explorer - VMware {code}](https://developer.vmware.com/apis/1355/vsphere)
  - [Data Object - VirtualMachineConfigInfo(vim.vm.ConfigInfo)](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/c476b64b-c93c-4b21-9d76-be14da0148f9/04ca12ad-59b9-4e1c-8232-fd3d4276e52c/SDK/vsphere-ws/docs/ReferenceGuide/vim.vm.ConfigInfo.html)
  - [Data Object - VirtualMachineConfigSpec(vim.vm.ConfigSpec)](https://vdc-repo.vmware.com/vmwb-repository/dcr-public/c476b64b-c93c-4b21-9d76-be14da0148f9/04ca12ad-59b9-4e1c-8232-fd3d4276e52c/SDK/vsphere-ws/docs/ReferenceGuide/vim.vm.ConfigSpec.html)
