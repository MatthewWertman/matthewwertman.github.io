---
title: Using the $PIR Table to get PCIe device order
date: 2024-04-20 00:04:00 +/-0800
categories: [writing, linux]
tags: [pcie, hiveos, mining, crypto, tutorial]
author: mwertman
---

Identifying PCIe devices physically in a system can be a challenging problem. In my experience, when you have 6, 8, 12 PCIe devices in a system (GPUs in my case), they often don't show up how you would expect. Now lets say one GPU starts having issues, how do you know which one is which?

With the hardware that I have worked with, I have had a lot of success with using the $PIR table to identify which bus addresses are attached to which physical PCIe slot. In this blog, I will show you how you can do so with the help of a script!

## Introduction To The Problem
You would think that if you had a series of 12 PCIe slots that they would show up in order (from left to right, starting closest to the CPU), but this is often not the case in my experience.
In HiveOS specifically, they show the detected GPU devices in the system along with their assigned bus address after login:

![HiveOS's motd commmand](../../assets/img/2024-04-20-motd.PNG)

But not knowing the order of how the GPU devices were assigned makes it not feasible to hunt down a specific card by only its bus address. Leaving only the process of elimination to determine the problem card.

## The $PIR Table
The PCI Interrupt Routing Table ($PIR) is a table in read-only memory containing the routing information of the system's PCI slot entries (embedded devices or physical PCIe slots). Each entry defines PCI device information that is attached to that slot, even an 'slot number' indicating the corresponding physical slot. So if we can figure out each PCIe slot's corresponding slot number, we have all the information that we need to generate a "map" of the current rig!

## Getting Started
Let's build this tool together and as we get further it should get a bit clearer of what is going on. Lets say we have 12 GPUs in a unit that is running HiveOS. As shown before, this is what we get when we log into the system:
![HiveOS's motd commmand](../../assets/img/2024-04-20-motd.PNG)

We can actually get this $PIR table data pretty easily within HiveOS with the following command:
```bash
biosdecode
```
A lot of other BIOS information gets outputted as well but we are specifically after the `PCI Interrupt Routing 1.0 present` header. This is essentially the $PIR table printed in a human-readable format

![biosdecode PIR output](../../assets/img/2024-04-20-biosdecodePIR.PNG)

This section is mainly made up of slot entries which each represents PCI device. Each `Slot Entry` contains the device bus ID attached and slot number if available which acts like a designation for the physical slot. If we were to remove an GPU from the system and reboot, we would see that device ID and its corresponding table entry missing from the table. We can use these assigned slot numbers as identifiers to each physical slot on the motherboard and effectively made our map! Lets see what this looks like.

Adding in GPUs one by one starting at the PCIe slot closest to the CPU will get us the `slot number` for each slot and identify which device is in which slot. Using this method, this is the order for our example:

| Physical Slot (starting at CPU) | BIOS Slot Number |
|---------------|------------------|
| Slot 1        | 16               |
| Slot 2        | 37               |
| Slot 3        | 38               |
| Slot 4        | 39               |
| Slot 5        | 40               |
| Slot 6        | 41               |
| Slot 7        | 42               |
| Slot 8        | 43               |
| Slot 9        | 48               |
| Slot 10       | 53               |
| Slot 11       | 54               |
| Slot 12       | 55               |

Already we can put together our map of the GPU devices vs. their physical motherboard slots by replacing those slot numbers with the corresponding GPU bus IDs!

| Physical Slot (starting at CPU) | GPU Bus ID |
|---------------|------------------|
| Slot 1        | 01:00.0          |
| Slot 2        | 07:00.0          |
| Slot 3        | 08:00.0          |
| Slot 4        | 09:00.0          |
| Slot 5        | 0a:00.0          |
| Slot 6        | 0b:00.0          |
| Slot 7        | 0c:00.0          |
| Slot 8        | 0d:00.0          |
| Slot 9        | 02:00.0          |
| Slot 10       | 03:00.0          |
| Slot 11       | 04:00.0          |
| Slot 12       | 05:00.0          |

## Let's write a script!
Now that we have all of the needed information and know how to generate a GPU vs. PCIe slot map, we will automate it with a shell script:

```bash
#!/bin/bash
##
## gpu_lookup_table -- generates a PCI bus address vs. PCIe slot map

# $PIR found flag
PIR=0

# slot designations in order of left to right (starting at CPU)
pir_map=(16 37 38 39 40 41 42 43 48 53 54 55)

# store the output of biosdecode
C_BIOS_PIR_TABLE="$(sudo biosdecode)"

# not every system may have a $PIR table so we have a function to check
function check_for_pir_table {
    if echo "$C_BIOS_PIR_TABLE" | grep -q "PCI Interrupt Routing"; then
        return 0
    fi
    return 1
}

# hiveos stores the detected GPU devices in json
# parse $GPU_DETECT_JSON to get detected GPU bus addresses
function get_gpu_info {
    [[ ! -f $GPU_DETECT_JSON ]] && return 1
    local gpu_detect_json="$(< $GPU_DETECT_JSON)"

    BUSID=()

    local idx=-1
    while IFS=";" read busid brand vendor mem vbios name; do
        ((idx++))
        BUSID[idx]="$busid"

    done < <( echo "$gpu_detect_json" | jq -r -c '.[] | (.busid+";"+.brand+";"+.vendor+";"+.mem+";"+.vbios+";"+.name)' 2>/dev/null )

    [[ ${#BUSID[@]} -eq 0 ]] && return 1
    return 0
}

# get_pir_device_slot
# $1 - GPU bus address
# returns PCIe designation for given bus adddress in $PIR
function get_pir_device_slot () {
    local C_GET_PIR_PCI_SLOT_NUMBER
    local pci_slot
    pci_slot=""
    
    C_GET_PIR_PCI_SLOT_NUMBER=$(echo "$C_BIOS_PIR_TABLE" | grep "$(echo "$1" | cut -d"." -f1)" | grep -Poe 'slot number ([0-9]{1,})' | awk '{print $3}')
    
    if [[ -z $pci_slot ]]; then
        for (( idx=0; idx < ${#pir_map[@]}; idx++ )); do
            if [[ "${pir_map[$idx]}" == "$C_GET_PIR_PCI_SLOT_NUMBER" ]]; then
               pci_slot="PCIE $((idx+1))"
               break
            fi
        done
    fi
    echo "$pci_slot"
    unset C_GET_PIR_PCI_SLOT_NUMBER
}

if check_for_pir_table; then
    PIR=1  # $PIR was found
fi
get_gpu_info

if [[ PIR -eq 1 ]]; then
	for(( idx=1; idx < ${#BUSID[@]}; idx++ )); do  # idx starts at 1 to skip CPU busid
        dev=${BUSID[idx]}
        printf "${BUSID[idx]} $(get_pir_device_slot $dev)\n"
    done
else
    echo "\$PIR Table not found. Exitting..."
fi
exit 0
```
{: file="gpu_lookup_table"}

Running this script will get us the table we got previously:

![gpu_lookup_table script output](../../assets/img/2024-04-20-gltoutputex1.jpg)

## Adding support for AMD GPUs
Generally AMD GPUs detect in the system a little bit differently than Nvidia. Here is an example:

![AMD GPU in lspci](../../assets/img/2024-04-20-amdlspci.png)

When looking at one within `lspci`, there are multiple bus addresses to one card. The one we are after is the 'VGA compatible controller' (03:00.0 in this case).
But when running the `biosdecode` command we see that the device IDs are actually the 'Upstream Port of PCI Express Switch' (01.00.0).
![AMD GPU in biosdecode](../../assets/img/2024-04-20-amdbiosdecode.jpg)

To account for this we can add another function for AMD cards to get the proper bus address to search the $PIR with:
```bash
# store lspci output
C_LSPCI_TABLE="$(lspci -mm)"

function get_amd_pir_busid () {
    local C_GET_AMD_BUSID
    local dev=
    if ! echo "$C_LSPCI_TABLE" | grep -q "Ellesmere"; then  # Ellesmere behave like nvidia cards
        C_GET_AMD_BUSID=$(echo "$C_LSPCI_TABLE" | grep "$1" -B 2 | grep "$1" -B 2 | grep -Po '([0-9a-f]{2})(:00)\.0' -m 1)
        dev=${C_GET_AMD_BUSID}
    else
        dev=$1
    fi
    unset C_GET_AMD_BUSID
    echo "$dev"
}
```

This is the full script with AMD GPU support:

```bash
#!/bin/bash
##
## gpu_lookup_table -- generates a PCI bus address vs. PCIe slot map; now with AMD support!

# $PIR found flag
PIR=0

# slot designations in order of left to right (starting at CPU)
pir_map=(16 37 38 39 40 41 42 43 48 53 54 55)

# store the output of biosdecode
C_BIOS_PIR_TABLE="$(sudo biosdecode)"
# store lspci output
C_LSPCI_TABLE="$(lspci -mm)"

# not every system may have a $PIR table so we have a function to check
function check_for_pir_table {
    if echo "$C_BIOS_PIR_TABLE" | grep -q "PCI Interrupt Routing"; then
        return 0
    fi
    return 1
}

# hiveos stores the detected GPU devices in json
# parse $GPU_DETECT_JSON to get detected GPU bus addresses
function get_gpu_info {
    [[ ! -f $GPU_DETECT_JSON ]] && return 1
    local gpu_detect_json="$(< $GPU_DETECT_JSON)"

    BUSID=()

    local idx=-1
    while IFS=";" read busid brand vendor mem vbios name; do
        ((idx++))
        BUSID[idx]="$busid"
        BRAND[idx]="$brand"

    done < <( echo "$gpu_detect_json" | jq -r -c '.[] | (.busid+";"+.brand+";"+.vendor+";"+.mem+";"+.vbios+";"+.name)' 2>/dev/null )

    [[ ${#BUSID[@]} -eq 0 ]] && return 1
    return 0
}

# get_pir_device_slot
# $1 - GPU bus address
# returns PCIe designation for given bus adddress in $PIR
function get_pir_device_slot () {
    local C_GET_PIR_PCI_SLOT_NUMBER
    local pci_slot
    pci_slot=""
    
    C_GET_PIR_PCI_SLOT_NUMBER=$(echo "$C_BIOS_PIR_TABLE" | grep "$(echo "$1" | cut -d"." -f1)" | grep -Poe 'slot number ([0-9]{1,})' | awk '{print $3}')
    
    if [[ -z $pci_slot ]]; then
        for (( idx=0; idx < ${#pir_map[@]}; idx++ )); do
            if [[ "${pir_map[$idx]}" == "$C_GET_PIR_PCI_SLOT_NUMBER" ]]; then
               pci_slot="PCIE $((idx+1))"
               break
            fi
        done
    fi
    echo "$pci_slot"
    unset C_GET_PIR_PCI_SLOT_NUMBER
}

function get_amd_pir_busid () {
    local C_GET_AMD_BUSID
    local dev=
    if ! echo "$C_LSPCI_TABLE" | grep -q "Ellesmere"; then  # Ellesmere behave like nvidia cards
        C_GET_AMD_BUSID=$(echo "$C_LSPCI_TABLE" | grep "$1" -B 2 | grep "$1" -B 2 | grep -Po '([0-9a-f]{2})(:00)\.0' -m 1)
        dev=${C_GET_AMD_BUSID}
    else
        dev=$1
    fi
    unset C_GET_AMD_BUSID
    echo "$dev"
}

if check_for_pir_table; then
    PIR=1  # $PIR was found
fi
get_gpu_info

if [[ PIR -eq 1 ]]; then
	for(( idx=1; idx < ${#BUSID[@]}; idx++ )); do  # idx starts at 1 to skip CPU busid
        dev=${BUSID[idx]}
        if [[ "${BRAND[idx]}" == "amd" ]]; then
            dev=$(get_amd_pir_busid ${BUSID[idx]})
        fi
        printf "${BUSID[idx]} $(get_pir_device_slot $dev)\n"
    done
else
    echo "\$PIR Table not found. Exitting..."
fi
exit 0
```
{: file="gpu_lookup_table"}

## Extending the script
Even though this script works for this system, its actually most likely not work with another. What we have done previously only works on systems with that same mapping/motherboard. If we wanted to add support for another motherboard, we would have to map it by adding GPU devices one by one and recording the attached slot numbers. Then its just a matter of adding that map to the above script. This is easier said than done, given there can be quirks when reading the $PIR table like this.

It should be noted that the $PIR table contains data provided by the OEM/BIOS manufacturer. This makes it very dependent on the OEM to give accurate information. Information provided by the OEM may be incomplete, inaccurate, or wrong. This is why the data provided by the script can’t be 100% trusted. The goal of this script is to at least give a starting point when debugging rigs.

## Conclusion
I mainly focused on HiveOS, but it should be possible to adapt to other Linux-based OSes (or even Windows) and other hardware. Obviously the main drawback with this approach is all the work necessary to ensure compatibility across systems, but what is front-facing to the user is a easy to understand table of how the GPU devices are mapped in the system for servicing!

