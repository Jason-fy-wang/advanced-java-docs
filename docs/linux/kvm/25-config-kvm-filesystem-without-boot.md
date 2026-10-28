---
tags:
  - config-fs
  - kvm
  - config
  - filesystem
  - fs
---
## Installation
```shell
## install the tools on the hosts
### redhat/centos
sudo dnf install -y libguestfs-tools qemu-img qemu-kvm

### ubuntu/Debian
sudo apt-get install -t libguestfs-tools qemu-utils
```

## virt-customize
`virt-customize` is the fastest, cleanest, and most scriptable tool. It inspects the OS filesystem automatically and executes commands or file copies offline without booting the VM.
```shell
## config the fs 

#!/usr/bin/env bash

set -euo pipefail
IMAGE="target-disk.qcow2"
SSH_PUB_KEY="${home}/.ssh/id_rsa.pub"
INTERNAL_NEXUS="https://nexus.internal/repository/yum-rhel8"
ADMIN_USER="admin"
## Generate SHA-512 password hash with: openssl passwd -6 "MySecretPassword123!"
ADMIN_PASSWORD_HASH=""

#1. prepare local temporary payload files
TMP_DIR=$(mktemp -d)
trap 'rm -rf $TMP_DIR' EXIT

# custome corporate Yum repo
cat <<EOF >"${TMP_DIR}/internal.repo"
[internal-base]
name=Corporate Mirror - BaseOS
baseurl=${INTERNAL_NEXUS}/BaseOS/
enabled=1
gpgcheck=0

[internal-appstream]
name=Corporate Mirror - AppStream
baseurl=${INTERNAL_NEXUS}/AppStream/
enabled=1
gpgcheck=0
EOF

# BootStrap systemUnit
cat <<EOF >"$TMP_DIR/bootstrap.service"
[Unit]
Description=KVM Bootstrapper
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/bootstrap.sh
RemainAfterExit=yes
StandardOutput=journal+console

[Install]
WantBy=multi-user.target
EOF

# Bootstrap shell script
cat <<EOF >"$TMP_DIR/bootstrap.sh"
#!/usr/bin/env bash
echo "[$(date)] Running node initialization"
EOF

chmod +x $TMP_DIR/bootstrap.sh

#2. Run virt-customize execution
echo "===> Customing $IMAGE"
virt-customize -a "$IMAGE" \
	--hostname "kvm-node1" \
	--timezone "UTC" \
	## Admin user and credentials \
	--root-password disable  \
	--run-command "useradd -m -s /bin/bash ${ADMIN_USER} || true" \
	--password "${ADMIN_USER}:encrypted:${ADMIN_PASSWORD_HASH}"  \
	--run-command "echo '${ADMIN_USER} ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/99-${ADMIN_USER} " \
	## Ssh keys injection   \
	--ssh-inject "${ADMIN_USER}:file:${SSH_PUB_KEY}"  \
	--ssh-inject "root:file:${SSH_PUB_KEY}" \
	# Yum Repository injection  \
	--copy-in "$TMP_DIR/internal.repo:/etc/yum.repos.d/" \
	# bootstrap service injection \
	--copy-in "$TMP_DIR/bootstrap.sh:/usr/local/bin"  \
	--copy-in "$TMP_DIR/bootstrap.service:/etc/systemd/system" \
	--run-command "chmod 755 /usr/local/bin/bootstrap.sh" \
	--run-command "systemctl enable bootstrap.service" \
	# Fix SELinux labels \
	--selinux-relabel

```


## guestfish
`guestfish` is the interactive or scriptable. It expose the raw libguestfs API directly and allows editing partition tables, mounting LVM volumes, creating directoies, and injecting files.

```shell
#!/usr/bin/env bash
set -euo pipefail
IMAGE="target-disk.qcow2"
SSH_PUB_KEY="$(cat ${home}/.ssh/id_rsa.pub)"

guestfish --rw -a "$IMAGE" -i <<EOF
#1.create .ssh directory and write authorized_keys
mkdir -p /root/.ssh
chmod 0700 /root/.ssh
write /root/.ssh/authorized_keys ${SSH_PUB_KEY} 
chmod 0600 /root/.ssh/authorized_keys

#2.Configure package mirror repository
mkdir -p /etc/yum.repos.d
write /etc/yum.repos.d/internal.repo "[internal-base]\nname=Corporate Mirror - BaseOS\nbaseurl=${INTERNAL_NEXUS}/BaseOS/\nenabled=1\ngpgcheck=0"

#3.Create bootstrap script
write /usr/local/bin/bootstrap.sh "#!/usr/bin/env bash\necho '[$(date)] Running node initialization'"

#4.Inject Systemd service unit and create symbolic link to enable it 
write /etc/systemd/system/bootstrap.service "[Unit]\nDescription=KVM Bootstrapper\nAfter=network-online.target\nWants=network-online.target\n[Service]\nType=oneshot\nExecStart=/usr/local/bin/bootstrap.sh\nRemainAfterExit=yes\nStandardOutput=journal+console\n[Install]\nWantBy=multi-user.target"

#5. Lock root password
## replace root line in shadow with disabled password marker
sh "passwd -l root || usermod -L root"

#6. Flag filesystem for SELinux relabel on first boot
touch /.autorelabel
EOF


```


## qemu-nbd
`qemu-nbd` exposes the QEMU disk as a standard Linux block device(/dev/nbd0).  This allows you to use standard Linux tools(mount, cp, chroot, useradd). It is ideal for local development , raw ext4 Firecracker rootfs images. or environments where libguestfs is not available.
```shell
#!/usr/bin/env bash
set -euo pipefail

if [[ -$EUID -ne 0 ]]; then
	echo "Error: qemu-nbd required root privileges"
	exit 1
fi

IMAGE="target-disk.qcow2"
NBD_DEV="dev/nbd0"
MOUNT_DIR="/mnt/vm-disk"

# 1. Load the network Block Device kernel model with partition support
modprobe nbd max_part=8
# 2. Connect the image to the nbd device
qemu-nbd --connect="NBD_DEV" "$IMAGE"
# clean up on exit or error
clean() {
	echo "Cleaning up mounts..."
	umount -R "$MOUNT_DIR" 2>/dev/null || true
	qemu-nbd --disconnect $NBD_DEV  2>/dev/null || true
	rm -rf "$MOUNT_DIR"
}

trap cleanup EXIT

# Wait for kernel to discover partitions
sleep 1

# Detect root partition (example assumes partition 1 or LVM)
# For standard REHL/Centos
ROOT_PART="${NBD_DEV}p1"
if [[ -b $"ROOT_PART" ]]; then
	ROOT_PART="$NBD_DEV"  # If raw ext4 without partition table (e.g firecracker)
fi

mkdir -p $MOUNT_DIR
mount $ROOT_PART  $MOUNT_DIR

echo " Injecting configuration directly onto filesystem"
# -- 1. Base SSH Authorized keys
mkdir -p "$MOUNT_DIR/root/.ssh"
chmod 700 "$MOUNT_DIR/root/.ssh"
cat /root/.ssh/id_rsa.pub > "$MOUNT_DIR/root/.ssh/authorized_keys"
chmod 600 "$MOUNT_DIR/root/.ssh/authorized_keys"

# -- 2. Internal Package Mirror
cat <<EOF > "$MOUNT_DIR/etc/yum.repos.d/internal.repo"
[internal-base]
name=Corporate Mirror - BaseOS
baseurl=${INTERNAL_NEXUS}/BaseOS/
enabled=1
gpgcheck=0

[internal-appstream]
name=Corporate Mirror - AppStream
baseurl=${INTERNAL_NEXUS}/AppStream/
enabled=1
gpgcheck=0
EOF

# -- 3. Bootstrap Script and Systemd Service Unit
cat <<EOF >"$MOUNT_DIR/usr/local/bin/bootstrap.sh"
#!/usr/bin/env bash
echo "[$(date)] Running node initialization"
EOF
chmod 755 "$MOUNT_DIR/usr/local/bin/bootstrap.sh"

cat <<EOF >"$MOUNT_DIR/etc/systemd/system/bootstrap.service"
[Unit]
Description=KVM Bootstrapper
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/bootstrap.sh
RemainAfterExit=yes
StandardOutput=journal+console

[Install]
WantBy=multi-user.target
EOF

# Enable the service by creating the systemd symlink
mkdir -p $MOUNT_DIR/etc/systemd/system/multi-user.target.wants
ln -sf "/etc/systemd/system/bootstrap.service"  \
		"$MOUNT_DIR/etc/systemd/system/multi-user.target.wants/bootstrap.service"

# -- 4. User credentials & chroot Execution
# Bind mount system pseudo-filesystem to chroot and modify users cleanly
mount --bind /dev "$MOUNT_DIR/dev"
mount --bind /proc "$MOUNT_DIR/proc"
mount --bind /sys "$MOUNT_DIR/sys"

chroot $MOUNT_DIR  /bin/bash -c "
	# Lock root passwd and enforce key access
	passwd -d -l root
	# Set SELinux autorelabel flag so file contexts are corrected on boot
	touch /.autorelabel
"

```


## system-service

The last soulation is config a one time service, which will config the server after server bring up.




