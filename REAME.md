## Creating OpenStack Formatted Cloudinit ConfigDrive ISOs ##

While IaaS clouds already support mechanisms to supply cloudinit userdata to declare guest instances configurations, some virtualization environments do not. For those environments, an ISO CDROM image can be attached to BIG-IP Virtual Edition guests prior to initial booting. If the ISO image is formatted as a cloudinit ConfigDrive data source, the F5 cloudinit modules can process it.

As an example, VMWare Workstation can be use to deploy a BIG-IP Virtual Edition instance from a patched OVA archive containing F5 cloudinit modules. It will build the instance attributes per the F5-defined OVF in the OVA archive, and the instance will be powered off. Prior to starting the instance, the user can add an IDE CDROM drive device and attach a ConfigDrive ISO file. When TMOS boots for the first time, it will discover the attached IDE CDROM drive and attempt to read it as a cloudinit ConfigDrive data source.

TMOS supports cloudinit OpenStack formatted ConfigDrive. The ISO CDROM attached needs to have a volume label of `config-2` and must follow a specific layout of files, containing a specific JSON file with a specific attribute defined.

<pre>
/iso (volume label config-2)
└── openstack
    └── latest
        ├── meta_data.json
        ├───────────────────> {"uuid": "a7481585-ee3f-432b-9f7e-ddc7e5b02c27"}
        ├── user_data
        └───────────────────> THIS CAN CONTAIN ANY USERDATA YOU WANT
</pre>

If you generate an ISO9660 filesystem with Rock Ridge extensions to allow TMOS cloudinit to mount your device and act on your userdata, you can treat any virtualization environment like a properly declared IaaS. You can use any ISO9660 filesystem generation tool you want, as long as it conforms to the standard for the OpenStack ConfigDrive cloudinit data source.

This repository includes a Dockerfile and an ISO9660 generation script, called `tmos_configdrive_builder`, which will build the CDROM ISO image file with your customized inputs.

You can build the docker image from the `tmos_configdrive_builder` Dockerfile.

```
$ docker build -t tmos_configdrive_builder:latest .
```

This will use an generic Ubuntu 18.04 image, install all the necessary tools to create ISO image, and designate the python script which creates the ISO image as the execution entrypoint for the container image.

After the build process completes, a docker image will be available locally.

```
$ docker images|grep tmos_configdrive_builder
tmos_configdrive_builder     latest     23c1d99efdd5     17 seconds ago     274MB
```

The script takes inputs as environment variables instead of command line arguments.  Content, like your declarations or JSON dictionaries, are made available via files in specific directories. Docker consumes these via volume mounts.

The CDROM is built to be used in two modes:

1. `tmos_declared` mode - this mode builds f5-declarative-onboarding and, optionally, f5-appsvcs-extensions declarations, `phone_home_url`, and `phone_home_cli` variables, into a CDROM ISO usable with the `tmos_declared` cloudinit module.

2. Fully explicit mode - this mode builds a CDROM ISO from a fully defined set of `user_data`, and optionally `meta_data.json`, `vendor_data.json`, and `network_data.json` files. This allows for the construction of any settings in your `user_data` you want. This can be used to work with any of the modules defined in the tmos-cloudinit repository.

#### tmos_declared Mode Environment Variables and Files ####

The following environment variables determine how the CDROM ISO is built:

| Environment Variable | Required | Default | Description|
| --------------------- | ----- | ---------- | ---------------|
| DO_DECLARATION_FILE   | Yes | /declarations/do_declaration.json | Your f5-declarative-onboarding declaration in a text file. The declaration can be in JSON or YAML format. |
| AS3_DECLARATION_FILE  | No | /declarations/as3_declaration.json | Your f5-appsvcs-extension declaration in a text file. The declaration can be in JSON or YAML format. |
| PHOME_HOME_URL | No | None | The URL to use as the `phone_home_url` attributed of the `tmos_declared` declaration. |
| PHOME_HOME_CLI | No | None | The URL to use as the `phone_home_url` attributed of the `tmos_declared` declaration. |
| CONFIGDRIVE_FILE | Yes | /configdrives/configdrive.iso | The output ISO file. |

The files specified above must be available to the Docker instance, and need to be mounted as volumes by the Docker instance.

Example: If you have a local subfolder `declarations` directory containing both `do_declaration.json` and `as3_declaration.json`, you could create your basic CDROM ISO in the current directory using the following bash shell command.

```
docker run --rm -it -v $(pwd)/declarations:/declarations -v $(pwd):/configdrives tmos_configdrive_builder
2019-05-30 20:00:59,009 - tmos_image_patcher - INFO - building ISO9660 for tmos_declared module with declarations
2019-05-30 20:00:59,029 - tmos_image_patcher - INFO - adding f5-declarative-onboarding declaration to user_data
2019-05-30 20:00:59,029 - tmos_image_patcher - INFO - adding f5-appsvcs-extensions declaration to user_data
2019-05-30 20:00:59,042 - tmos_image_patcher - INFO - generating OpenStack mandatory ID
2019-05-30 20:00:59,046 - tmos_image_patcher - INFO - output IS09660 file: /configdrives/configdrive.iso
ls ./configdrive.iso
configdrive.iso
```

To define `phone_home_url` or `phone_home_cli` attributes in your `tmos_declared` declaration, specify them as Docker environment variables.

#### Expected Docker Volume Mounts ####

The configdrive ISO builder script expects two mount points. One is a file path to your declarations, either default or explicitly defined by Docker environment variables. The other is the directory to write your ISO file.

| Docker Volume Mount | Required | Description |
| --------------------- | ----- | ---------- |
| /declarations   | Yes | Path to the directory containing your declarations |
| /configdrives   | Yes | Path to the directory to write your ISO files |

Example: to create a `configdrive.iso` file in the current directory which would create metadata for the `tmos_declared` cloudint module, using declaration files from the `./declarations` directory, while explicitly defining a `phone_home_url`, the Docker `run` syntax would look like the following.


```
docker run --rm -it -e PHONE_HOME_URL=https://webhook.site/5f8cd8a7-b051-4648-9296-8f6afad34c93 -v $(pwd)/declarations:/declarations -v $(pwd):/configdrives tmos_configdrive_builder
```

#### Explicit Mode Environment Variables and Files ####

If the script has a `USERDATA_FILE` or finds a `/declarations/user_data` file, it will automatically prefer explicit mode. 

The other defined optional files `METADATA_FILE`, `VENDORDATA_FILE`, and `NETWORKDATA_FILE`, should conform to the OpenStack metadata standards for use with cloudinit.

| Environment Variable | Required | Default | Description|
| --------------------- | ----- | ---------- | ---------------|
| USERDATA_FILE   | Yes | /declarations/user_data | Your fully defined user_data to include. See Using F5 TMOS Cloudinit Modules below. |
| METADATA_FILE   | No | /declarations/meta_data.json | Your fully defined instance meta_data to include in JSON format. |
| VENDOR_FILE   | No | /declarations/vendor_data.json | Your fully defined instance vendor_data to include in JSON format. |
| NETWORKDATA_FILE   | No | /declarations/network_data.json | Your fully defined instance network_data to include in JSON format. |
| CONFIGDRIVE_FILE | Yes | /configdrives/configdrive.iso | The output ISO file. |

```
docker run --rm -it -e USERDATA_FILE=/declarations/instance2224 -v $(pwd)/declarations:/declarations -v $(pwd):/configdrives tmos_configdrive_builder
2019-05-30 20:04:12,158 - tmos_image_patcher - INFO - building ISO9660 configdrive user_data from /declarations/instance2224
2019-05-30 20:04:12,158 - tmos_image_patcher - INFO - generating OpenStack mandatory ID
2019-05-30 20:04:12,163 - tmos_image_patcher - INFO - output IS09660 file: /configdrives/configdrive.iso
ls ./configdrive.iso
configdrive.iso
```
